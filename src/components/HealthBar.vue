<script setup lang="ts">
import { computed } from 'vue';

// 1. Define the props
const props = defineProps<{
  current: number;
  max: number;
  color: string;
}>();

// 2. Calculate the width
const healthPercentage = computed(() => {
  const percentage = (props.current / props.max) * 100;
  return `${Math.max(0, percentage)}%`; 
});
</script>

<template>
  <div class="health-bar-container">
    <div 
      class="health-bar-fill" 
      :style="{ width: healthPercentage, backgroundColor: color }"
    ></div>

    <div class="health-text">
      {{ current }} / {{ max }}
    </div>
  </div>
</template>

<style scoped>
.health-bar-container {
  width: 100%;
  height: 35px;
  background-color: #333;
  border-radius: 6px;
  position: relative;
  overflow: hidden;
}

.health-bar-fill {
  height: 100%;
  transition: width 0.2s ease-out; 
}

.health-text {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  color: white;
  font-weight: bold;
}
</style>