<script setup>
import { ref, onMounted, onUnmounted, watch } from "vue";

const props = defineProps({
  thrust: {
    type: Object,
    default: () => ({ x: 0, y: 0 }),
  },
  gameState: {
    type: String,
    default: "playing",
  },
  shipConfig: {
    type: Object,
    default: () => ({
      mass: 1.0,
      thrustPower: 0.08,
      maxSpeed: 8,
      color: "#63b3ed",
    }),
  },
  obstacleMode: {
    type: Boolean,
    default: false,
  },
});

const emit = defineEmits(["docked", "crashed", "update-stats", "reset"]);

const canvas = ref(null);
let ctx = null;
let animationId = null;
let lastTime = 0;

const CANVAS_WIDTH = 800;
const CANVAS_HEIGHT = 600;

const DOCKING_DISTANCE = 30;
const DOCKING_SPEED = 0.5;
const CRASH_SPEED = 3.0;

const DAMPING_BASE = 0.999;

const ship = ref({
  x: 0,
  y: 0,
  vx: 0,
  vy: 0,
  angle: 0,
  angularVelocity: 0,
  width: 30,
  height: 40,
});

const station = ref({
  x: 0,
  y: 0,
  width: 120,
  height: 80,
  dockingPort: { x: 0, y: 0, width: 40, height: 10 },
});

const stars = ref([]);
const thrustParticles = ref([]);
const obstacles = ref([]);

function initStars() {
  stars.value = [];
  for (let i = 0; i < 150; i++) {
    stars.value.push({
      x: Math.random() * CANVAS_WIDTH,
      y: Math.random() * CANVAS_HEIGHT,
      size: Math.random() * 2 + 0.5,
      brightness: Math.random() * 0.5 + 0.5,
    });
  }
}

function initObstacles() {
  obstacles.value = [];
  if (!props.obstacleMode) return;

  const count = 5 + Math.floor(Math.random() * 4);
  for (let i = 0; i < count; i++) {
    let x,
      y,
      valid = false,
      attempts = 0;
    while (!valid && attempts < 50) {
      x = 50 + Math.random() * (CANVAS_WIDTH - 100);
      y = 50 + Math.random() * (CANVAS_HEIGHT - 100);
      const dxToStation = x - station.value.x;
      const dyToStation = y - station.value.y;
      const distToStation = Math.sqrt(
        dxToStation * dxToStation + dyToStation * dyToStation,
      );
      const dxToShip = x - ship.value.x;
      const dyToShip = y - ship.value.y;
      const distToShip = Math.sqrt(dxToShip * dxToShip + dyToShip * dyToShip);
      valid = distToStation > 120 && distToShip > 80;
      attempts++;
    }
    if (valid) {
      const size = 12 + Math.random() * 18;
      const points = [];
      const verts = 6 + Math.floor(Math.random() * 4);
      for (let j = 0; j < verts; j++) {
        const angle = (j / verts) * Math.PI * 2;
        const r = size * (0.7 + Math.random() * 0.6);
        points.push({
          x: Math.cos(angle) * r,
          y: Math.sin(angle) * r,
        });
      }
      obstacles.value.push({
        x,
        y,
        vx: (Math.random() - 0.5) * 0.8,
        vy: (Math.random() - 0.5) * 0.8,
        rotation: Math.random() * Math.PI * 2,
        rotationSpeed: (Math.random() - 0.5) * 0.02,
        size,
        points,
      });
    }
  }
}

function initGame() {
  station.value.x = CANVAS_WIDTH / 2;
  station.value.y = CANVAS_HEIGHT / 2 - 50;
  station.value.dockingPort.x = station.value.x;
  station.value.dockingPort.y = station.value.y + station.value.height / 2;

  const angle = Math.random() * Math.PI * 2;
  const distance = 180 + Math.random() * 100;
  ship.value.x = station.value.x + Math.cos(angle) * distance;
  ship.value.y = station.value.y + Math.sin(angle) * distance;
  ship.value.vx = (Math.random() - 0.5) * 1;
  ship.value.vy = (Math.random() - 0.5) * 1;
  ship.value.angle =
    Math.atan2(
      station.value.dockingPort.y - ship.value.y,
      station.value.dockingPort.x - ship.value.x,
    ) +
    Math.PI / 2;
  ship.value.angularVelocity = 0;
  thrustParticles.value = [];

  initObstacles();
}

