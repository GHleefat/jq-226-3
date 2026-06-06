<script setup>
import { ref, onMounted, onUnmounted, computed } from "vue";

const props = defineProps({
  stats: {
    type: Object,
    default: () => ({ speed: 0, distance: 0, vx: 0, vy: 0 }),
  },
  shipTypes: {
    type: Object,
    default: () => ({}),
  },
  selectedShip: {
    type: String,
    default: "standard",
  },
  obstacleMode: {
    type: Boolean,
    default: false,
  },
});

const emit = defineEmits([
  "thrust-change",
  "reset",
  "ship-change",
  "obstacle-mode-change",
]);

const keys = ref({
  up: false,
  down: false,
  left: false,
  right: false,
});

const joystick = ref({
  active: false,
  baseX: 0,
  baseY: 0,
  knobX: 0,
  knobY: 0,
  maxRadius: 60,
});

const thrust = computed(() => {
  let x = 0;
  let y = 0;

  if (keys.value.left) x -= 1;
  if (keys.value.right) x += 1;
  if (keys.value.up) y -= 1;
  if (keys.value.down) y += 1;

  if (joystick.value.active) {
    const dx = joystick.value.knobX - joystick.value.baseX;
    const dy = joystick.value.knobY - joystick.value.baseY;
    const dist = Math.sqrt(dx * dx + dy * dy);
    if (dist > 0) {
      const normalizedDist = Math.min(dist / joystick.value.maxRadius, 1);
      x = (dx / dist) * normalizedDist;
      y = (dy / dist) * normalizedDist;
    }
  }

  const mag = Math.sqrt(x * x + y * y);
  if (mag > 1) {
    x /= mag;
    y /= mag;
  }

  return { x, y };
});

const GAME_KEYS = [
  "arrowup",
  "arrowdown",
  "arrowleft",
  "arrowright",
  "w",
  "a",
  "s",
  "d",
];

function handleKeyDown(e) {
  const key = e.key.toLowerCase();
  if (GAME_KEYS.includes(key)) {
    e.preventDefault();
  }
  if (key === "arrowup" || key === "w") keys.value.up = true;
  if (key === "arrowdown" || key === "s") keys.value.down = true;
  if (key === "arrowleft" || key === "a") keys.value.left = true;
  if (key === "arrowright" || key === "d") keys.value.right = true;
}

function handleKeyUp(e) {
  const key = e.key.toLowerCase();
  if (GAME_KEYS.includes(key)) {
    e.preventDefault();
  }
  if (key === "arrowup" || key === "w") keys.value.up = false;
  if (key === "arrowdown" || key === "s") keys.value.down = false;
  if (key === "arrowleft" || key === "a") keys.value.left = false;
  if (key === "arrowright" || key === "d") keys.value.right = false;
}

const joystickRef = ref(null);

function getRelativePos(e) {
  const rect = joystickRef.value.getBoundingClientRect();
  const clientX = e.touches ? e.touches[0].clientX : e.clientX;
  const clientY = e.touches ? e.touches[0].clientY : e.clientY;
  return {
    x: clientX - rect.left,
    y: clientY - rect.top,
  };
}

function startJoystick(e) {
  e.preventDefault();
  const pos = getRelativePos(e);
  joystick.value.active = true;
  joystick.value.baseX = pos.x;
  joystick.value.baseY = pos.y;
  joystick.value.knobX = pos.x;
  joystick.value.knobY = pos.y;
}

function moveJoystick(e) {
  if (!joystick.value.active) return;
  e.preventDefault();
  const pos = getRelativePos(e);
  const dx = pos.x - joystick.value.baseX;
  const dy = pos.y - joystick.value.baseY;
  const dist = Math.sqrt(dx * dx + dy * dy);
  if (dist > joystick.value.maxRadius) {
    const ratio = joystick.value.maxRadius / dist;
    joystick.value.knobX = joystick.value.baseX + dx * ratio;
    joystick.value.knobY = joystick.value.baseY + dy * ratio;
  } else {
    joystick.value.knobX = pos.x;
    joystick.value.knobY = pos.y;
  }
}

