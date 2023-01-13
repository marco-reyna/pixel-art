<script setup>
import { ref, onMounted, watch } from 'vue'

const canvas = ref(null);
const ctx = ref(null);
const gridLines = ref(null);
const toggle = ref(false);
const gridPixelLength = ref(null);
const gridSize = ref(8);
const colorSelected = ref('#009578');

onMounted(() => {
  setCanvas()
  setGridLines()
});

function setCanvas() {
  ctx.value = canvas.value.getContext('2d');
  ctx.value.fillStyle = '#ffffff';
  ctx.value.fillRect(0, 0, canvas.value.width, canvas.value.height);
  gridPixelLength.value = canvas.value.width / gridSize.value
}

const colorHistory = ref({});

function paintOnCanvas(e) {
  const boundingRect = canvas.value.getBoundingClientRect();
  const x = e.clientX - boundingRect.left;
  const y = e.clientY - boundingRect.top;
  const cellX = Math.floor(x / gridPixelLength.value);
  const cellY = Math.floor(y / gridPixelLength.value);
  const currentColor = colorHistory.value[`${cellX}_${cellY}`];

  if (e.ctrlKey) {
    if (currentColor) {
      colorSelected.value = currentColor;
    }
  } else {
    paintCell(cellX, cellY);
  }
}

const cellXStart = ref(0);
const cellYStart = ref(0);

function paintCell(x, y) {
  cellXStart.value = x * gridPixelLength.value
  cellYStart.value = y * gridPixelLength.value

  ctx.value.fillStyle = colorSelected.value;
  ctx.value.fillRect(cellXStart.value, cellYStart.value, gridPixelLength.value, gridPixelLength.value);
  colorHistory.value[`${x}_${y}`] = colorSelected.value
}

function clearCanvas() {
  ctx.value.fillStyle = '#ffffff';
  ctx.value.fillRect(0, 0, canvas.value.width, canvas.value.height);
}

function setGridLines() {
  gridLines.value.style.width = `${canvas.value.width}px`;
  gridLines.value.style.height = `${canvas.value.height}px`;
  gridLines.value.style.gridTemplateColumns = `repeat(${gridSize.value}, 1fr)`;
  gridLines.value.style.gridTemplateRows = `repeat(${gridSize.value}, 1fr)`;

  console.log(gridSize.value ** 2);

  [...Array(gridSize.value ** 2)].forEach(() =>
    gridLines.value.insertAdjacentHTML(
      "beforeend", "<div style='border: 1px solid rgba(0, 0, 0, 0.1);'></div>"
    )
  );

}

function toggleLines() {
  toggle.value = !toggle.value
  gridLines.value.style.display = toggle.value ? null : "none";
}

// function increaseGridSize() {
//   gridSize.value = 12
//   clearCanvas();
//   setCanvas();
//   setGridLines();
// }

</script>

<template>
  <h2>Pixel Art App</h2>
  
  <section>
    <div>
      <div 
        ref="gridLines"
        id="gridLines"
      ></div>
      <canvas
        ref="canvas"
        width="600"
        height="600"
        @mousedown="paintOnCanvas"
      />
    </div>
    <div>

    </div>
    <div class="options-block">
      <div>
        <label for="colorSelected">
          Select color: 
        </label>
        <input
          id="colorSelected"
          v-model="colorSelected"
          :style="colorSelected"
          type="color"
        >
      </div>
      <button
        @click="toggleLines"
      >
        Show lines
      </button>
      <button
        @click="clearCanvas"
        type="button">
        Clear canvas
      </button>
    </div>
    <p>
      Ctrl + click to copy a color
    </p>
    <div>
      <button @click="increaseGridSize">
       +
      </button>
      <p>
        Grid size: {{ gridSize }}
      </p>
    </div>
  </section>

</template>

<style scoped>

  canvas {
    cursor: pointer;
  }
  
  #gridLines {
    display: grid;
    pointer-events: none;
    position: absolute;
  }

  .options-block {
    display: flex;
    gap: 20px;
  }

</style>

