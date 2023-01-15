<script setup>
import { ref, onMounted } from 'vue'

const canvas = ref(null);
const ctx = ref(null);
const gridLines = ref(null);
const toggle = ref(true);
const gridPixelLength = ref(null);
const gridSize = ref(8);
const colorSelected = ref('#009578');
const selectGridSize = ref([]);
const downloadOptions = ref([]);
const isFillToolSelected = ref(false);

onMounted(() => {
  setCanvas();
  setGridLines();
  selectGridSize.value = [
    {size: 8, click: () => selectGridSizeButton(8)},
    {size: 12, click: () => selectGridSizeButton(12)},
    {size: 16, click: () => selectGridSizeButton(16)},
    {size: 32, click: () => selectGridSizeButton(32)},
  ];
  downloadOptions.value = [
    {type: 'jpg', click: () => downloadPixelArt('jpg')},
    {type: 'png', click: () => downloadPixelArt('png')},
    {type: 'gif', click: () => downloadPixelArt('gif')},
  ];
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

  cellXStart.value = cellX * gridPixelLength.value;
  cellYStart.value = cellY * gridPixelLength.value;

  if (e.ctrlKey) {
    if (currentColor) {
      colorSelected.value = currentColor;
    }
  } else {
    if (!isFillToolSelected.value) {
      paintCell(cellXStart.value, cellYStart.value);
    } else {
      useFillTool(cellXStart.value, cellYStart.value, colorSelected.value);
    }
  }
}

const cellXStart = ref(0);
const cellYStart = ref(0);

function paintCell(x, y) {
  ctx.value.fillStyle = colorSelected.value;
  ctx.value.fillRect(x, y, gridPixelLength.value, gridPixelLength.value);
  colorHistory.value[`${x}_${y}`] = colorSelected.value
}

function hexToRgb(hex) {
  let result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
  return result ? {
    r: parseInt(result[1], 16),
    g: parseInt(result[2], 16),
    b: parseInt(result[3], 16)
  } : null;
}

function useFillTool(startX, startY, currentColor) {

  let color = hexToRgb(currentColor)

  //get imageData
  let colorLayer = ctx.value.getImageData(
    0,
    0,
    canvas.value.width,
    canvas.value.height
  );

  let startPos = (startY * canvas.value.width + startX) * 4;

  //clicked color
  let startR = colorLayer.data[startPos];
  let startG = colorLayer.data[startPos + 1];
  let startB = colorLayer.data[startPos + 2];
  //exit if color is the same
  if (
    color.r === startR &&
    color.g === startG &&
    color.b === startB
  ) {
    return;
  }
  //Start with click coords
  let pixelStack = [[startX, startY]];
  let newPos, x, y, pixelPos, reachLeft, reachRight;
  floodFill();
  function floodFill() {
    newPos = pixelStack.pop();
    x = newPos[0];
    y = newPos[1];

    //get current pixel position
    pixelPos = (y * canvas.value.width + x) * 4;
    // Go up as long as the color matches and are inside the canvas
    while (y >= 0 && matchStartColor(pixelPos)) {
      y--;
      pixelPos -= canvas.value.width * 4;
    }
    //Don't overextend
    pixelPos += canvas.value.width * 4;
    y++;
    reachLeft = false;
    reachRight = false;
    // Go down as long as the color matches and in inside the canvas
    while (y < canvas.value.height && matchStartColor(pixelPos)) {
      colorPixel(pixelPos);

      if (x > 0) {
        if (matchStartColor(pixelPos - 4)) {
          if (!reachLeft) {
            //Add pixel to stack
            pixelStack.push([x - 1, y]);
            reachLeft = true;
          }
        } else if (reachLeft) {
          reachLeft = false;
        }
      }

      if (x < canvas.value.width - 1) {
        if (matchStartColor(pixelPos + 4)) {
          if (!reachRight) {
            //Add pixel to stack
            pixelStack.push([x + 1, y]);
            reachRight = true;
          }
        } else if (reachRight) {
          reachRight = false;
        }
      }
      y++;
      pixelPos += canvas.value.width * 4;
    }

    if (pixelStack.length) {
      floodFill();
    }
  }

  //render floodFill result
  ctx.value.putImageData(colorLayer, 0, 0);

  //helpers
  function matchStartColor(pixelPos) {
    let r = colorLayer.data[pixelPos];
    let g = colorLayer.data[pixelPos + 1];
    let b = colorLayer.data[pixelPos + 2];
    return r === startR && g === startG && b === startB;
  }

  function colorPixel(pixelPos) {
    colorLayer.data[pixelPos] = color.r;
    colorLayer.data[pixelPos + 1] = color.g;
    colorLayer.data[pixelPos + 2] = color.b;
    colorLayer.data[pixelPos + 3] = 255;
  }
}

function clearCanvas() {
  ctx.value.fillStyle = '#ffffff';
  ctx.value.fillRect(0, 0, canvas.value.width, canvas.value.height);
}


const squaresLines = ref([])

function setGridLines() {
  gridLines.value.style.width = `${canvas.value.width}px`;
  gridLines.value.style.height = `${canvas.value.height}px`;
  gridLines.value.style.gridTemplateColumns = `repeat(${gridSize.value}, 1fr)`;
  gridLines.value.style.gridTemplateRows = `repeat(${gridSize.value}, 1fr)`;
  squaresLines.value = gridSize.value ** 2
}

function toggleLines() {
  toggle.value = !toggle.value
  gridLines.value.style.display = toggle.value ? null : "none";
}

function selectGridSizeButton(size) {
  gridSize.value = size
  clearCanvas();
  setCanvas();
  setGridLines();
}

function downloadPixelArt(type) {
  let anchor = document.createElement("a");
  anchor.download = "download." + type;
  anchor.href = canvas.value.toDataURL("pixel-art/" + type);
  anchor.click();
  anchor.remove();
}

</script>

<template>
  <h2>Pixel Art App</h2>
  
  <section>
    <div>
      <div 
        ref="gridLines"
        id="gridLines"
      >
      <div 
        style="border: 1px solid rgba(0, 0, 0, 0.1);"
        v-for="square in squaresLines"
        :key="square"
      >
      </div>
      </div>
      <canvas
        ref="canvas"
        width="800"
        height="800"
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

    <button
      v-for="b in selectGridSize"
      :key="b"
      @click="b.click"
      >
      {{ b.size }}
    </button>

    <button @click="isFillToolSelected = !isFillToolSelected">
      Fill tool
    </button>

    <button
      v-for="download in downloadOptions"
      :key="download"
      @click="download.click"
      >
      {{ download.type }}
    </button>

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

