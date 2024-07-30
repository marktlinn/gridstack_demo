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
      {{ widget }}
      <q-card>
        <div class="text-h6">Hello from {{ widget.id }}</div>
      </q-card>
    </div>
  </div>
</template>

<script setup lang="ts">
/*  eslint-disable */
// @ts-nocheck
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
