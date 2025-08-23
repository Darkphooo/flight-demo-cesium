<template>
  <div ref="cesiumRef" class="cesium-container"></div>
  <div class="button-list">
    <div v-if="!isFetching" class="button" @click="refreshFlightData">
      开始更新当前区域航班
    </div>
    <div v-else class="button" @click="stopRefreshFlightData">
      停止更新当前区域航班
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, useTemplateRef, ref } from "vue";
import { TdtImageryProvider } from "@cesium-china/cesium-map";
import * as cesium from "cesium";
import axios from "axios";
import A320Model from "@/assets/a320.glb";

interface FlightData {
  alt: number;
  callsign: string;
  fr24_id: string;
  gspeed: number;
  hex: string;
  lat: number;
  lon: number;
  source: string;
  squawk: string;
  timestamp: Date;
  track: number;
  vspeed: number;
}

const cesiumRef = useTemplateRef("cesiumRef");
const viewer = ref();
const isFetching = ref(false);
const timer = ref();

function getFlightData() {
  const extend = viewer.value.camera.computeViewRectangle();
  axios
    .get(`https://fr24api.flightradar24.com/api/live/flight-positions/light`, {
      params: {
        bounds: `${cesium.Math.toDegrees(extend.north)},${cesium.Math.toDegrees(
          extend.south
        )},${cesium.Math.toDegrees(extend.west)},${cesium.Math.toDegrees(
          extend.east
        )}`,
      },
      headers: {
        Accept: "application/json",
        "Accept-Version": "v1",
        Authorization: `Bearer ${import.meta.env.VITE_FLIGHTRADAR_KEY}`,
      },
    })
    .then((res: { data: { data: FlightData[] } }) => {
      displayFlights(res.data.data);
    });
}

function displayFlights(flights: FlightData[]) {
  viewer.value.entities.removeAll();
  flights.forEach((flight) => {
    viewer.value.entities.add({
      id: flight.fr24_id,
      position: cesium.Cartesian3.fromDegrees(
        flight.lon,
        flight.lat,
        flight.alt
      ),
      orientation: cesium.Transforms.headingPitchRollQuaternion(
        cesium.Cartesian3.fromDegrees(flight.lon, flight.lat, 0),
        new cesium.HeadingPitchRoll(
          cesium.Math.toRadians(flight.track + 90),
          0,
          0
        )
      ),
      label: flight.callsign,
      model: {
        uri: A320Model,
        minimumPixelSize: 64,
        maximumScale: 128,
        show: true,
        color: cesium.Color.fromAlpha(cesium.Color.RED, 1),
        colorBlendMode: cesium.ColorBlendMode.HIGHLIGHT,
      },
    });
  });
}

function refreshFlightData() {
  isFetching.value = true;
  timer.value && clearInterval(timer.value);
  getFlightData();
  timer.value = setInterval(getFlightData, 10000);
}

function stopRefreshFlightData() {
  timer.value && clearInterval(timer.value);
  isFetching.value = false;
}

onMounted(() => {
  if (cesiumRef.value) {
    viewer.value = new cesium.Viewer(cesiumRef.value, {
      animation: false,
      shouldAnimate: true,
      selectionIndicator: false,
      baseLayerPicker: false,
      fullscreenButton: false,
      geocoder: false,
      homeButton: false,
      infoBox: false,
      sceneModePicker: false,
      timeline: false,
      navigationHelpButton: false,
      navigationInstructionsInitiallyVisible: false,
      showRenderLoopErrors: false,
      shadows: false,
    });

    viewer.value.scene.fxaa = true;
    viewer.value.scene.postProcessStages.fxaa.enabled = false;
    viewer.value.scene.globe.showGroundAtmosphere = true;
    viewer.value.scene.screenSpaceCameraController.constrainedPitch =
      cesium.Math.toRadians(-20);
    viewer.value.scene.screenSpaceCameraController.autoResetHeadingPitch =
      false;
    viewer.value.scene.screenSpaceCameraController.inertiaZoom = 0.5;
    viewer.value.scene.screenSpaceCameraController.minimumZoomDistance = 50;
    viewer.value.scene.screenSpaceCameraController.maximumZoomDistance = 20000000;
    viewer.value.scene.screenSpaceCameraController.zoomEventTypes = [
      cesium.CameraEventType.RIGHT_DRAG,
      cesium.CameraEventType.WHEEL,
      cesium.CameraEventType.PINCH,
    ];
    viewer.value.scene.screenSpaceCameraController.tiltEventTypes = [
      cesium.CameraEventType.MIDDLE_DRAG,
      cesium.CameraEventType.PINCH,
      {
        eventType: cesium.CameraEventType.LEFT_DRAG,
        modifier: cesium.KeyboardEventModifier.CTRL,
      },
      {
        eventType: cesium.CameraEventType.RIGHT_DRAG,
        modifier: cesium.KeyboardEventModifier.CTRL,
      },
    ];

    // @ts-ignore
    viewer.value._cesiumWidget._creditContainer.style.display = "none";

    viewer.value.imageryLayers.add(
      new cesium.ImageryLayer(
        new TdtImageryProvider({
          style: "img", //style: vec、cva、img、cia、ter
          key: import.meta.env.VITE_TIANDITU_KEY,
        })
      )
    );

    viewer.value.imageryLayers.addImageryProvider(
      new cesium.UrlTemplateImageryProvider({
        url: `http://t0.tianditu.gov.cn/cia_w/wmts?SERVICE=WMTS&REQUEST=GetTile&VERSION=1.0.0&LAYER=img&STYLE=default&TILEMATRIXSET=w&FORMAT=tiles&TILEMATRIX={z}&TILEROW={y}&TILECOL={x}&tk=${
          import.meta.env.VITE_TIANDITU_KEY
        }`,
        tilingScheme: new cesium.WebMercatorTilingScheme(),
        maximumLevel: 18,
      })
    );

    viewer.value.camera.flyTo({
      destination: cesium.Cartesian3.fromDegrees(121.34, 31.2, 50000.0),
      complete: function callback() {},
    });
  }
});
</script>

<style>
html,
body,
#app {
  width: 100%;
  height: 100%;
  overflow: hidden;
  margin: 0;
  padding: 0;
}

.cesium-container {
  width: 100%;
  height: 100%;
  overflow: hidden;
}
.button-list {
  position: absolute;
  top: 32px;
  right: 32px;
  z-index: 99;
}

.button {
  display: inline-block;
  outline: 0;
  cursor: pointer;
  border: none;
  padding: 0 56px;
  height: 45px;
  line-height: 45px;
  border-radius: 7px;
  background-color: #0070f3;
  color: white;
  font-weight: 400;
  font-size: 16px;
  box-shadow: 0 4px 14px 0 rgb(0 118 255 / 39%);
  transition: background 0.2s ease, color 0.2s ease, box-shadow 0.2s ease;
}
.button:hover {
  background: rgba(0, 118, 255, 0.9);
  box-shadow: 0 6px 20px rgb(0 118 255 / 23%);
}
</style>
