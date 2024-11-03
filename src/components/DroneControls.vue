<script setup lang="ts">
interface Props {
  onReset: () => void;
  onSphere: () => void;
  onHelix: () => void;
}

defineProps<Props>();
</script>

<template>
  <div class="controls">
    <button @click="onReset">Reset All</button>
    <button @click="onSphere">Sphere Formation</button>
    <button @click="onHelix">Double Helix</button>
  </div>
</template>

<style scoped>
.controls {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 1000;
  display: flex;
  gap: 10px;
}

button {
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: white;
  padding: 10px 20px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.3s ease;
  backdrop-filter: blur(10px);
}

button:hover {
  background: rgba(255, 255, 255, 0.2);
  border-color: rgba(255, 255, 255, 0.3);
}
</style>