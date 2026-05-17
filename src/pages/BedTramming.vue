<template>
  <div class="bed-tramming">

    <!-- Controls -->
    <div class="controls">
      <label>
        Grid:
        <select v-model.number="gridSize">
          <option :value="3">3 x 3</option>
          <option :value="5">5 x 5</option>
          <option :value="7">7 x 7</option>
        </select>
      </label>

      <label>
        Bed X (mm):
        <input type="number" v-model.number="bedX" />
      </label>

      <label>
        Bed Y (mm):
        <input type="number" v-model.number="bedY" />
      </label>

      <label>
        Inset (mm):
        <input type="number" v-model.number="inset" />
      </label>

      <label>
        Lift Z (mm):
        <input type="number" v-model.number="liftZ" />
      </label>
    </div>

    <!-- BED WRAPPER -->
    <div class="bed-wrapper">

        <!-- BACKGROUND SVG (LOCKED TO SAME COORD SYSTEM) -->
        <img
            class="bed-bg"
            :src="bedSvg"
            :width="bedX"
            :height="bedY"
        />

        <!-- OVERLAY -->
        <svg
            class="overlay"
            :viewBox="`0 0 ${bedX} ${bedY}`"
            :width="bedX"
            :height="bedY"
        >

        <!-- GRID POINTS -->
        <circle
          v-for="(p, i) in points"
          :key="i"
          :cx="p.x"
          :cy="p.y"
          r="6"
          class="point"
          :fill="i === currentIndex ? '#ffcc00' : '#3498db'"
          @click="selectPoint(i)"
        />

        <!-- TARGET GHOST -->
        <circle
          v-if="targetPoint"
          :cx="targetPoint.x"
          :cy="targetPoint.y"
          r="6"
          fill="#ffcc00"
          opacity="0.35"
        />

        <!-- MOVE LINE -->
        <line
          v-if="moveLine"
          :x1="moveLine.x1"
          :y1="moveLine.y1"
          :x2="moveLine.x2"
          :y2="moveLine.y2"
          stroke="#ffcc00"
          stroke-width="2"
          opacity="0.7"
        />

        <!-- LIVE HEAD -->
        <circle
          :cx="head.x"
          :cy="head.y"
          r="7"
          class="head"
        />

      </svg>

    </div>

    <!-- ACTIONS -->
    <div class="actions">

      <button @click="homeAll">Home All</button>
      <button @click="homeX">Home X</button>
      <button @click="homeY">Home Y</button>
      <button @click="homeZ">Home Z</button>

      <span class="sep"></span>

      <button @click="prevPoint">Previous</button>
      <button @click="goToPoint">Move</button>
      <button @click="nextPoint">Next</button>

      <span class="sep"></span>

      <button @click="finishTramming">Done</button>

    </div>

    <!-- STATUS -->
    <div class="status">
      Point: {{ currentIndex + 1 }} / {{ points.length }} |
      X: {{ headRaw.x.toFixed(1) }} |
      Y: {{ headRaw.y.toFixed(1) }}
    </div>

  </div>
</template>

<script>
import bedSvg from "@/assets/bed.svg";

