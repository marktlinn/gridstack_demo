<!-- eslint-disable vue/no-unused-vars -->
<template>
  <q-page class="container bg-red">
    <h1>Testing new grid layout</h1>

    <div class="self-center">
      <button type="button" @click="addNewWidget">Add Widget pos [0,0]</button>
      <button type="button" @click="removeLastWidget">
        Remove Last Widget
      </button>
      <br />
      <button type="button" @click="changeFloat">Float: {{ gridFloat }}</button>
      <button type="button" @click="resetGrid">reset</button>
      <br />
    </div>

    <div class="grid-stack">
      <GridWidget
        v-for="w in items"
        :widget="w as GridStackNode"
        :key="w.id"
        @removeWidget="(w: GridStackNode) => remove(w)"
      />
    </div>
  </q-page>
</template>

<script setup lang="ts">
/*  eslint-disable */
// @ts-nocheck
import 'gridstack/dist/gridstack.min.css';
// import 'gridstack/dist/gridstack-extra.min.css'; // needed to alter the number of columns
import { ref, onMounted, nextTick } from 'vue';
import { GridStack, type GridStackNode } from 'gridstack';
import GridWidget from 'components/GridWidget.vue';

let gridFloat = ref(false);
let grid: GridStack | null = null; // DO NOT use ref(null) as proxies GS will break all logic when comparing structures... see https://github.com/gridstack/gridstack.js/issues/2115
const itemHeight = 4;
const itemWidth = 4;
let items = ref<GridStackNode[]>([]);

onMounted(() => {
  grid = GridStack.init({
    // DO NOT use grid.value = GridStack.init(), see above
    float: true,
    cellHeight: '70px',
    minRow: 1,
  });

  if (!grid) {
    console.error('Grid not ready/found on GridArea component mount');
  }
  // grid.on('dragstop', function (_, element) {
  // });

  grid.on('change', onChange);
});

const changeFloat = () => {
  gridFloat.value = !gridFloat.value;
  if (grid) {
    grid.float(gridFloat.value);
  }
};

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
    if (x + w > 12) {
      // 12 is the total number of columns in the grid
      x = 0;
      y += h;
    }
  }
};

// reset function
const resetGrid = () => {
  let x = 0; // Starting x position
  let y = 0; // Starting y position

  // Create a new array for the reset items
  items.value = items.value.map((i) => {
    const newItem = {
      ...i,
      x: x,
      y: y,
      w: itemWidth,
      h: itemHeight,
    };

    // Move to the next position
    x += itemWidth;
    if (x + itemWidth > 12) {
      // If the next item exceeds the total number of columns
      x = 0;
      y += itemHeight;
    }

    return newItem;
  });

  grid.load(items.value);
};

// widget functions
const addNewWidget = () => {
  if (!grid) {
    console.error('cannot add new widget: grid is undefined');
    return;
  }

  const id = crypto.randomUUID();

  const { x, y } = findNextAvailablePosition(
    itemWidth,
    itemHeight,
    items.value
  );

  const node: GridStackNode = {
    x,
    y,
    w: itemWidth,
    h: itemHeight,
    id: `w_${id}`,
  };
  items.value.push(node);

  nextTick(() => grid!.makeWidget(node.id));
};

const remove = (widget: GridStackNode) => {
  if (!grid) {
    console.error('cannot add remove widget: grid is undefined');
    return;
  }

  console.log('items before del: ', items.value);
  items.value = items.value.filter((w) => w.id !== widget.id);

  const selector = `#${widget.id}`;
  grid.removeWidget(selector, false);

  console.log('items after del: ', items.value);
};

const removeLastWidget = () => {
  if (items.value.length <= 0) {
    return;
  }
  const lastItem = items.value[items.value.length - 1] as GridStackNode;
  remove(lastItem);
};
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