function updateObstacles() {
  obstacles.value.forEach((o) => {
    o.x += o.vx;
    o.y += o.vy;
    o.rotation += o.rotationSpeed;

    if (o.x < o.size) {
      o.x = o.size;
      o.vx *= -1;
    }
    if (o.x > CANVAS_WIDTH - o.size) {
      o.x = CANVAS_WIDTH - o.size;
      o.vx *= -1;
    }
    if (o.y < o.size) {
      o.y = o.size;
      o.vy *= -1;
    }
    if (o.y > CANVAS_HEIGHT - o.size) {
      o.y = CANVAS_HEIGHT - o.size;
      o.vy *= -1;
    }
  });
}

function checkObstacleCollision() {
  for (const o of obstacles.value) {
    const dx = ship.value.x - o.x;
    const dy = ship.value.y - o.y;
    const dist = Math.sqrt(dx * dx + dy * dy);
    if (dist < o.size + 15) {
      const speed = Math.sqrt(ship.value.vx ** 2 + ship.value.vy ** 2);
      emit("crashed", { speed, reason: "obstacle" });
      return true;
    }
  }
  return false;
}

function normalizeAngle(angle) {
  while (angle > Math.PI) angle -= Math.PI * 2;
  while (angle < -Math.PI) angle += Math.PI * 2;
  return angle;
}

function updatePhysics(dt) {
  if (props.gameState !== "playing") return;

  const mass = props.shipConfig.mass || 1.0;
  const thrustPower = props.shipConfig.thrustPower || 0.08;
  const maxSpeed = props.shipConfig.maxSpeed || 8;
  const damping = Math.pow(DAMPING_BASE, 1 / mass);
  const rotationAccel = props.shipConfig.rotationAccel || 0.08;
  const maxAngVel = props.shipConfig.maxAngularVelocity || 0.15;
  const rotationDamping = props.shipConfig.rotationDamping || 0.94;

  ship.value.vx += (props.thrust.x * thrustPower) / mass;
  ship.value.vy += (props.thrust.y * thrustPower) / mass;

  ship.value.vx *= damping;
  ship.value.vy *= damping;

  const speed = Math.sqrt(ship.value.vx ** 2 + ship.value.vy ** 2);
  if (speed > maxSpeed) {
    ship.value.vx = (ship.value.vx / speed) * maxSpeed;
    ship.value.vy = (ship.value.vy / speed) * maxSpeed;
  }

  ship.value.x += ship.value.vx;
  ship.value.y += ship.value.vy;

  const hasThrust =
    Math.abs(props.thrust.x) > 0.1 || Math.abs(props.thrust.y) > 0.1;
  if (hasThrust) {
    const targetAngle =
      Math.atan2(props.thrust.y, props.thrust.x) + Math.PI / 2;
    let angleDiff = normalizeAngle(targetAngle - ship.value.angle);
    ship.value.angularVelocity += angleDiff * rotationAccel;
    spawnThrustParticle();
  }

  if (Math.abs(ship.value.angularVelocity) > maxAngVel) {
    ship.value.angularVelocity =
      Math.sign(ship.value.angularVelocity) * maxAngVel;
  }
  ship.value.angularVelocity *= rotationDamping;
  ship.value.angle += ship.value.angularVelocity;

  if (ship.value.x < -50) ship.value.x = CANVAS_WIDTH + 50;
  if (ship.value.x > CANVAS_WIDTH + 50) ship.value.x = -50;
  if (ship.value.y < -50) ship.value.y = CANVAS_HEIGHT + 50;
  if (ship.value.y > CANVAS_HEIGHT + 50) ship.value.y = -50;

  if (props.obstacleMode) {
    updateObstacles();
    if (checkObstacleCollision()) return;
  }

  checkDocking();
  updateThrustParticles();

  emit("update-stats", {
    x: ship.value.x,
    y: ship.value.y,
    vx: ship.value.vx,
    vy: ship.value.vy,
    speed: speed,
    distance: getDistanceToPort(),
  });
}

function getDistanceToPort() {
  const dx = ship.value.x - station.value.dockingPort.x;
  const dy = ship.value.y - station.value.dockingPort.y;
  return Math.sqrt(dx * dx + dy * dy);
}

function checkDocking() {
  const dx = ship.value.x - station.value.dockingPort.x;
  const dy = ship.value.y - station.value.dockingPort.y;
  const distance = Math.sqrt(dx * dx + dy * dy);
  const speed = Math.sqrt(ship.value.vx ** 2 + ship.value.vy ** 2);

  if (distance < DOCKING_DISTANCE) {
    if (speed < DOCKING_SPEED) {
      const score = calculateScore(distance, speed);
      emit("docked", { score, distance, speed });
    } else if (speed > CRASH_SPEED) {
      emit("crashed", { speed });
    }
  }
}

