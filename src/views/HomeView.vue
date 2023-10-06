<script setup lang="ts">
import { onMounted } from 'vue';
import Map from 'ol/Map';
import View from 'ol/View';
import { Vector, Group } from "ol/layer";
import { Vector as SourceVector } from "ol/source";
import { GeoJSON } from "ol/format";
import { Text, Fill, Stroke, Style } from 'ol/style';
import { transformExtent, fromLonLat } from "ol/proj";

import { MAP_DEFAULT_OPTIONS, EPSG4326 } from "@/configs/constans";

let map;

onMounted(() => {
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

  function createLayer(url: string) {
    const layer = new Vector({
      source: new SourceVector({
        url,
        format: new GeoJSON(),
      }),
    });

    layer.setStyle(createLayerStyle)

    return layer
  }

  const asiaLayer = createLayer('/geojson/asia.json')
  const chinaLayer = createLayer('/geojson/china.json')
  const japanLayer = createLayer('/geojson/japan.json')

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
