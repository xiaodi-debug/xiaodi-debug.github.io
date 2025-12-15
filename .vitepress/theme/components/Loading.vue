<template>
  <Teleport to="body">
    <Transition name="fade" mode="out-in">
      <div v-if="loadingStatus" class="loading">
        <div class="spinner" />
        <span class="text">加载中…</span>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { storeToRefs } from "pinia";
import { mainStore } from "@/store";

const store = mainStore();
const { loadingStatus } = storeToRefs(store);
</script>

<style lang="scss" scoped>
.loading {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: rgba(0, 0, 0, 0.45);
  backdrop-filter: blur(6px);
  z-index: 9999;
  flex-direction: column;
  gap: 10px;
  .spinner {
    width: 48px;
    height: 48px;
    border-radius: 50%;
    border: 4px solid rgba(255, 255, 255, 0.25);
    border-top-color: #ffffff;
    animation: spin 0.7s linear infinite;
  }
  .text {
    font-size: 14px;
    white-space: nowrap;
  }
}
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
</style>