function calculateScore(distance, speed) {
  const distanceScore = Math.max(0, 100 - distance * 2);
  const speedScore = Math.max(0, 100 - speed * 20);
  return Math.round((distanceScore + speedScore) / 2);
}

function spawnThrustParticle() {
  const backAngle = ship.value.angle + Math.PI;
  thrustParticles.value.push({
    x: ship.value.x + Math.cos(backAngle - Math.PI / 2) * 15,
    y: ship.value.y + Math.sin(backAngle - Math.PI / 2) * 15,
    vx:
      Math.cos(backAngle - Math.PI / 2) * (1 + Math.random() * 2) +
      (Math.random() - 0.5),
    vy:
      Math.sin(backAngle - Math.PI / 2) * (1 + Math.random() * 2) +
      (Math.random() - 0.5),
    life: 20 + Math.random() * 10,
    maxLife: 30,
    size: 2 + Math.random() * 3,
  });
}

function updateThrustParticles() {
  thrustParticles.value = thrustParticles.value.filter((p) => {
    p.x += p.vx;
    p.y += p.vy;
    p.life--;
    return p.life > 0;
  });
}

function shadeColor(color, percent) {
  const num = parseInt(color.replace("#", ""), 16);
  const amt = Math.round(2.55 * percent);
  const R = (num >> 16) + amt;
  const G = ((num >> 8) & 0x00ff) + amt;
  const B = (num & 0x0000ff) + amt;
  return (
    "#" +
    (
      0x1000000 +
      (R < 255 ? (R < 1 ? 0 : R) : 255) * 0x10000 +
      (G < 255 ? (G < 1 ? 0 : G) : 255) * 0x100 +
      (B < 255 ? (B < 1 ? 0 : B) : 255)
    )
      .toString(16)
      .slice(1)
  );
}

function drawObstacles() {
  obstacles.value.forEach((o) => {
    ctx.save();
    ctx.translate(o.x, o.y);
    ctx.rotate(o.rotation);

    const gradient = ctx.createRadialGradient(0, 0, 0, 0, 0, o.size);
    gradient.addColorStop(0, "#718096");
    gradient.addColorStop(0.7, "#4a5568");
    gradient.addColorStop(1, "#2d3748");

    ctx.fillStyle = gradient;
    ctx.strokeStyle = "#1a202c";
    ctx.lineWidth = 2;

    ctx.beginPath();
    o.points.forEach((p, i) => {
      if (i === 0) ctx.moveTo(p.x, p.y);
      else ctx.lineTo(p.x, p.y);
    });
    ctx.closePath();
    ctx.fill();
    ctx.stroke();

    ctx.fillStyle = "rgba(26, 32, 44, 0.5)";
    for (let i = 0; i < 3; i++) {
      const cx = (Math.random() - 0.5) * o.size;
      const cy = (Math.random() - 0.5) * o.size;
      const cr = 2 + Math.random() * 3;
      ctx.beginPath();
      ctx.arc(cx, cy, cr, 0, Math.PI * 2);
      ctx.fill();
    }

    ctx.restore();
  });
}

function drawStars() {
  stars.value.forEach((star) => {
    ctx.fillStyle = `rgba(255, 255, 255, ${star.brightness})`;
    ctx.beginPath();
    ctx.arc(star.x, star.y, star.size, 0, Math.PI * 2);
    ctx.fill();
  });
}

