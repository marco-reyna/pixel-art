<script setup>
import { ref, onMounted } from 'vue'

const canvas = ref(null);
const ctx = ref(null);
const gridLines = ref(null);
const toggle = ref(true);
const gridPixelLength = ref(null);
const gridSize = ref(8);
const colorSelected = ref('#ffe54c');
const selectGridSize = ref([]);
const downloadOptions = ref([]);
const isFillToolSelected = ref(false);
const colorHistory = ref({});

// SETUP CANVAS:
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
  ctx.value.fillStyle = 'transparent';
  ctx.value.fillRect(0, 0, canvas.value.width, canvas.value.height);
  gridPixelLength.value = canvas.value.width / gridSize.value
}

// PAINT ON CANVAS - CELL:
function paintOnCanvas(e) {
  // get the cell clicked
  const boundingRect = canvas.value.getBoundingClientRect();
  const x = e.clientX - boundingRect.left;
  const y = e.clientY - boundingRect.top;
  const cellX = Math.floor(x / gridPixelLength.value);
  const cellY = Math.floor(y / gridPixelLength.value);
  const currentColor = colorHistory.value[`${cellX}_${cellY}`]

  if (e.ctrlKey) {
    // to copy an used color from canavas
    if (currentColor) colorSelected.value = currentColor;
  } else {
    cellXStart.value = cellX * gridPixelLength.value;
    cellYStart.value = cellY * gridPixelLength.value;
    if (!isFillToolSelected.value) {
      paintCell(cellX, cellY, cellXStart.value, cellYStart.value);
    } else {
      useFillTool(cellXStart.value, cellYStart.value, colorSelected.value);
    }
  }
}
const cellXStart = ref(0);
const cellYStart = ref(0);
function paintCell(cellX, cellY, startX, startY) {
  ctx.value.fillStyle = colorSelected.value;
  ctx.value.fillRect(startX, startY, gridPixelLength.value, gridPixelLength.value);
  colorHistory.value[`${cellX}_${cellY}`] = colorSelected.value
}