function endJoystick(e) {
  if (e) e.preventDefault();
  joystick.value.active = false;
  joystick.value.knobX = joystick.value.baseX;
  joystick.value.knobY = joystick.value.baseY;
}

import { watch } from "vue";
watch(
  thrust,
  (val) => {
    emit("thrust-change", val);
  },
  { deep: true },
);

onMounted(() => {
  window.addEventListener("keydown", handleKeyDown);
  window.addEventListener("keyup", handleKeyUp);
  window.addEventListener("mousemove", moveJoystick);
  window.addEventListener("mouseup", endJoystick);
  window.addEventListener("touchmove", moveJoystick, { passive: false });
  window.addEventListener("touchend", endJoystick);
});

onUnmounted(() => {
  window.removeEventListener("keydown", handleKeyDown);
  window.removeEventListener("keyup", handleKeyUp);
  window.removeEventListener("mousemove", moveJoystick);
  window.removeEventListener("mouseup", endJoystick);
  window.removeEventListener("touchmove", moveJoystick);
  window.removeEventListener("touchend", endJoystick);
});
</script>

<template>
  <div class="control-panel">
    <div class="panel-section">
      <h3 class="section-title">🚀 选择飞船</h3>
      <div class="ship-selector">
        <button
          v-for="ship in shipTypes"
          :key="ship.id"
          class="ship-btn"
          :class="{ active: selectedShip === ship.id }"
          :style="{ '--ship-color': ship.color }"
          @click="emit('ship-change', ship.id)"
        >
          <div class="ship-name">{{ ship.name }}</div>
          <div class="ship-diff" :class="'diff-' + ship.difficulty">
            {{ ship.difficulty }}
          </div>
          <div class="ship-desc">{{ ship.description }}</div>
          <div class="ship-stats">
            <div class="stat-mini">
              <span class="mini-label">质量</span>
              <span class="mini-bar"
                ><span :style="{ width: (ship.mass / 2) * 100 + '%' }"></span
              ></span>
            </div>
            <div class="stat-mini">
              <span class="mini-label">推力</span>
              <span class="mini-bar"
                ><span
                  :style="{ width: (ship.thrustPower / 0.12) * 100 + '%' }"
                ></span
              ></span>
            </div>
          </div>
        </button>
      </div>
    </div>

    <div class="panel-section">
      <h3 class="section-title">☄️ 游戏模式</h3>
      <label class="toggle-wrapper">
        <span class="toggle-label">障碍物模式</span>
        <div
          class="toggle-switch"
          :class="{ active: obstacleMode }"
          @click="emit('obstacle-mode-change', !obstacleMode)"
        >
          <div class="toggle-knob"></div>
        </div>
      </label>
      <p class="mode-hint">
        {{ obstacleMode ? "小心漂浮的太空垃圾！" : "安全模式，无障碍物" }}
      </p>
    </div>

    <div class="panel-section">
      <h3 class="section-title">飞行状态</h3>
      <div class="stats-grid">
        <div class="stat-item">
          <span class="stat-label">速度</span>
          <span
            class="stat-value"
            :class="{
              'stat-safe': stats.speed < 0.5,
              'stat-warn': stats.speed >= 0.5 && stats.speed < 3,
              'stat-danger': stats.speed >= 3,
            }"
            >{{ stats.speed?.toFixed(2) }} u/s</span
          >
        </div>
        <div class="stat-item">
          <span class="stat-label">距离</span>
          <span class="stat-value">{{ stats.distance?.toFixed(1) }} px</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Vx</span>
          <span class="stat-value">{{ stats.vx?.toFixed(2) }}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Vy</span>
          <span class="stat-value">{{ stats.vy?.toFixed(2) }}</span>
        </div>
      </div>
    </div>

    <div class="panel-section">
      <h3 class="section-title">虚拟摇杆</h3>
      <div
        ref="joystickRef"
        class="joystick-container"
        @mousedown="startJoystick"
        @touchstart="startJoystick"
      >
        <div class="joystick-base">
          <div class="joystick-axis-x"></div>
          <div class="joystick-axis-y"></div>
          <div
            class="joystick-knob"
            :style="{
              left: (joystick.active ? joystick.knobX : 80) + 'px',
              top: (joystick.active ? joystick.knobY : 80) + 'px',
            }"
          ></div>
        </div>
      </div>
      <p class="hint">拖拽摇杆或使用键盘 ↑↓←→ / WASD</p>
    </div>

    <div class="panel-section">
      <h3 class="section-title">方向键控制</h3>
      <div class="direction-pad">
        <button
          class="dir-btn dir-up"
          :class="{ active: keys.up }"
          @mousedown="keys.up = true"
          @mouseup="keys.up = false"
          @mouseleave="keys.up = false"
          @touchstart.prevent="keys.up = true"
          @touchend.prevent="keys.up = false"
        >
          ▲
        </button>
        <div class="dir-row">
          <button
            class="dir-btn dir-left"
            :class="{ active: keys.left }"
            @mousedown="keys.left = true"
            @mouseup="keys.left = false"
            @mouseleave="keys.left = false"
            @touchstart.prevent="keys.left = true"
            @touchend.prevent="keys.left = false"
          >
            ◀
          </button>
          <div class="dir-center"></div>
          <button
            class="dir-btn dir-right"
            :class="{ active: keys.right }"
            @mousedown="keys.right = true"
            @mouseup="keys.right = false"
            @mouseleave="keys.right = false"
            @touchstart.prevent="keys.right = true"
            @touchend.prevent="keys.right = false"
          >
            ▶
          </button>
        </div>
        <button
          class="dir-btn dir-down"
          :class="{ active: keys.down }"
          @mousedown="keys.down = true"
          @mouseup="keys.down = false"
          @mouseleave="keys.down = false"
          @touchstart.prevent="keys.down = true"
          @touchend.prevent="keys.down = false"
        >
          ▼
        </button>
      </div>
    </div>

    <button class="reset-btn" @click="$emit('reset')">🔄 重新开始</button>
  </div>
