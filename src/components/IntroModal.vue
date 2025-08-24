<template>
  <div v-if="visible" class="intro-modal-wrapper">
    <div class="intro-modal">
      <div class="intro-title">
        <div class="intro-title-text">简介</div>
        <div class="intro-title-close" @click="visible = false">×</div>
      </div>
      <div class="intro-content">
        <p>本项目主要演示内容为在地图可见范围内显示实时航班的功能</p>
        <p>项目由Vue3+Vite+Typescript+Cesium开发</p>
        <p>实时航班数据接口来自FlightRadar24 API</p>
        <p>已知问题：</p>
        <p>1. 受API请求次数限制，航班位置更新较慢，且未能实现路径显示功能</p>
        <p>
          2.
          受天地图API限制，移动地图后服务器会大量返回429错误，部分区块更新会不及时
        </p>
        <p>
          3.
          由于鸟瞰视角不便查看飞机模型实际高度，且3D视角在倾斜情况下获取到的相机边界经纬度有误，故将航班检索范围设定为当前中心点经纬度±0.2度
        </p>
        <p>
          4.
          如点击开始更新后地图较长时间未显示航班，可能是当前区域没有返回航班信息，可以拖拽到航班较为密集的地区尝试
        </p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { defineModel } from "vue";
const visible = defineModel("visible", { default: false });
</script>

<style>
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