// FLOOD FILL FUNCTIONALITY:
function useFillTool(startX, startY, currentColor) {

  let color = hexToRgb(currentColor)
  let colorLayer = ctx.value.getImageData(
    0,
    0,
    canvas.value.width,
    canvas.value.height
  );

  let startPos = (startY * canvas.value.width + startX) * 4;

  // clicked color
  let startR = colorLayer.data[startPos];
  let startG = colorLayer.data[startPos + 1];
  let startB = colorLayer.data[startPos + 2];
  // exit if color is the same
  if (
    color.r === startR &&
    color.g === startG &&
    color.b === startB
  ) {
    return;
  }
  // start with click coords
  let pixelStack = [[startX, startY]];
  let newPos, x, y, pixelPos, reachLeft, reachRight;
  floodFill();
  function floodFill() {
    newPos = pixelStack.pop();
    x = newPos[0];
    y = newPos[1];

    // get current pixel position
    pixelPos = (y * canvas.value.width + x) * 4;
    // go up as long as the color matches and are inside the canvas
    while (y >= 0 && matchStartColor(pixelPos)) {
      y--;
      pixelPos -= canvas.value.width * 4;
    }
    // don't overextend
    pixelPos += canvas.value.width * 4;
    y++;
    reachLeft = false;
    reachRight = false;
    // go down as long as the color matches and in inside the canvas
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
            // add pixel to stack
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

  // render floodFill
  ctx.value.putImageData(colorLayer, 0, 0);

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
// convert hex color to rgb
function hexToRgb(hex) {
  let result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
  return result ? {
    r: parseInt(result[1], 16),
    g: parseInt(result[2], 16),
    b: parseInt(result[3], 16)
  } : null;
}

// CLEAR CANVAS:
function clearCanvas() {
  ctx.value.fillStyle = 'transparent';
  ctx.value.clearRect(0, 0, canvas.value.width, canvas.value.height);
}

// HANDLING GRID LINES:
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

// SELECT GRID SIZE:
function selectGridSizeButton(size) {
  gridSize.value = size
  clearCanvas();
  setCanvas();
  setGridLines();
}

// DOWNLOAD:
function downloadPixelArt(type) {
  let anchor = document.createElement("a");
  anchor.download = "download." + type;
  anchor.href = canvas.value.toDataURL("pixel-art/" + type);
  anchor.click();
  anchor.remove();
}

</script>

<template>
  <body>      
    <!-- configurations/options section -->
    <section class="configurations-wrapper hide-on-small-screens">

      <h1 :style="{color: colorSelected}">
        Pixel Art App
      </h1>

      <input
        id="colorSelected"
        class="palette"
        v-model="colorSelected"
        type="color"
      >

      <div class="select-color">
        <p>
          Click above
          <span>
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 384 512">
              <path d="M214.6 41.4c-12.5-12.5-32.8-12.5-45.3 0l-160 160c-12.5
              12.5-12.5 32.8 0 45.3s32.8 12.5 45.3 0L160 141.2V448c0 17.7 14.3
              32 32 32s32-14.3 32-32V141.2L329.4 246.6c12.5 12.5 32.8 12.5 45.3
              0s12.5-32.8 0-45.3l-160-160z"/>
            </svg>
          </span>
          to chose a color from the palette
        </p>
      </div>

      <div class="buttons-wrapper">
        <button
          @click="isFillToolSelected = false"
          class="config-button"
          :style="!isFillToolSelected ? 'background-color: #ffffff;' : ''"
        >
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" class="icon" v-show="!isFillToolSelected">
            <path d="M362.7 19.3L314.3 67.7 444.3 197.7l48.4-48.4c25-25 25-65.5 0-90.5L453.3 19.3c-25-25-65.5-25-90.5
            0zm-71 71L58.6 323.5c-10.4 10.4-18 23.3-22.2 37.4L1 481.2C-1.5 489.7 .8 498.8 7 505s15.3 8.5 23.7
            6.1l120.3-35.4c14.1-4.2 27-11.8 37.4-22.2L421.7 220.3 291.7 90.3z"/>
          </svg>
          <span v-show="isFillToolSelected">Pen Tool</span>
        </button>

        <button
          @click="isFillToolSelected = true"
          class="config-button"
          :style="isFillToolSelected ? 'background-color: #ffffff;' : ''"
        >
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 576 512" class="icon" v-show="isFillToolSelected">
            <path d="M41.4 9.4C53.9-3.1 74.1-3.1 86.6 9.4L168 90.7l53.1-53.1c28.1-28.1 73.7-28.1
            101.8 0L474.3 189.1c28.1 28.1 28.1 73.7 0 101.8L283.9 481.4c-37.5 37.5-98.3 37.5-135.8
            0L30.6 363.9c-37.5-37.5-37.5-98.3 0-135.8L122.7 136 41.4 54.6c-12.5-12.5-12.5-32.8 0-45.3zm176
            221.3L168 181.3 75.9 273.4c-4.2 4.2-7 9.3-8.4 14.6H386.7l42.3-42.3c3.1-3.1 3.1-8.2 0-11.3L277.7
            82.9c-3.1-3.1-8.2-3.1-11.3 0L213.3 136l49.4 49.4c12.5 12.5 12.5 32.8 0 45.3s-32.8 12.5-45.3 0zM512
            512c-35.3 0-64-28.7-64-64c0-25.2 32.6-79.6 51.2-108.7c6-9.4 19.5-9.4 25.5 0C543.4 368.4 576 422.8
            576 448c0 35.3-28.7 64-64 64z"/>
          </svg>
          <span v-show="!isFillToolSelected">Fill Tool</span>
        </button>
      </div>

      <div class="button-full-width">
        <button
          @click="toggleLines"
          class="config-button"
          :style="toggle ? 'background-color: #ffffff;' : ''"
        >
          {{ !toggle ? 'Show Lines' : 'Hide Lines'  }}
        </button>
      </div>

      <div class="copy-color">
        <p>
          Press
          <span class="yellow-text">Ctrl+Click</span>
          to copy a color from the canvas
        </p>
      </div>

      <hr/>

      <h2>Select grid size:</h2>
      <div class="grid-size-buttons-wrapper">
        <button
          v-for="b in selectGridSize"
          :key="b"
          @click="b.click"
          class="small-buttons"
          :style="gridSize === b.size ?
          'background-color: #ffe54c; color: #242424; border-color: #242424;'
          : ''"
          >
          {{ b.size }}
        </button>
      </div>

      <h2>Download options:</h2>
      <div class="download-buttons-wrapper">
        <button
          v-for="download in downloadOptions"
          :key="download"
          @click="download.click"
          class="small-buttons"
          >
          {{ download.type }}
        </button>
      </div>

      <div class="button-full-width">
        <button
          @click="clearCanvas"
          class="button-clear-canvas"
        >
          Clear canvas
        </button>
      </div>
    </section>

    <!-- canvas section -->
    <section class="hide-on-small-screens">
      <div ref="gridLines" id="gridLines">
        <div 
          style="border: 0.5px solid rgba(255,255,255,0.2)"
          v-for="square in squaresLines"
          :key="square"
        ></div>
      </div>
      <canvas ref="canvas" width="800" height="800" @mousedown="paintOnCanvas" />
    </section>

    <!-- small screens message-->
    <section class="coming-soon-message">
      <img src="../assets/pizza.png" alt="image of a pizza's slice">
      <h3>App for small screens <span class="yellow-text">coming soon</span></h3>
    </section>    

  </body>
</template>

<style scoped>

#gridLines {
  display: grid;
  pointer-events: none;
  position: absolute;
}

body {
  display: flex;
  gap: 50px;
}

canvas {
  cursor: pointer;
  border-radius: 2px;
  padding: 0;
}

h1 {
  font-size: 40px;
  line-height: 40px;
  font-weight: 700;
  text-transform: uppercase;
  margin-bottom: -12px;
  letter-spacing: 5px;
  text-align: center;
}

h2 {
  color: #ffffff;
}

h3 {
  color: #ffffff;
}

button {
  cursor: pointer;
}

p {
  color: #ffffff;
}

hr {
  border: 2px dashed #FFF;
  width: 100%;
  margin-bottom: 35px;
}

.configurations-wrapper {
  padding-top: 20px;
}

.palette {
  -webkit-appearance: none;
  -moz-appearance: none;
  appearance: none;
  cursor: pointer;
  width: 100%;
  height: 10%;
  border: none;
  background-color: transparent;
  margin-bottom: 10px;
}

.palette::-webkit-color-swatch {
  border-radius: 1px;
  border: none;
}
.palette::-moz-color-swatch {
  border-radius: 1px;
  border: none;
}

.icon {
  width: 20px;
}

.buttons-wrapper {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  grid-gap: 10px;
  grid-auto-rows: minmax(60px, auto);
  padding: 20px 0;
}

.button-full-width {
  display: grid;
  grid-template-columns: repeat(1, 1fr);
  grid-auto-rows: minmax(60px, auto);
  padding: 10px 0 30px 0;
}

.config-button {
  font-size: 16px;
  font-weight: 300;
  letter-spacing: 1px;
  outline: 0;
  border: 1.2px solid rgb(18, 18, 18);
  position: relative;
  background-color: rgba(0, 0, 0, 0);
  user-select: none;
  -webkit-user-select: none;
  touch-action: manipulation;
  color: #242424;
}

.config-button:after {
  content: "";
  background-color: #ffe54c;
  width: 100%;
  z-index: -1;
  position: absolute;
  height: 100%;
  top: 7px;
  left: 7px;
  transition: 0.2s;
}

.config-button:hover:after {
  top: 0px;
  left: 0px;
}

.copy-color {
  font-size: 18px;
  padding: 0px 0 10px 0;
}

.yellow-text {
  color: #ffe54c;
  font-weight: 500;
}

.select-color {
  width: 100%;
  display: flex;
  font-size: 17.5px;
  justify-content: space-between;
}

.select-color span svg {
  fill: #ffe54c;
  width: 18px;
  padding: 0 3px 0 3px;
}

.grid-size-buttons-wrapper {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-gap: 20px;
  grid-auto-rows: minmax(20px, auto);
  padding: 10px 0 40px 0;
}

.small-buttons {
  background-color: transparent;
  border: 1px solid #ffffff;
  border-radius: 5px;
  box-sizing: border-box;
  color: #ffffff;
  display: inline-block;
  font-size: 16px;
  font-weight: 600;
  line-height: 20px;
  margin: 0;
  outline: none;
  padding: 13px 23px;
  position: relative;
  text-align: center;
  text-decoration: none;
  touch-action: manipulation;
  transition: box-shadow .2s,-ms-transform .1s,-webkit-transform .1s,transform .1s;
  user-select: none;
  -webkit-user-select: none;
  width: auto;
}

.small-buttons:active {
  background-color: #ffe54c;
  border-color: #ffe54c;
  color: #242424;
  transform: scale(.96);
}

.download-buttons-wrapper {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-gap: 20px;
  grid-auto-rows: minmax(20px, auto);
  padding: 10px 0 40px 0;
}

.button-clear-canvas {
  font-size: 16px;
  letter-spacing: 2px;
  text-decoration: none;
  text-transform: uppercase;
  color: #f2f2f2;
  background-color: transparent;
  border: 3px solid;
  padding: 0.25em 0.5em;
  box-shadow: 1px 1px 0px 0px, 2px 2px 0px 0px, 3px 3px 0px 0px, 4px 4px 0px 0px, 5px 5px 0px 0px;
  position: relative;
  user-select: none;
  -webkit-user-select: none;
  touch-action: manipulation;
}

.button-clear-canvas:active {
  box-shadow: 0px 0px 0px 0px;
  top: 5px;
  left: 5px;
}

.coming-soon-message {
  display: none;
  max-width: 50%;
  max-height: 50%;
  margin: 0 auto;
  text-align: center;
}

.coming-soon-message img {
  width: 100%;
}

@media (max-width: 1200px) {
  .hide-on-small-screens {
    display: none;
  }

  .coming-soon-message {
    display: unset;
  }
}
</style>

