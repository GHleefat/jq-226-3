<script setup>
import { computed } from "vue";

const props = defineProps({
  gameState: {
    type: String,
    default: "playing",
  },
  lastResult: {
    type: Object,
    default: null,
  },
  bestScore: {
    type: Number,
    default: 0,
  },
  attempts: {
    type: Number,
    default: 0,
  },
  successCount: {
    type: Number,
    default: 0,
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

const emit = defineEmits(["restart", "ship-change", "obstacle-mode-change"]);

const isObstacleCrash = computed(() => {
  return props.lastResult?.reason === "obstacle";
});

const scoreGrade = computed(() => {
  if (!props.lastResult?.score) return { text: "", color: "" };
  const s = props.lastResult.score;
  if (s >= 90) return { text: "完美！", color: "#48bb78" };
  if (s >= 75) return { text: "优秀！", color: "#68d391" };
  if (s >= 60) return { text: "良好", color: "#ecc94b" };
  if (s >= 40) return { text: "及格", color: "#ed8936" };
  return { text: "勉强成功", color: "#f56565" };
});

const successRate = computed(() => {
  if (props.attempts === 0) return 0;
  return Math.round((props.successCount / props.attempts) * 100);
});
</script>

<template>
  <div class="score-panel">
    <div class="panel-header">
      <h2 class="title">🚀 太空对接模拟</h2>
      <p class="subtitle">Space Docking Simulator</p>
    </div>

    <div class="stats-row">
      <div class="stat-card">
        <span class="stat-num">{{ attempts }}</span>
        <span class="stat-name">尝试次数</span>
      </div>
      <div class="stat-card">
        <span class="stat-num success">{{ successCount }}</span>
        <span class="stat-name">成功对接</span>
      </div>
      <div class="stat-card">
        <span class="stat-num rate">{{ successRate }}%</span>
        <span class="stat-name">成功率</span>
      </div>
      <div class="stat-card">
        <span class="stat-num best">{{ bestScore }}</span>
        <span class="stat-name">最高分</span>
      </div>
    </div>

    <div v-if="gameState === 'docked'" class="result-panel success">
      <div class="result-icon">✅</div>
      <h3 class="result-title" :style="{ color: scoreGrade.color }">
        对接成功 - {{ scoreGrade.text }}
      </h3>
      <div class="result-score">
        <span class="score-label">最终得分</span>
        <span class="score-value" :style="{ color: scoreGrade.color }">
          {{ lastResult?.score }}
        </span>
      </div>
      <div class="result-details">
        <div class="detail-item">
          <span>对接精度</span>
          <span>{{ lastResult?.distance?.toFixed(1) }} px</span>
        </div>
        <div class="detail-item">
          <span>相对速度</span>
          <span>{{ lastResult?.speed?.toFixed(2) }} u/s</span>
        </div>
      </div>
      <button class="action-btn success" @click="emit('restart')">
        再来一次
      </button>
    </div>

    <div v-else-if="gameState === 'crashed'" class="result-panel crash">
      <div class="result-icon">{{ isObstacleCrash ? "☄️" : "💥" }}</div>
      <h3 class="result-title">
        {{ isObstacleCrash ? "撞到太空垃圾！" : "对接失败" }}
      </h3>
      <p v-if="isObstacleCrash" class="crash-reason">
        飞船撞上了漂浮的太空碎片！<br />
        当前速度：<span class="crash-speed"
          >{{ lastResult?.speed?.toFixed(2) }} u/s</span
        >
      </p>
      <p v-else class="crash-reason">
        撞击速度过快！<br />
        <span class="crash-speed">{{ lastResult?.speed?.toFixed(2) }} u/s</span>
      </p>
      <p class="crash-hint">
        {{ isObstacleCrash ? '在障碍物模式下，请小心绕行所有漂浮碎片。' :
        '请将速度降至 <strong>0.5 u/s</strong> 以下再对接' }}
      </p>
      <button class="action-btn crash" @click="emit('restart')">
        重新尝试
      </button>
    </div>

    <div v-else class="playing-panel">
      <div class="objective">
        <h4>🎯 任务目标</h4>
        <ul>
          <li>缓慢靠近空间站对接口（绿色区域）</li>
          <li>保持相对速度 < 0.5 u/s</li>
          <li>距离对接口 < 30 px 时自动判定</li>
          <li>速度 > 3 u/s 会撞毁飞船！</li>
          <li v-if="obstacleMode" class="obstacle-warn">
            ⚠️ 小心绕行所有太空垃圾！
          </li>
        </ul>
      </div>
      <div class="ship-info">
        <h4>� 当前飞船</h4>
        <div
          class="ship-info-card"
          :style="{ '--ship-color': shipTypes[selectedShip]?.color }"
        >
          <div class="ship-info-name">{{ shipTypes[selectedShip]?.name }}</div>
          <div class="ship-info-desc">
            {{ shipTypes[selectedShip]?.description }}
          </div>
        </div>
      </div>
      <div class="tips">
        <h4>💡 操作提示</h4>
        <p>
          微重力环境下飞船不会自行停止，需要使用反向推力进行减速。耐心是关键！
        </p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.score-panel {
  width: 280px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  padding: 20px;
  background: linear-gradient(145deg, #1a1a2e, #16213e);
  border: 1px solid #2d3748;
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.4);
}

.panel-header {
  text-align: center;
  padding-bottom: 12px;
  border-bottom: 1px solid #2d3748;
}

.title {
  margin: 0;
  font-size: 20px;
  font-weight: 700;
  color: #e2e8f0;
}

.subtitle {
  margin: 4px 0 0;
  font-size: 11px;
  color: #718096;
  letter-spacing: 2px;
  text-transform: uppercase;
}

.stats-row {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
}

.stat-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  padding: 10px 6px;
  background: rgba(45, 55, 72, 0.5);
  border-radius: 8px;
}

.stat-num {
  font-size: 20px;
  font-weight: 700;
  font-family: "Courier New", monospace;
  color: #e2e8f0;
}

.stat-num.success {
  color: #48bb78;
}

.stat-num.rate {
  color: #63b3ed;
}

.stat-num.best {
  color: #ed8936;
}

.stat-name {
  font-size: 10px;
  color: #718096;
  text-transform: uppercase;
}

.result-panel {
  padding: 20px;
  border-radius: 12px;
  text-align: center;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.result-panel.success {
  background: linear-gradient(
    145deg,
    rgba(72, 187, 120, 0.15),
    rgba(72, 187, 120, 0.05)
  );
  border: 1px solid rgba(72, 187, 120, 0.4);
}

.result-panel.crash {
  background: linear-gradient(
    145deg,
    rgba(245, 101, 101, 0.15),
    rgba(245, 101, 101, 0.05)
  );
  border: 1px solid rgba(245, 101, 101, 0.4);
}

.result-icon {
  font-size: 48px;
}

.result-title {
  margin: 0;
  font-size: 20px;
  font-weight: 700;
  color: #e2e8f0;
}

.result-panel.crash .result-title {
  color: #f56565;
}

.result-score {
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 12px;
  background: rgba(0, 0, 0, 0.3);
  border-radius: 8px;
}

.score-label {
  font-size: 12px;
  color: #718096;
  text-transform: uppercase;
}

.score-value {
  font-size: 42px;
  font-weight: 800;
  font-family: "Courier New", monospace;
}

.result-details {
  display: flex;
  justify-content: center;
  gap: 20px;
}

.detail-item {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.detail-item span:first-child {
  font-size: 11px;
  color: #718096;
}

.detail-item span:last-child {
  font-size: 14px;
  font-weight: 600;
  color: #e2e8f0;
  font-family: "Courier New", monospace;
}

.crash-reason {
  margin: 0;
  font-size: 14px;
  color: #e2e8f0;
  line-height: 1.6;
}

.crash-speed {
  font-size: 24px;
  font-weight: 700;
  color: #f56565;
  font-family: "Courier New", monospace;
}

.crash-hint {
  margin: 0;
  font-size: 12px;
  color: #a0aec0;
  line-height: 1.5;
}

.action-btn {
  padding: 14px 24px;
  border: none;
  border-radius: 10px;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  margin-top: 4px;
}

.action-btn.success {
  background: linear-gradient(145deg, #48bb78, #38a169);
  color: #fff;
}

.action-btn.success:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(72, 187, 120, 0.4);
}

.action-btn.crash {
  background: linear-gradient(145deg, #f56565, #e53e3e);
  color: #fff;
}

.action-btn.crash:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(245, 101, 101, 0.4);
}

.action-btn:active {
  transform: translateY(0);
}

.playing-panel {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.objective h4,
.tips h4 {
  margin: 0 0 8px;
  font-size: 13px;
  font-weight: 600;
  color: #63b3ed;
}

.objective ul {
  margin: 0;
  padding-left: 18px;
  font-size: 12px;
  color: #a0aec0;
  line-height: 1.8;
}

.tips p {
  margin: 0;
  font-size: 12px;
  color: #a0aec0;
  line-height: 1.6;
  padding: 10px;
  background: rgba(237, 137, 54, 0.1);
  border-left: 3px solid #ed8936;
  border-radius: 4px;
}

.obstacle-warn {
  color: #ed8936 !important;
  font-weight: 600;
}

.ship-info {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.ship-info h4 {
  margin: 0;
  font-size: 13px;
  font-weight: 600;
  color: #63b3ed;
}

.ship-info-card {
  padding: 10px 12px;
  background: rgba(45, 55, 72, 0.5);
  border: 1px solid var(--ship-color, #63b3ed);
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.ship-info-name {
  font-size: 13px;
  font-weight: 600;
  color: var(--ship-color, #63b3ed);
}

.ship-info-desc {
  font-size: 11px;
  color: #a0aec0;
  line-height: 1.4;
}
</style>