function drawStation() {
  const s = station.value;

  ctx.save();
  ctx.translate(s.x, s.y);

  const gradient = ctx.createLinearGradient(-s.width / 2, 0, s.width / 2, 0);
  gradient.addColorStop(0, "#4a5568");
  gradient.addColorStop(0.5, "#718096");
  gradient.addColorStop(1, "#4a5568");

  ctx.fillStyle = gradient;
  ctx.strokeStyle = "#2d3748";
  ctx.lineWidth = 2;

  ctx.beginPath();
  ctx.roundRect(-s.width / 2, -s.height / 2, s.width, s.height, 8);
  ctx.fill();
  ctx.stroke();

  ctx.fillStyle = "#63b3ed";
  ctx.strokeStyle = "#2b6cb0";
  ctx.lineWidth = 2;
  for (let i = -1; i <= 1; i += 2) {
    ctx.beginPath();
    ctx.roundRect(i * (s.width / 2 + 30), -15, 30, 30, 4);
    ctx.fill();
    ctx.stroke();
    ctx.strokeStyle = "#2b6cb0";
    ctx.beginPath();
    ctx.moveTo(i * (s.width / 2 + 15), -15);
    ctx.lineTo(i * (s.width / 2 + 15), 15);
    ctx.moveTo(i * (s.width / 2 + 30), 0);
    ctx.lineTo(i * (s.width / 2 + 60), 0);
    ctx.stroke();
  }

  ctx.fillStyle = "#1a202c";
  for (let i = 0; i < 4; i++) {
    for (let j = 0; j < 2; j++) {
      ctx.beginPath();
      ctx.roundRect(
        -s.width / 2 + 12 + i * 25,
        -s.height / 2 + 12 + j * 28,
        15,
        20,
        2,
      );
      ctx.fill();
    }
  }

  const portGlow =
    props.gameState === "playing" ? 0.5 + Math.sin(Date.now() / 200) * 0.3 : 0;
  ctx.fillStyle = `rgba(72, 187, 120, ${0.6 + portGlow})`;
  ctx.shadowColor = "#48bb78";
  ctx.shadowBlur = 15;
  ctx.beginPath();
  ctx.roundRect(
    -s.dockingPort.width / 2,
    s.height / 2 - 5,
    s.dockingPort.width,
    s.dockingPort.height,
    3,
  );
  ctx.fill();
  ctx.shadowBlur = 0;

  ctx.fillStyle = "#fff";
  ctx.font = "bold 10px monospace";
  ctx.textAlign = "center";
  ctx.fillText("ISS", 0, 5);

  ctx.restore();
}

function drawShip() {
  const sh = ship.value;

  ctx.save();
  ctx.translate(sh.x, sh.y);
  ctx.rotate(sh.angle);

  thrustParticles.value.forEach((p) => {
    const alpha = p.life / p.maxLife;
    ctx.fillStyle = `rgba(255, ${150 + Math.random() * 50}, 50, ${alpha})`;
    ctx.beginPath();
    const localX = p.x - sh.x;
    const localY = p.y - sh.y;
    const cos = Math.cos(-sh.angle);
    const sin = Math.sin(-sh.angle);
    const rx = localX * cos - localY * sin;
    const ry = localX * sin + localY * cos;
    ctx.arc(rx, ry, p.size * alpha, 0, Math.PI * 2);
    ctx.fill();
  });

  const bodyGradient = ctx.createLinearGradient(
    -sh.width / 2,
    0,
    sh.width / 2,
    0,
  );
  bodyGradient.addColorStop(0, "#e2e8f0");
  bodyGradient.addColorStop(0.5, "#f7fafc");
  bodyGradient.addColorStop(1, "#e2e8f0");

  ctx.fillStyle = bodyGradient;
  ctx.strokeStyle = "#718096";
  ctx.lineWidth = 2;

  ctx.beginPath();
  ctx.moveTo(0, -sh.height / 2);
  ctx.lineTo(sh.width / 2, sh.height / 3);
  ctx.lineTo(sh.width / 3, sh.height / 2);
  ctx.lineTo(-sh.width / 3, sh.height / 2);
  ctx.lineTo(-sh.width / 2, sh.height / 3);
  ctx.closePath();
  ctx.fill();
  ctx.stroke();

  const shipColor = props.shipConfig.color || "#63b3ed";
  const darkerShipColor = shadeColor(shipColor, -30);

  ctx.fillStyle = shipColor;
  ctx.strokeStyle = darkerShipColor;
  ctx.lineWidth = 1.5;
  ctx.beginPath();
  ctx.ellipse(
    0,
    -sh.height / 6,
    sh.width / 4,
    sh.height / 5,
    0,
    0,
    Math.PI * 2,
  );
  ctx.fill();
  ctx.stroke();

  ctx.fillStyle = "#ed8936";
  ctx.strokeStyle = "#c05621";
  ctx.beginPath();
  ctx.roundRect(-sh.width / 2 - 5, sh.height / 4, 8, 12, 2);
  ctx.fill();
  ctx.stroke();
  ctx.beginPath();
  ctx.roundRect(sh.width / 2 - 3, sh.height / 4, 8, 12, 2);
  ctx.fill();
  ctx.stroke();

  ctx.restore();

  if (props.gameState === "playing") {
    drawDockingGuide();
  }
}