export default {
  name: "BedTramming",

  data() {
    return {
      printerIP: "192.168.1.33",

      bedSvg,

      bedX: 220,
      bedY: 220,
      gridSize: 3,
      inset: 30,
      liftZ: 5,

      currentIndex: 0,

      headRaw: { x: 0, y: 0 },
      head: { x: 0, y: 0 },

      moveLine: null,
      moveTimeout: null,
      targetPoint: null
    };
  },

  computed: {

    usableX() {
      return this.bedX - this.inset * 2;
    },

    usableY() {
      return this.bedY - this.inset * 2;
    },

    points() {
      const pts = [];
      const stepX = this.usableX / (this.gridSize - 1);
      const stepY = this.usableY / (this.gridSize - 1);

      for (let y = 0; y < this.gridSize; y++) {
        for (let x = 0; x < this.gridSize; x++) {

          const rawX = this.inset + x * stepX;
          const rawY = this.inset + y * stepY;

          const flippedY = this.bedY - rawY;

          pts.push({ x: rawX, y: flippedY });
        }
      }

      return pts;
    },

    currentPoint() {
      return this.points[this.currentIndex] || { x: 0, y: 0 };
    }
  },

  mounted() {
    this.startPolling();
  },

  methods: {

    toSvg(raw) {
      return {
        x: raw.x,
        y: this.bedY - raw.y
      };
    },

    toPrinter(svg) {
      return {
        x: svg.x,
        y: this.bedY - svg.y
      };
    },

    startPolling() {
      setInterval(async () => {
        try {
          const res = await fetch(
            `http://${this.printerIP}/printer/objects/query?toolhead`
          );

          const data = await res.json();
          const pos = data?.result?.status?.toolhead?.position;

          if (pos) {
            this.headRaw = { x: pos[0], y: pos[1] };
            this.head = this.toSvg(this.headRaw);
          }

        } catch (e) {}
      }, 300);
    },

    async sendGcode(cmd) {
      await fetch(`http://${this.printerIP}/printer/gcode/script`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ script: cmd })
      });
    },

    homeAll() { this.sendGcode("G28"); },
    homeX() { this.sendGcode("G28 X"); },
    homeY() { this.sendGcode("G28 Y"); },
    homeZ() { this.sendGcode("G28 Z"); },

    async liftUp() {
      await this.sendGcode(`G91\nG1 Z${this.liftZ} F600\nG90`);
    },

    async goToPoint() {
      const target = this.currentPoint;
      const start = { ...this.head };

      this.moveLine = {
        x1: start.x,
        y1: start.y,
        x2: target.x,
        y2: target.y
      };

      this.targetPoint = target;

      if (this.moveTimeout) clearTimeout(this.moveTimeout);

      await this.liftUp();

      const printerTarget = this.toPrinter(target);

      await this.sendGcode(
        `G1 X${printerTarget.x.toFixed(2)} Y${printerTarget.y.toFixed(2)} F6000`
      );

      await this.sendGcode("G1 Z0 F600");

      this.moveTimeout = setTimeout(() => {
        this.moveLine = null;
        this.targetPoint = null;
      }, 500);
    },

    selectPoint(i) {
      this.currentIndex = i;
      this.goToPoint();
    },

    nextPoint() {
      if (this.currentIndex < this.points.length - 1) {
        this.currentIndex++;
        this.goToPoint();
      }
    },

    prevPoint() {
      if (this.currentIndex > 0) {
        this.currentIndex--;
        this.goToPoint();
      }
    },

    async finishTramming() {
        // 1. Lift Z first (safe clearance)
        await this.sendGcode("G91\nG1 Z10 F600\nG90");

        // 2. Home X
        await this.sendGcode("G28 X");

        // 3. Home Y
        await this.sendGcode("G28 Y");

        // 4. Final home Z (safe after XY are known)
        await this.sendGcode("G28 Z");
        }
  }
};
</script>

<style scoped>
.bed-tramming {
  display: flex;
  flex-direction: column;
  gap: 12px;
  color: #ddd;
  font-family: system-ui, sans-serif;
}

.controls {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  padding: 10px;
  background: #1a1a1a;
  border: 1px solid #2c2c2c;
  border-radius: 8px;
}

.controls label {
  display: flex;
  flex-direction: column;
  font-size: 12px;
  color: #aaa;
}

.bed-wrapper {
  position: relative;
  max-width: 500px;
  stroke: rgb(56, 56, 56);
  stroke-width: 2px;
  vector-effect: non-scaling-stroke;
  
}

.bed-bg {
  position: absolute;
  width: 100%;
  height: 100%;
  stroke: rgb(56, 56, 56);
  stroke-width: 2px;
  object-fit: contain;
  z-index: 0;
  pointer-events: none;
}

.overlay {
  position: relative;
  width: 100%;
  height: 100%;
  z-index: 1;
}

.point {
  cursor: pointer;
}

.head {
  fill: #01004b;
  stroke: rgb(0, 110, 255);
  stroke-width: 1.5;
  filter: drop-shadow(0 0 9px #008cff);
}

.actions {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  padding: 10px;
  background: #1a1a1a;
  border-radius: 8px;
  border: 1px solid #2c2c2c;
}

button {
  background: #144fbd96;
  color: #ffffff;
  border: 2px solid #06c5ff2c;
  padding: 8px 12px;
  border-radius: 6px;
  cursor: pointer;
}

button:hover {
  background: #1a72ad;
}

.status {
  font-size: 20px;
  color: #ffffff;
}
</style>