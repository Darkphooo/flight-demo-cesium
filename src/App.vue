<template>
  <CesiumMap v-slot="slotProps" ref="cesium">
    <div class="button-list">
      <div class="button" @click="introVisible = true">简介</div>
      <div
        v-if="!slotProps.isFetching"
        class="button"
        @click="cesiumRef?.refreshFlightData"
      >
        开始更新当前区域航班
      </div>
      <div v-else class="button" @click="cesiumRef?.stopRefreshFlightData">
        停止更新当前区域航班
      </div>
    </div>
  </CesiumMap>
  <IntroModal v-model:visible="introVisible" />
</template>

<script setup lang="ts">
import { ref, useTemplateRef } from "vue";
import IntroModal from "./components/IntroModal.vue";
import CesiumMap from "./components/CesiumMap.vue";

const cesiumRef = useTemplateRef("cesium");

const introVisible = ref(false);
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

.button-list {
  position: fixed;
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
</style>
