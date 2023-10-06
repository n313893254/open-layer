<script setup lang="ts">
import { onMounted } from 'vue';
import Map from 'ol/Map';
import View from 'ol/View';
import { Vector, Group } from "ol/layer";
import { Vector as SourceVector } from "ol/source";
import { GeoJSON } from "ol/format";
import { Text, Fill, Stroke, Style } from 'ol/style';
import { transformExtent, fromLonLat } from "ol/proj";
import { intersects } from "ol/extent";

import { MAP_DEFAULT_OPTIONS, EPSG4326, PROVINCES } from "@/configs/constans";

function createLayerStyle(feature: any) {
  const text = new Text({
    text: feature.get('name_zh') || feature.get('name'),
    fill: new Fill({ color: 'black' }),
    stroke: new Stroke({ color: 'black', width: 1 }),
  })

  const stroke = new Stroke({
    color: feature.get('name') === 'China' ? 'transparent' : '#ddd',
    width: 1,
  })

  const style = new Style({
    text,
    stroke,
  })

  return [style]
}

function createLayer(url: string, options?: Record<string, string|number>) {
  console.log(url, 'url')
  const layer = new Vector({
    source: new SourceVector({
      url,
      format: new GeoJSON(),
    }),
    ...options,
  });

  layer.setStyle(createLayerStyle)

  return layer
}

enum LayerIndex {
  Zero = 0,
  First = 5,
  Second = 6,
}

const LayerCacheMap: Record<LayerIndex, Record<string, LayerMapItem>> = {
  [LayerIndex.Zero]:   {},
  [LayerIndex.First]:  {}, 
  [LayerIndex.Second]: {},
}

function addLayerCache(type: LayerIndex, name: string, layer: Vector<any>) {
  if (type > LayerIndex.First) {
    layer.setVisible(false)
  }

  if (!LayerCacheMap[type]) {
    LayerCacheMap[type] = {} 
  }

  if (!LayerCacheMap[type][name]) {
    LayerCacheMap[type][name] = {
      name,
      layer,
    }
  }
}

let map: Map;

onMounted(() => {
  const asiaLayer = createLayer('/geojson/asia.json')
  const chinaLayer = createLayer('/geojson/china.json')
  const japanLayer = createLayer('/geojson/japan.json')

  addLayerCache(LayerIndex.Zero, "asia", asiaLayer);
  addLayerCache(LayerIndex.First, "china", chinaLayer);
  addLayerCache(LayerIndex.Second, "japan", japanLayer);

  const layerGroup = new Group({
    layers: [asiaLayer, chinaLayer, japanLayer],
  });

  map = new Map({
    target: 'map',
    layers: [],
  });

  const { center, zoom, minZoom, maxZoom, extent } = MAP_DEFAULT_OPTIONS;

  const view = new View({
    center: fromLonLat(center),
    zoom,
    minZoom,
    maxZoom,
    constrainResolution: true,
    extent: transformExtent(extent, EPSG4326, map.getView().getProjection()),
  })

  map.setView(view)

  map.addLayer(layerGroup);

  for(const province in PROVINCES) {
    console.log(province, 'province')
    const layer = createLayer(`/geojson/china/${province}.json`, {
      renderBuffer: 100,
    })

    addLayerCache(LayerIndex.Second, province, layer);

    map.addLayer(layer)
  }

  map.getView().on('change', (event) => {
    const view = event.target;
    const zoom = view.getZoom();
    const currentExtent = view.calculateExtent(map.getSize());

    const transformedExtent = transformExtent(currentExtent, map.getView().getProjection(), EPSG4326);

    for (const _index in LayerCacheMap) {
      const index = _index as any as LayerIndex
      if (index <= LayerIndex.First) {
        continue
      }

      const chinaLayer = LayerCacheMap[LayerIndex.First].china.layer

      if (zoom > index) {
        for (const key in PROVINCES) {
          const extent = PROVINCES[key]
          const isCityInView = intersects(extent, transformedExtent);
          const layerCache = LayerCacheMap[index][key]

          if (!layerCache || !layerCache.layer) {
            continue
          }

          const layer = layerCache.layer

          if (isCityInView) {
            if (!layer.getVisible()) {
              layer.setVisible(true)

              chinaLayer.setVisible(false) 
            }
          } else {
            layer.setVisible(false)
          }
        }
      } else {
        for (const key in PROVINCES) {
          const layerCache = LayerCacheMap[index][key]

          if (layerCache && layerCache.layer) {
            layerCache.layer.setVisible(false)
            chinaLayer.setVisible(true)
          }
        }
      }
    }
  })
});
</script>

<template>
  <div class="page">
    <div id="map" />
  </div>
</template>

<style>
#map {
  width: 100%;
  height: 100vh;
}
</style>