function drawDockingGuide() {
  const sh = ship.value;
  const port = station.value.dockingPort;
  const distance = getDistanceToPort();

  if (distance < 150) {
    ctx.strokeStyle = `rgba(72, 187, 120, ${1 - distance / 150})`;
    ctx.lineWidth = 1;
    ctx.setLineDash([5, 5]);
    ctx.beginPath();
    ctx.moveTo(sh.x, sh.y);
    ctx.lineTo(port.x, port.y);
    ctx.stroke();
    ctx.setLineDash([]);
  }
}

function drawHUD() {
  const speed = Math.sqrt(ship.value.vx ** 2 + ship.value.vy ** 2);
  const distance = getDistanceToPort();
  const maxSpeed = props.shipConfig.maxSpeed || 8;

  const hudHeight = props.obstacleMode ? 110 : 90;

  ctx.fillStyle = "rgba(0, 0, 0, 0.6)";
  ctx.fillRect(10, 10, 200, hudHeight);
  ctx.strokeStyle = "#4a5568";
  ctx.lineWidth = 1;
  ctx.strokeRect(10, 10, 200, hudHeight);

  ctx.fillStyle = "#e2e8f0";
  ctx.font = "bold 12px monospace";
  ctx.textAlign = "left";
  ctx.fillText(`速度: ${speed.toFixed(2)} u/s`, 20, 32);
  ctx.fillText(`距离: ${distance.toFixed(1)} px`, 20, 52);

  let speedColor = "#48bb78";
  if (speed > DOCKING_SPEED && speed < CRASH_SPEED) speedColor = "#ecc94b";
  if (speed >= CRASH_SPEED) speedColor = "#f56565";
  ctx.fillStyle = speedColor;
  ctx.fillRect(20, 62, Math.min((speed / maxSpeed) * 170, 170), 8);
  ctx.strokeStyle = "#4a5568";
  ctx.strokeRect(20, 62, 170, 8);

  ctx.fillStyle = "#e2e8f0";
  ctx.font = "10px monospace";
  ctx.fillText("对接", 22, 85);
  ctx.fillText("撞击", 165, 85);

  if (props.obstacleMode) {
    ctx.fillStyle = "#ed8936";
    ctx.font = "bold 11px monospace";
    ctx.fillText("☄ 障碍物模式", 20, 102);
  }

  const shipColor = props.shipConfig.color || "#63b3ed";
  ctx.fillStyle = shipColor;
  ctx.font = "bold 11px monospace";
  ctx.textAlign = "right";
  ctx.fillText(`🚀 ${props.shipConfig.name || "飞船"}`, CANVAS_WIDTH - 15, 28);

  ctx.fillStyle = "#718096";
  ctx.font = "10px monospace";
  ctx.fillText(
    "↑↓←→ 或 WASD 控制推进器",
    CANVAS_WIDTH - 15,
    CANVAS_HEIGHT - 15,
  );
}

function draw(timestamp) {
  if (!ctx) return;

  const dt = Math.min((timestamp - lastTime) / 16.67, 2);
  lastTime = timestamp;

  updatePhysics(dt);

  ctx.fillStyle = "#0a0a1a";
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  const nebulaGradient = ctx.createRadialGradient(
    CANVAS_WIDTH / 2,
    CANVAS_HEIGHT / 2,
    0,
    CANVAS_WIDTH / 2,
    CANVAS_HEIGHT / 2,
    400,
  );
  nebulaGradient.addColorStop(0, "rgba(50, 20, 80, 0.3)");
  nebulaGradient.addColorStop(1, "rgba(10, 10, 26, 0)");
  ctx.fillStyle = nebulaGradient;
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  drawStars();
  drawStation();
  if (props.obstacleMode) {
    drawObstacles();
  }
  drawShip();

  if (props.gameState === "playing") {
    drawHUD();
  }

  animationId = requestAnimationFrame(draw);
}

watch(
  () => props.gameState,
  (newState, oldState) => {
    if (oldState !== "playing" && newState === "playing") {
      initGame();
    }
  },
);

defineExpose({ initGame });

onMounted(() => {
  ctx = canvas.value.getContext("2d");
  initStars();
  initGame();
  lastTime = performance.now();
  animationId = requestAnimationFrame(draw);
});

onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId);
  }
});
</script>

<template>
  <canvas
    ref="canvas"
    :width="CANVAS_WIDTH"
    :height="CANVAS_HEIGHT"
    class="game-canvas"
  />
</template>

<style scoped>
.game-canvas {
  display: block;
  border: 2px solid #2d3748;
  border-radius: 12px;
  box-shadow:
    0 0 40px rgba(99, 179, 237, 0.15),
    inset 0 0 60px rgba(0, 0, 0, 0.5);
}
</style>
