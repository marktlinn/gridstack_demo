<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated>
      <q-toolbar>
        <q-toolbar-title> Grid </q-toolbar-title>
        <div>Quasar v{{ $q.version }}</div>
      </q-toolbar>
    </q-header>

    <q-page-container>
      <GridArea />
    </q-page-container>
  </q-layout>
</template>

<script setup lang="ts">
import GridArea from 'components/GridArea.vue';
</script>
