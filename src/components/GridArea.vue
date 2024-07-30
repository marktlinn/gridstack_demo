<!-- eslint-disable vue/no-unused-vars -->
<template>
  <q-page class="container bg-purple">
    <h1>Testing new grid layout</h1>

    <div class="self-center q-gutter-sm">
      <q-btn color="dark" type="button" @click="addNewWidget" label="Add Widget" />
      <q-btn color="dark" type="button" @click="resetGrid" label="Reset Grid" />
      <q-btn color="dark" type="button" @click="clearGrid" label="Clear Grid" />
      <q-btn color="dark" type="button" @click="setResize"
        :label="`${isResizable ? 'Disable' : 'Enable'} Item Resize`" />
      <q-btn color="dark" type="button" @click="setDraggable"
        :label="`${isDragDisabled ? 'Enable' : 'Disable'} Item Drag`" />
      <q-select color="dark" :model-value="sizeSelectModel" :options="Object.keys(sizes)" label="Grid Item Size"
        @update:model-value="setItemSize" />
      <br />
    </div>

    <div class="grid-stack">
      <GridWidget v-for="w in items" :widget="w as GridStackNode" :key="w.id"
        @removeWidget="(w: GridStackNode) => remove(w)">
        <div class="text-h6">Hello from slot in {{ w.id }}</div>
      </GridWidget>
    </div>
  </q-page>
</template>

<script setup lang="ts">
/*  eslint-disable */
// @ts-nocheck
import 'gridstack/dist/gridstack.min.css';
// import 'gridstack/dist/gridstack-extra.min.css'; // needed to alter the number of columns + other extended functionality
import { ref, onMounted, nextTick, onUnmounted } from 'vue';
import { GridItemHTMLElement, GridStack, type GridStackNode } from 'gridstack';
import GridWidget from 'components/GridWidget.vue';

let grid: GridStack | null = null; // DO NOT use ref(null) as proxies GS will break all logic when comparing structures... see https://github.com/gridstack/gridstack.js/issues/2115
const COLUMNS = 12; // Default number of columns used for all breaks points and page sizes
const isDragDisabled = ref(false);
const isResizable = ref(true);

const sizes = { MIN: 1, XS: 2, SM: 3, M: 4, LG: 6, MAX: 12 };
const sizeSelectModel = ref();
const itemHeight = 4;
const itemWidth = ref(sizes['M']);
let items = ref<GridStackNode[]>([]);

onMounted(() => {
  // initialise grid on component mounting
  grid = GridStack.init({
    // DO NOT use grid.value = GridStack.init(), see above
    float: true,
    cellHeight: '70px',
    minRow: 1,
    resizable: isResizable.value,
    disableDrag: isDragDisabled.value,
  });

  if (!grid) {
    console.error('Grid not ready/found on GridArea component mount');
  }

  // setup any event listeners for the grid

  // grid.on('dragstop', function (_, element) {
  // });

  grid.on('change', onChange);
});
/**
 * OnChange is used in conjunction with any native `change` event is called on the Grid.
 * Change events occur when widgets change their position/size due to constrain  of the grid
 * or when direct changes are made by the user on a specfic widget.
 */
const onChange = (_: Event, changeItems: GridStackNode[]) => {
  // update item position
  changeItems.forEach((item: GridStackNode) => {
    const widget = items.value.find((w) => w.id == item.id);
    if (!widget) {
      alert('Widget not found: ' + item.id);
      return;
    }

    widget.x = item.x;
    widget.y = item.y;
    widget.w = item.w;
    widget.h = item.h;
  });
};

// Position functions
const isPositionAvailable = (
  x: number,
  y: number,
  w: number,
  h: number
): boolean => {
  // Check if the position collides with any existing item
  return !items.value.some(
    (item) =>
      x < item.x + item.w &&
      x + w > item.x &&
      y < item.y + item.h &&
      y + h > item.y
  );
};

const findNextAvailablePosition = (
  w: number,
  h: number
): { x: number; y: number } => {
  // Start from the top-left corner of the grid
  let x = 0;
  let y = 0;

  // Iterate through rows and columns to find the next available position
  while (true) {
    // Check if the current position is available
    if (isPositionAvailable(x, y, w, h)) {
      return { x, y };
    }

    // Move to the next position
    x += w;
    if (x + w > COLUMNS) {
      // COLUMNS is the total number of columns in the grid - default is 12.
      x = 0;
      y += h;
    }
  }
};

// reset function
const resetGrid = () => {
  let x = 0; // Starting x position - having been reset
  let y = 0; // Starting y position - having been reset

  // Create a new array for the reset items
  items.value = items.value.map((i) => {
    const newItem = {
      ...i,
      x: x,
      y: y,
      w: itemWidth.value,
      h: itemHeight,
    };

    // Move to the next position
    x += itemWidth.value;
    if (x + itemWidth.value > COLUMNS) {
      // If the next item exceeds the total number of columns - default 12.
      x = 0;
      y += itemHeight;
    }

    return newItem;
  });

  // load the the updated items array into the grid, which updates all elements in the layout.
  grid.load(items.value);
};

const clearGrid = () => {
  if (!grid) {
    console.error('cannot clear grid: grid is undefined');
    return;
  }

  grid?.removeAll();
  items.value = [];
};

// widget functions
const addNewWidget = () => {
  if (!grid) {
    console.error('cannot add new widget: grid is undefined');
    return;
  }

  const id = crypto.randomUUID();

  const { x, y } = findNextAvailablePosition(
    itemWidth.value,
    itemHeight,
    items.value
  );

  const node: GridStackNode = {
    x,
    y,
    w: itemWidth.value,
    h: itemHeight,
    id: `w_${id}`,
  };
  items.value.push(node);

  // Wait for nextTick i.e. until after the any current DOM updates have been flushed.
  nextTick(() => grid!.makeWidget(node.id));
};

const setItemSize = (s) => {
  itemWidth.value = sizes[s];
  sizeSelectModel.value = s;
  resetGrid();
};

const remove = (widget: GridStackNode) => {
  if (!grid) {
    console.error('cannot add remove widget: grid is undefined');
    return;
  }

  items.value = items.value.filter((w) => w.id !== widget.id);

  const selector = `#${widget.id}`;
  grid.removeWidget(selector, false);
};

const setResize = () => {
  isResizable.value = !isResizable.value;
  grid?.enableResize(isResizable.value);
};

const setDraggable = () => {
  isDragDisabled.value = !isDragDisabled.value;

  // passing the opposite of dragDisabled, as true activates drag and false deactivates it.
  grid?.enableMove(!isDragDisabled.value);
};

onUnmounted(() => {
  // cleanup destroy grid DOM nodes
  if (!grid) {
    console.error('cannot unmount grid: grid is undefined');
    return;
  }

  // grid.save() native method to save grid state that could potentially be used alongside existing localStorage functionality
  grid.destroy();
  grid = null;
});
</script>

<style scoped>
.container {
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.grid-stack {
  flex: 1;
  /* This makes sure the grid-stack element grows to fill the container */
  background-color: grey;
}
</style>
