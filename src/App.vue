<template>
  <div ref="cesiumRef" class="cesium-container"></div>
  <div class="button-list">
    <div class="button" @click="introVisible = true">简介</div>
    <div v-if="!isFetching" class="button" @click="refreshFlightData">
      开始更新当前区域航班
    </div>
    <div v-else class="button" @click="stopRefreshFlightData">
      停止更新当前区域航班
    </div>
  </div>
  <div v-if="introVisible" class="intro-modal-wrapper">
    <div class="intro-modal">
      <div class="intro-title">
        <div class="intro-title-text">简介</div>
        <div class="intro-title-close" @click="introVisible = false">×</div>
      </div>
      <div class="intro-content">
        <p>本项目主要演示内容为在地图可见范围内显示实时航班的功能</p>
        <p>项目由Vue3+Vite+Typescript+Cesium开发</p>
        <p>实时航班数据接口来自FlightRadar24 API</p>
        <p>已知问题：</p>
        <p>1. 受API请求次数限制，航班位置更新较慢，且未能实现路径显示功能</p>
        <p>
          2. 受天地图API限制，移动地图后服务器会大量返回429错误，部分区块更新会不及时
        </p>
        <p>
          3. 由于鸟瞰视角不便查看飞机模型实际高度，且3D视角在倾斜情况下获取到的相机边界经纬度有误，故将航班检索范围设定为当前中心点经纬度±0.2度
        </p>
        <p>
          4. 如点击开始更新后地图较长时间未显示航班，可能是当前区域没有返回航班信息，可以拖拽到航班较为密集的地区尝试
        </p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import A320Model from "@/assets/a320.glb";
import { TdtImageryProvider } from "@cesium-china/cesium-map";
import axios from "axios";
import * as cesium from "cesium";
import { onMounted, ref, useTemplateRef } from "vue";

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
const introVisible = ref(false);
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
      destination: cesium.Cartesian3.fromDegrees(118.79, 32.05, 100000.0),
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
  display: flex;
  flex-direction: column;
  gap: 16px;
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
  text-align: center;
  box-shadow: 0 4px 14px 0 rgb(0 118 255 / 39%);
  transition: background 0.2s ease, color 0.2s ease, box-shadow 0.2s ease;
}
.button:hover {
  background: rgba(0, 118, 255, 0.9);
  box-shadow: 0 6px 20px rgb(0 118 255 / 23%);
}
.intro-modal-wrapper {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background-color: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 199;
}
.intro-modal {
  width: 600px;
  height: auto;
  background-color: #fff;
  display: flex;
  flex-direction: column;
}
.intro-title {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  padding: 16px;
  border-bottom: 1px solid #e5e5e5;
}
.intro-title-text {
  font-size: 24px;
  font-weight: bold;
}
.intro-title-close {
  width: 24px;
  height: 24px;
  cursor: pointer;
}
.intro-content {
  padding: 16px;
}
</style>
