<template>
  <q-page class="container bg-primary">
    <h1 class="text-center text-white">Testing new grid layout</h1>

    <div class="self-center q-gutter-sm">
      <q-btn color="dark" type="button" @click="addNewWidget" label="Add Widget" />
      <q-btn color="dark" type="button" @click="resetGrid" label="Reset Grid" />
      <q-btn color="dark" type="button" @click="clearGrid" label="Clear Grid" />
      <q-btn color="dark" type="button" @click="setResize"
        :label="`${isResizableDisabled ? 'Enable' : 'Disable'} Item Resize`" />
      <q-btn color="dark" type="button" @click="setDraggable"
        :label="`${isDragDisabled ? 'Enable' : 'Disable'} Item Drag`" />
      <q-select :model-value="sizeSelectModel" :options="Object.keys(SIZES)" label="Grid Item Size"
        @update:model-value="setItemSize" label-color="white" input-style="color: 'white'" rounded />
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
// import 'gridstack/dist/gridstack-extra.min.css'; // needed to alter the number of columns + other extended functionality
import 'gridstack/dist/gridstack.min.css';
import { ref, onMounted, nextTick, onUnmounted } from 'vue';
import { GridStack, type GridStackNode } from 'gridstack';
import GridWidget from 'components/GridWidget.vue';

const COLUMNS = 12; // Default number of columns used for all breaks points and page sizes
const SIZES: Record<string, number> = {
  MIN: 1,
  XS: 2,
  SM: 3,
  M: 4,
  LG: 6,
  MAX: 12,
};
const itemHeight = 4;
const itemWidth = ref(SIZES['M']);

const sizeSelectModel = ref('M');
let grid: GridStack | null = null; // DO NOT use ref(null) as proxies GS will break all logic when comparing structures... see https://github.com/gridstack/gridstack.js/issues/2115
const isDragDisabled = ref(false);
const isResizableDisabled = ref(false);

let items = ref<GridStackNode[]>([]);

onMounted(() => {
  // initialise grid on component mounting
  grid = GridStack.init({
    // DO NOT use grid.value = GridStack.init(), see above
    float: true,
    cellHeight: '70px',
    minRow: 1,
    disableResize: isResizableDisabled.value,
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
  changeItems.forEach((item) => {
    const widget = items.value.find((w) => w.id == item.id);
    if (!widget) {
      console.error(`Widget not found:  ${item.id}`);
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
  return !items.value.some((item) => {
    // Ensure the properties are defined before using them
    if (
      item.x === undefined ||
      item.y === undefined ||
      item.w === undefined ||
      item.h === undefined
    ) {
      return false;
    }

    return (
      x < item.x + item.w && // Check if the right edge of the new item is to the right of the left edge of the existing item
      x + w > item.x &&      // Check if the left edge of the new item is to the left of the right edge of the existing item
      y < item.y + item.h && // Check if the bottom edge of the new item is below the top edge of the existing item
      y + h > item.y         // Check if the top edge of the new item is above the bottom edge of the existing item
    );
  });
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
  if (!grid) {
    console.error('cannot reset grid: grid is undefined');
    return;
  }

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

  const { x, y } = findNextAvailablePosition(itemWidth.value, itemHeight);

  const node = {
    x,
    y,
    w: itemWidth.value,
    h: itemHeight,
    id: `w_${id}`,
  };
  items.value.push(node);

  // Wait for nextTick i.e. until after the any current DOM updates have been flushed.
  nextTick(() => grid?.makeWidget(node.id));
};

const setItemSize = (s: string) => {
  itemWidth.value = SIZES[s];
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
  isResizableDisabled.value = !isResizableDisabled.value;

  // passing the opposite of isResizeable, as true activates resize and false deactivates it.
  grid?.enableResize(!isResizableDisabled.value);
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
