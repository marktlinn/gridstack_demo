<template>
  <div
    class="grid-stack-item"
    :gs-x="widget.x"
    :gs-y="widget.y"
    :gs-w="widget.w"
    :gs-h="widget.h"
    :gs-id="widget.id"
    :id="widget.id"
    :key="widget.id"
  >
    <div class="grid-stack-item-content">
      <button @click="$emit('removeWidget', widget)">remove</button>
      <button @click="$emit('toggleMove', widget)">toggleMove</button>
      <button @click="$emit('toggleResize', widget)">toggleResize</button>
      {{ widget }}
      <q-card>
        <slot />
      </q-card>
    </div>
  </div>
</template>

<script setup lang="ts">
import { type GridStackNode } from 'gridstack';
defineProps<{ widget: GridStackNode }>();
</script>

<style scoped>
.grid-stack-item {
  background-color: blueviolet;
  width: 100px;
  height: 100px;
  border: 1px solid green;
}
</style>
