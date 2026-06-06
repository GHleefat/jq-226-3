<script setup>
import { ref } from "vue";
import GameCanvas from "./components/GameCanvas.vue";
import ControlPanel from "./components/ControlPanel.vue";
import ScorePanel from "./components/ScorePanel.vue";

const SHIP_TYPES = {
  light: {
    id: "light",
    name: "轻型侦察舰",
    description: "质量轻、推力大、机动灵活，适合新手",
    mass: 0.6,
    thrustPower: 0.12,
    maxSpeed: 10,
    rotationAccel: 0.15,
    maxAngularVelocity: 0.25,
    rotationDamping: 0.92,
    color: "#48bb78",
    difficulty: "简单",
  },
  standard: {
    id: "standard",
    name: "标准运输船",
    description: "各属性均衡的经典型号",
    mass: 1.0,
    thrustPower: 0.08,
    maxSpeed: 8,
    rotationAccel: 0.08,
    maxAngularVelocity: 0.15,
    rotationDamping: 0.94,
    color: "#63b3ed",
    difficulty: "普通",
  },
  heavy: {
    id: "heavy",
    name: "重型货运舰",
    description: "质量大、推力小，惯性极强，挑战操作",
    mass: 1.8,
    thrustPower: 0.05,
    maxSpeed: 6,
    rotationAccel: 0.035,
    maxAngularVelocity: 0.08,
    rotationDamping: 0.97,
    color: "#ed8936",
    difficulty: "困难",
  },
};

const thrust = ref({ x: 0, y: 0 });
const gameState = ref("playing");
const lastResult = ref(null);
const stats = ref({ speed: 0, distance: 0, vx: 0, vy: 0 });
const bestScore = ref(0);
const attempts = ref(0);
const successCount = ref(0);
const selectedShip = ref("standard");
const obstacleMode = ref(false);

const gameCanvasRef = ref(null);

function handleThrustChange(val) {
  thrust.value = val;
}

function handleUpdateStats(val) {
  stats.value = val;
}

function handleDocked(result) {
  gameState.value = "docked";
  lastResult.value = result;
  attempts.value++;
  successCount.value++;
  if (result.score > bestScore.value) {
    bestScore.value = result.score;
  }
}

function handleCrashed(result) {
  gameState.value = "crashed";
  lastResult.value = result;
  attempts.value++;
}

function handleRestart() {
  gameState.value = "playing";
  lastResult.value = null;
}

function handleShipChange(shipId) {
  selectedShip.value = shipId;
  handleRestart();
}

function handleObstacleModeChange(val) {
  obstacleMode.value = val;
  handleRestart();
}
</script>

<template>
  <div class="app-container">
    <ScorePanel
      :gameState="gameState"
      :lastResult="lastResult"
      :bestScore="bestScore"
      :attempts="attempts"
      :successCount="successCount"
      :shipTypes="SHIP_TYPES"
      :selectedShip="selectedShip"
      :obstacleMode="obstacleMode"
      @restart="handleRestart"
      @ship-change="handleShipChange"
      @obstacle-mode-change="handleObstacleModeChange"
    />

    <div class="canvas-wrapper">
      <GameCanvas
        ref="gameCanvasRef"
        :thrust="thrust"
        :gameState="gameState"
        :shipConfig="SHIP_TYPES[selectedShip]"
        :obstacleMode="obstacleMode"
        @docked="handleDocked"
        @crashed="handleCrashed"
        @update-stats="handleUpdateStats"
      />
    </div>

    <ControlPanel
      :stats="stats"
      :shipTypes="SHIP_TYPES"
      :selectedShip="selectedShip"
      :obstacleMode="obstacleMode"
      @thrust-change="handleThrustChange"
      @reset="handleRestart"
      @ship-change="handleShipChange"
      @obstacle-mode-change="handleObstacleModeChange"
    />
  </div>
</template>

<style scoped>
.app-container {
  height: 100vh;
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 20px;
  padding: 16px;
  box-sizing: border-box;
  flex-wrap: nowrap;
  overflow: hidden;
}

.canvas-wrapper {
  flex-shrink: 0;
  display: flex;
  align-items: center;
}

@media (max-width: 1400px) {
  .app-container {
    flex-direction: row;
    gap: 12px;
    padding: 10px;
    overflow-x: auto;
    overflow-y: hidden;
  }
}
</style>
