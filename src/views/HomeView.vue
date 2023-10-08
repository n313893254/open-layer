<script setup lang="ts">
import { onMounted } from 'vue';
import { Map, View, Feature, Overlay } from 'ol';
import { Vector, Group } from "ol/layer";
import { Vector as SourceVector } from "ol/source";
import { GeoJSON } from "ol/format";
import { Text, Fill, Stroke, Style, Icon } from 'ol/style';
import { transformExtent, fromLonLat, transform } from "ol/proj";
import { intersects } from "ol/extent";
import { Point, LineString, } from 'ol/geom';
import { Select } from 'ol/interaction';
import { pointerMove } from 'ol/events/condition'
import { toRadians } from 'ol/math'

import { MAP_DEFAULT_OPTIONS, EPSG4326, PROVINCES, EPSG3857 } from "@/configs/constans";

const YU_LIN_COORD = [110.1810, 22.6545]
const NAN_NING_COORD = [108.3660, 22.8172]
const Duration = 2500;

function countDegrees(startPoint: number[], endPoint: number[]) {
  const dx = endPoint[0] - startPoint[0];
  const dy = endPoint[1] - startPoint[1];

  const radians = Math.atan2(dy, dx);
  let degrees = radians * (180 / Math.PI);

  if (degrees < 0) {
    degrees += 360;
  }

  return degrees;
}

function interpolatePoints(
  start: number[],
  end: number[],
  numPoints: number
) {
  const interpolatedPoints = [];
  const stepX = (end[0] - start[0]) / (numPoints + 1);
  const stepY = (end[1] - start[1]) / (numPoints + 1);

  for (let i = 0; i < numPoints; i++) {
    const x = start[0] + stepX * (i + 1);
    const y = start[1] + stepY * (i + 1);
    interpolatedPoints.push([x, y]);
  }

  return interpolatedPoints.map((c) => fromLonLat(c));
}


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

function createPoint(src: string, coords?: number[], opt: Record<string, any> = {}) {
  const layer = new Vector({
    source: new SourceVector(),
  });

  const feature = new Feature({
    geometry: new Point(coords ? fromLonLat(coords) : []),
  })

  const style = new Style({
    image: new Icon({
      src,
      ...opt,
    })
  })

  feature.setStyle(style)

  layer.getSource()?.addFeature(feature)

  map.addLayer(layer)

  return {
    layer,
    feature,
  }
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

  const { layer: previewLayer } = createPoint('/images/icons/marker.svg', YU_LIN_COORD, {
    color: 'red',
    scale: 1,
    anchor: [0.15, 0.9],
  })

  createPoint('/images/icons/location.svg', NAN_NING_COORD, {
    scale: 1,
  })

  const carExtent = transform(YU_LIN_COORD, EPSG3857, EPSG4326);
  const degrees = countDegrees(NAN_NING_COORD, carExtent);

  const { 
    feature: carFeature,
    layer: carLayer, 
  } = createPoint('/images/icons/car.svg', undefined, {
    scale: 0.5,
    rotation: toRadians(45 + 360 - degrees),
  })

  const lineLayer = new Vector({
    source: new SourceVector()
  })

  map.addLayer(lineLayer)

  const coords = interpolatePoints(NAN_NING_COORD, YU_LIN_COORD, 100);
  const lineFeature = new Feature({
    geometry: new LineString(coords)
  })

  const random = () => Math.floor(Math.random() * 100) + 150
  const color = `rgb(${random()}, ${random()}, ${random()})`

  const lineStyle = new Style({
    stroke: new Stroke({
      color,
      width: 4,
      lineDash: [5, 10],
    })
  })

  lineFeature.setStyle(lineStyle)

  lineLayer.getSource()?.addFeature(lineFeature)

  function playAnimate(
    feature: Feature,
    coordsList: number[][],
    callback: Function,
  ) {
    let startTime = new Date().getTime()

    function animate() {
      const currentTime = new Date().getTime()
      const elapsedTime = currentTime - startTime
      const fraction = elapsedTime / Duration
      const index = Math.round(coordsList.length * fraction)

      if (index < coordsList.length) {
        const geometry = feature.getGeometry()

        if (geometry instanceof Point) {
          console.log(coordsList[index], 'Point')
          geometry?.setCoordinates(coordsList[index])
        }

        requestAnimationFrame(animate)
      } else {
        callback()
      }
    }

    animate()
  }

  const previewEle = document.getElementById('map-marker-preview') as HTMLElement

  if (previewEle) {
    const preview = new Overlay({
      element: previewEle,
      positioning: 'center-center',
      offset: [20, -75],
      stopEvent: false,
    })

    map.addOverlay(preview)

    const interaction = new Select({
      layers: [previewLayer],
      condition: pointerMove,
      style: null,
    })

    map.addInteraction(interaction)

    let markerInfo: {
      info: Record<string, any>,
      coords: number[],
    } = {
      info: {},
      coords: [],
    }

    let animateDone = false

    interaction.on('select', (event) => {
      if (event.selected.length > 0) {
        const selectedFeature = event.selected[0]

        const geometry = selectedFeature.getGeometry()
        if (geometry instanceof Point) {
          markerInfo.coords = geometry.getCoordinates()
        }

        map.getTargetElement().style.cursor = 'pointer'
        preview.setPosition(markerInfo.coords)

        const line = lineFeature?.getGeometry()
        const coordsList = line?.getCoordinates() || []

        if (animateDone) {
          carLayer.getSource()?.addFeature(carFeature)
          animateDone = false
        }

        playAnimate(carFeature, coordsList, () => {
          carLayer.getSource()?.removeFeature(carFeature)
          animateDone = true
        })
      } else {
        preview.setPosition(undefined)
        map.getTargetElement().style.cursor = 'default'
      }
    })
  }
});
</script>

<template>
  <div class="page">
    <div id="map" >
      <div id="map-marker-preview">
        <div class="img-box">
          <img src="/images/preview/mid-autumn.jpeg" alt="">
        </div>
        <div class="info">
          <div class="title">
            广西壮族自治区玉林市玉州区
          </div>
          <div class="desc">
            和女朋友一起过中秋
          </div>
          <div class="date">
            2023-09-30
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="less">
#map {
  width: 100%;
  height: 100vh;
}

#map-marker-preview {
  display: flex;
  box-sizing: border-box;
  position: absolute;
  pointer-events: none;
  z-index: 100;
  width: 300px;
  height: 120px;
  background-color: #eaeaea;
  border-radius: 10px;
  padding: 15px;
  cursor: pointer;

  .img-box {
    max-width: 100px;
    height: auto;
    border-radius: 5px;
    margin-right: 20px;
    padding-bottom: 0;

    img {
      height: 100%;
    }
  }

  .info {
    display: flex;
    flex-direction: column;
    justify-content: space-between;

    span {
      display: inline-block;
    }

    .title {
      width: 100%;
      font-size: 14px;
      opacity: 0.8;
    }

    .desc {
      font-size: 12px;
      opacity: 0.7;
    }

    .date {
      text-align: right;
      font-size: 12px;
      opacity: 0.5;
    }
  }
}
</style>
