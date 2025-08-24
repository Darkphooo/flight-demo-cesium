<template>
  <div class="cesium">
    <div ref="cesium-container" class="cesium-container"></div>
    <slot :isFetching="isFetching"></slot>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref, useTemplateRef } from "vue";

import A320Model from "@/assets/a320.glb";
import { TdtImageryProvider } from "@cesium-china/cesium-map";
import axios from "axios";
import * as cesium from "cesium";

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

const cesiumRef = useTemplateRef("cesium-container");
const viewer = ref();
const isFetching = ref(false);
const timer = ref();

function getFlightData() {
  const center = viewer.value.camera.pickEllipsoid(
    new cesium.Cartesian2(
      viewer.value.canvas.clientWidth / 2,
      viewer.value.canvas.clientHeight / 2
    )
  );
  const currentPos = cesium.Ellipsoid.WGS84.cartesianToCartographic(center);
  const currentLon = (currentPos.longitude * 180) / Math.PI;
  const currentLat = (currentPos.latitude * 180) / Math.PI;
  axios
    .get(`https://fr24api.flightradar24.com/api/live/flight-positions/light`, {
      params: {
        bounds: `${currentLat + 0.2},${currentLat - 0.2},${currentLon - 0.2},${
          currentLon + 0.2
        }`,
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
      label: {
        text: flight.callsign,
        font: "500 14px Helvetica,Arial,sans-serif",
        scale: 1,
        fillColor: cesium.Color.WHITE,
        heightReference: cesium.HeightReference.CLAMP_TO_3D_TILE,
        verticalOrigin: cesium.VerticalOrigin.BOTTOM,
        showBackground: true,
      },
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
  timer.value = setInterval(getFlightData, 6000);
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

    viewer.value.camera.flyTo({
      destination: cesium.Cartesian3.fromDegrees(118.79, 32.05, 100000.0),
      complete: function callback() {},
    });
  }
});

defineExpose({ refreshFlightData, stopRefreshFlightData });
</script>

<style>
.cesium {
  width: 100%;
  height: 100%;
  position: relative;
}

.cesium-container {
  width: 100%;
  height: 100%;
  overflow: hidden;
}
</style>