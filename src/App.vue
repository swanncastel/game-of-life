<template>
  <div id="app">
    <canvas
      ref="canvas"
      :width="width"
      :height="height"
      :class="{
        grabbing: mouseDown && running,
        pointing: !mouseDown && running
      }"
      @mousemove="onMouseMove"
      @mousedown="mouseDown = true"
      @mouseup="mouseDown = false"
      @wheel="onWheel"
      @click="onClicked"
    />

    <div class="buttons">
      <button v-if="running" class="edit" @click="running = false">
        Edit
      </button>
      <button v-else class="edit" @click="running = true">Run</button>
      <label class="edit">
        Import
        <input
          v-show="false"
          type="file"
          accept=".rle"
          name="rle"
          @change="onFile"
        />
      </label>
    </div>
  </div>
</template>

<script>
import { render, nextGeneration } from './gameoflife';
import { parseRle } from './rle-parser';
import template from './template.json';

export default {
  name: 'App',

  data() {
    return {
      width: window.innerWidth,
      height: window.innerHeight,
      grid: new Set(template),
      mouseDown: false,
      running: true,
      offsetX: window.innerWidth / 2,
      offsetY: window.innerHeight / 2,
      zoom: 1,
      cellSize: 5,
      intervalId: null
    };
  },

  watch: {
    grid: 'renderTheGrid',
    offsetX: 'renderTheGrid',
    offsetY: 'renderTheGrid',
    width: 'renderTheGrid',
    running: {
      immediate: true,
      handler() {
        if (this.running) {
          this.intervalId = setInterval(() => {
            this.grid = nextGeneration(this.grid);
          }, 50);
        } else {
          clearInterval(this.intervalId);
        }
      }
    }
  },

  mounted() {
    window.addEventListener('resize', () => {
      this.width = window.innerWidth;
      this.height = window.innerHeight;
    });

    this.renderTheGrid();
  },

  updated() {
    this.renderTheGrid();
  },

  methods: {
    renderTheGrid() {
      const ctx = this.$refs.canvas.getContext('2d');

      ctx.fillStyle = '#ffffff';
      ctx.fillRect(0, 0, ctx.canvas.width, ctx.canvas.height);

      ctx.fillStyle = '#555555';
      render(ctx, this.grid, {
        offsetX: this.offsetX,
        offsetY: this.offsetY,
        zoom: this.zoom,
        cellSize: this.cellSize
      });
    },

    getCoordInTheGame(screenX, screenY) {
      return [
        screenX / this.zoom - this.offsetX,
        screenY / this.zoom - this.offsetY
      ];
    },

    onMouseMove(e) {
      if (this.mouseDown && this.running) {
        this.offsetX += e.movementX / this.zoom;
        this.offsetY += e.movementY / this.zoom;
      }
    },

    onWheel({ clientX, clientY, deltaY }) {
      const [beforeX, beforeY] = this.getCoordInTheGame(clientX, clientY);

      this.zoom = this.zoom * (deltaY > 0 ? 0.9 : 1.1);

      const [afterX, afterY] = this.getCoordInTheGame(clientX, clientY);

      this.offsetX += afterX - beforeX;
      this.offsetY += afterY - beforeY;
    },

    onClicked({ clientX, clientY }) {
      if (!this.running) {
        const coord = this.getCoordInTheGame(clientX, clientY)
          .map(v => v / this.cellSize)
          .map(Math.floor)
          .join(',');

        if (this.grid.has(coord)) {
          this.grid.delete(coord);
        } else {
          this.grid.add(coord);
        }

        this.renderTheGrid();
      }
    },

    async onFile({ target: { files } }) {
      this.running = false;
      const file = await files[0].text();
      this.grid = parseRle(file);
    }
  }
};
</script>

<style>
#app {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  display: flex;
}

canvas {
  cursor: crosshair;
}

.pointing {
  cursor: pointer;
}

.grabbing {
  cursor: grabbing;
}

.buttons {
  position: fixed;
  left: 20px;
  bottom: 20px;
}

.edit {
  display: inline-block;
  outline: none;
  border: none;
  height: 50px;
  line-height: 50px;
  padding: 0 20px;
  color: white;
  background-color: #555555;
  font-size: 16px;
  font-family: Aria;
}
</style>
