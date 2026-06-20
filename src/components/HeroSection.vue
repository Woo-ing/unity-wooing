<script setup lang="ts">
import { ref } from 'vue';

// 交互终端日志列表，采用 UPPER_SNAKE_CASE 作为常量定义
const CONSOLE_LOGS = [
  'Initializing Unity-Wooing engine...',
  'Loading rendering pipelines: CoreRP & URP... OK',
  'Setting up 3D skeleton joints & dual-quaternion blending... OK',
  'Initializing Verlet collision systems... OK',
  'Running dynamic bone spring simulation... ACTIVE',
  'Found a really useful game engine algorithm!'
];

// 响应式变量，采用 camelCase
const isHovered = ref(false);
const activeLogIndex = ref(0);

// 终端 hover 处理方法，采用 camelCase
const toggleHover = (state: boolean) => {
  isHovered.value = state;
};

// 交互式控制台点击方法，采用 camelCase
const nextLog = () => {
  if (activeLogIndex.value < CONSOLE_LOGS.length - 1) {
    activeLogIndex.value++;
  } else {
    activeLogIndex.value = 0; // 循环展示
  }
};
</script>

<template>
  <section class="HeroSection">
    <!-- 背景渐变光晕，采用 PascalCase 类名 -->
    <div class="GlowOrbs">
      <div class="OrbOne"></div>
      <div class="OrbTwo"></div>
    </div>

    <div class="HeroContent">
      <div class="AlgorithmBadge">
        <span class="BadgeDot"></span>
        <span class="BadgeText">Engine Algorithm Research</span>
      </div>

      <h1 class="SloganTitle">
        Found a really useful<br />
        <span class="PrimaryGradientText">game engine algorithm</span>
      </h1>
      
      <p class="HeroSubtitle">
        致力于探索渲染、高通用性骨骼动画、高效物理碰撞与拟真动态骨骼算法。
        提供在实际生产环境中高性能、易接入的 Unity 开发套件。
      </p>

      <div class="CtaButtons">
        <a href="#AnimationMirror" class="PrimaryButton">
          <span>查看核心作品</span>
          <svg class="ButtonIcon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <line x1="5" y1="12" x2="19" y2="12" />
            <polyline points="12 5 19 12 12 19" />
          </svg>
        </a>
        <a href="#TechFields" class="SecondaryButton">
          <span>了解技术技术栈</span>
        </a>
      </div>

      <!-- 交互控制台，体现极客感 -->
      <div 
        class="InteractiveConsole GlassCard"
        @mouseenter="toggleHover(true)"
        @mouseleave="toggleHover(false)"
        @click="nextLog"
        :class="{ 'ConsoleFocused': isHovered }"
        id="HeroInteractiveConsole"
      >
        <div class="ConsoleHeader">
          <div class="DotGroup">
            <span class="DotRed"></span>
            <span class="DotYellow"></span>
            <span class="DotGreen"></span>
          </div>
          <span class="ConsoleTitle">unity_wooing_console.sh</span>
        </div>
        <div class="ConsoleBody">
          <div 
            v-for="(log, index) in CONSOLE_LOGS.slice(0, activeLogIndex + 1)" 
            :key="index"
            class="ConsoleLine"
          >
            <span class="PromptSymbol">></span> {{ log }}
          </div>
          <div class="ConsoleHint" v-if="activeLogIndex < CONSOLE_LOGS.length - 1">
            点击控制台执行下一条引擎初始化指令...
          </div>
          <div class="ConsoleHint Finished" v-else>
            引擎初始化完成。点击重新循环。
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
.HeroSection {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  min-height: calc(100vh - 70px);
  padding-top: 40px;
}

.GlowOrbs {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  z-index: 0;
  pointer-events: none;
}

.OrbOne, .OrbTwo {
  position: absolute;
  border-radius: 50%;
  filter: blur(120px);
  opacity: 0.2;
}

.OrbOne {
  top: 20%;
  left: 15%;
  width: 300px;
  height: 300px;
  background-color: var(--primaryColor);
  animation: floatOrb 8s infinite alternate ease-in-out;
}

.OrbTwo {
  bottom: 20%;
  right: 15%;
  width: 350px;
  height: 350px;
  background-color: var(--secondaryColor);
  animation: floatOrb 10s infinite alternate-reverse ease-in-out;
}

@keyframes floatOrb {
  0% { transform: translateY(0) scale(1); }
  100% { transform: translateY(-30px) scale(1.15); }
}

.HeroContent {
  max-width: 800px;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 28px;
}

.AlgorithmBadge {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid var(--cardBorderColor);
  padding: 6px 14px;
  border-radius: 30px;
}

.BadgeDot {
  width: 8px;
  height: 8px;
  background-color: var(--primaryColor);
  border-radius: 50%;
  box-shadow: 0 0 8px var(--primaryColor);
}

.BadgeText {
  font-size: 13px;
  font-weight: 600;
  color: var(--textHighlightColor);
  letter-spacing: 0.05em;
  text-transform: uppercase;
}

.SloganTitle {
  font-size: 54px;
  line-height: 1.15;
  letter-spacing: -0.03em;
}

.HeroSubtitle {
  font-size: 18px;
  color: var(--textColor);
  max-width: 650px;
  line-height: 1.6;
}

.CtaButtons {
  display: flex;
  gap: 16px;
  margin-top: 8px;
}

.ButtonIcon {
  width: 16px;
  height: 16px;
}

/* 交互控制台 */
.InteractiveConsole {
  width: 100%;
  max-width: 580px;
  margin-top: 30px;
  text-align: left;
  border-radius: 12px;
  overflow: hidden;
  cursor: pointer;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.5);
}

.ConsoleFocused {
  border-color: rgba(245, 128, 38, 0.4);
  box-shadow: 0 20px 50px rgba(245, 128, 38, 0.08);
}

.ConsoleHeader {
  background: rgba(255, 255, 255, 0.02);
  border-bottom: 1px solid var(--cardBorderColor);
  padding: 10px 16px;
  display: flex;
  align-items: center;
  gap: 16px;
}

.DotGroup {
  display: flex;
  gap: 6px;
}

.DotGroup span {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  display: inline-block;
}

.DotRed { background-color: #ff5f56; }
.DotYellow { background-color: #ffbd2e; }
.DotGreen { background-color: #27c93f; }

.ConsoleTitle {
  font-family: var(--fontMono);
  font-size: 12px;
  color: var(--textColor);
  opacity: 0.8;
}

.ConsoleBody {
  padding: 16px;
  font-family: var(--fontMono);
  font-size: 13px;
  min-height: 180px;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  gap: 8px;
  background-color: rgba(5, 5, 8, 0.95);
}

.ConsoleLine {
  color: #c5c9db;
  line-height: 1.5;
}

.PromptSymbol {
  color: var(--primaryColor);
  font-weight: bold;
}

.ConsoleHint {
  margin-top: auto;
  font-size: 11px;
  color: var(--secondaryColor);
  opacity: 0.7;
  animation: flashText 1.5s infinite alternate;
}

.Finished {
  color: var(--primaryColor);
}

@keyframes flashText {
  0% { opacity: 0.4; }
  100% { opacity: 1; }
}

@media (max-width: 768px) {
  .SloganTitle {
    font-size: 38px;
  }
  .HeroSubtitle {
    font-size: 15px;
  }
  .InteractiveConsole {
    max-width: 90%;
  }
}
</style>