</template>

<style scoped>
.control-panel {
  width: 260px;
  display: flex;
  flex-direction: column;
  gap: 20px;
  padding: 20px;
  background: linear-gradient(145deg, #1a1a2e, #16213e);
  border: 1px solid #2d3748;
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.4);
}

.ship-selector {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.ship-btn {
  text-align: left;
  padding: 10px 12px;
  border: 2px solid #2d3748;
  border-radius: 10px;
  background: rgba(45, 55, 72, 0.3);
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.ship-btn:hover {
  border-color: var(--ship-color);
  background: rgba(45, 55, 72, 0.6);
}

.ship-btn.active {
  border-color: var(--ship-color);
  background: rgba(45, 55, 72, 0.7);
  box-shadow: 0 0 12px color-mix(in srgb, var(--ship-color) 40%, transparent);
}

.ship-name {
  font-size: 13px;
  font-weight: 600;
  color: var(--ship-color);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.ship-diff {
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 4px;
  font-weight: 600;
}

.diff-简单 {
  background: rgba(72, 187, 120, 0.2);
  color: #48bb78;
}

.diff-普通 {
  background: rgba(99, 179, 237, 0.2);
  color: #63b3ed;
}

.diff-困难 {
  background: rgba(237, 137, 54, 0.2);
  color: #ed8936;
}

.ship-desc {
  font-size: 11px;
  color: #718096;
  line-height: 1.4;
}

.ship-stats {
  display: flex;
  flex-direction: column;
  gap: 3px;
  margin-top: 2px;
}

.stat-mini {
  display: flex;
  align-items: center;
  gap: 6px;
}

.mini-label {
  font-size: 10px;
  color: #718096;
  width: 24px;
  text-transform: uppercase;
}

.mini-bar {
  flex: 1;
  height: 4px;
  background: rgba(45, 55, 72, 0.8);
  border-radius: 2px;
  overflow: hidden;
}

.mini-bar span {
  display: block;
  height: 100%;
  background: var(--ship-color);
  border-radius: 2px;
  transition: width 0.3s;
}

.toggle-wrapper {
  display: flex;
  align-items: center;
  justify-content: space-between;
  cursor: pointer;
  user-select: none;
}

.toggle-label {
  font-size: 13px;
  color: #e2e8f0;
  font-weight: 500;
}

.toggle-switch {
  width: 44px;
  height: 24px;
  border-radius: 12px;
  background: #2d3748;
  position: relative;
  transition: background 0.2s;
  border: 2px solid #4a5568;
}

.toggle-switch.active {
  background: #3182ce;
  border-color: #63b3ed;
}

.toggle-knob {
  position: absolute;
  top: 1px;
  left: 1px;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #e2e8f0;
  transition: transform 0.2s;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.toggle-switch.active .toggle-knob {
  transform: translateX(20px);
  background: #fff;
}

.mode-hint {
  margin: 6px 0 0;
  font-size: 11px;
  color: #718096;
  font-style: italic;
}

.panel-section {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.section-title {
  margin: 0;
  font-size: 14px;
  font-weight: 600;
  color: #63b3ed;
  text-transform: uppercase;
  letter-spacing: 1px;
  border-bottom: 1px solid #2d3748;
  padding-bottom: 8px;
}

.stats-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.stat-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 8px 10px;
  background: rgba(45, 55, 72, 0.5);
  border-radius: 8px;
}

.stat-label {
  font-size: 11px;
  color: #718096;
  text-transform: uppercase;
}

.stat-value {
  font-size: 16px;
  font-weight: 600;
  font-family: "Courier New", monospace;
  color: #e2e8f0;
}

.stat-safe {
  color: #48bb78;
}

.stat-warn {
  color: #ecc94b;
}

.stat-danger {
  color: #f56565;
}

.joystick-container {
  display: flex;
  justify-content: center;
  user-select: none;
  touch-action: none;
}

.joystick-base {
  position: relative;
  width: 160px;
  height: 160px;
  border-radius: 50%;
  background: radial-gradient(circle, #2d3748 0%, #1a202c 100%);
  border: 2px solid #4a5568;
  box-shadow: inset 0 0 20px rgba(0, 0, 0, 0.5);
}

.joystick-axis-x,
.joystick-axis-y {
  position: absolute;
  background: rgba(99, 179, 237, 0.3);
}

.joystick-axis-x {
  top: 50%;
  left: 10%;
  right: 10%;
  height: 1px;
  transform: translateY(-50%);
}

.joystick-axis-y {
  left: 50%;
  top: 10%;
  bottom: 10%;
  width: 1px;
  transform: translateX(-50%);
}

.joystick-knob {
  position: absolute;
  width: 50px;
  height: 50px;
  border-radius: 50%;
  background: radial-gradient(circle at 30% 30%, #63b3ed, #3182ce);
  border: 2px solid #90cdf4;
  transform: translate(-50%, -50%);
  box-shadow: 0 4px 12px rgba(99, 179, 237, 0.4);
  cursor: grab;
  transition: box-shadow 0.2s;
}

.joystick-knob:hover {
  box-shadow: 0 6px 20px rgba(99, 179, 237, 0.6);
}

.hint {
  margin: 0;
  font-size: 11px;
  color: #718096;
  text-align: center;
}

.direction-pad {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

.dir-row {
  display: flex;
  gap: 4px;
}

.dir-btn {
  width: 50px;
  height: 50px;
  border: 2px solid #4a5568;
  border-radius: 10px;
  background: linear-gradient(145deg, #2d3748, #1a202c);
  color: #a0aec0;
  font-size: 18px;
  cursor: pointer;
  transition: all 0.1s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.dir-btn:hover {
  border-color: #63b3ed;
  color: #e2e8f0;
}

.dir-btn.active {
  background: linear-gradient(145deg, #3182ce, #2c5282);
  border-color: #63b3ed;
  color: #fff;
  box-shadow: 0 0 15px rgba(99, 179, 237, 0.5);
  transform: scale(0.95);
}

.dir-center {
  width: 50px;
  height: 50px;
  border-radius: 10px;
  background: rgba(45, 55, 72, 0.3);
}

.reset-btn {
  padding: 14px 20px;
  border: 2px solid #c05621;
  border-radius: 10px;
  background: linear-gradient(145deg, #dd6b20, #c05621);
  color: #fff;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.reset-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(221, 107, 32, 0.4);
}

.reset-btn:active {
  transform: translateY(0);
}
</style>
