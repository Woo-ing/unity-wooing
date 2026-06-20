<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';

// 动态骨骼物理常量定义，采用 UPPER_SNAKE_CASE
const SEGMENT_COUNT = 8;
const SEGMENT_LENGTH = 18;
const GRAVITY_Y = 0.25;
const DAMPING = 0.985;
const NUM_ITERATIONS = 5; // 约束迭代次数

// Canvas 与状态，采用 camelCase
const canvasRef = ref<HTMLCanvasElement | null>(null);
const isHovered = ref(false);
const mousePos = ref({ x: 0, y: 0 });
const isVisible = ref(false);

let animationFrameId: number;
let observer: IntersectionObserver | null = null;

// Verlet 质点类，采用 PascalCase
interface VerletNode {
  x: number;
  y: number;
  oldX: number;
  oldY: number;
}

const nodes: VerletNode[] = [];

// 初始化骨骼节点链，起始于画布顶部中心
const initNodes = (startX: number, startY: number) => {
  nodes.length = 0;
  for (let i = 0; i <= SEGMENT_COUNT; i++) {
    nodes.push({
      x: startX,
      y: startY + i * SEGMENT_LENGTH,
      oldX: startX,
      oldY: startY + i * SEGMENT_LENGTH
    });
  }
};

// 物理演算循环方法，采用 camelCase
const updateVerlet = () => {
  if (!isVisible.value) return;
  const canvas = canvasRef.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  ctx.clearRect(0, 0, canvas.width, canvas.height);

  const rect = canvas.getBoundingClientRect();
  const mx = mousePos.value.x - rect.left;
  const my = mousePos.value.y - rect.top;

  // 1. 固定根节点 (如果鼠标移入，随鼠标移动，否则悬挂在顶部中心)
  const targetRootX = isHovered.value ? mx : canvas.width / 2;
  const targetRootY = isHovered.value ? my : 30;

  // 平滑插值更新根节点位置以防止骨骼断裂
  nodes[0].x += (targetRootX - nodes[0].x) * 0.25;
  nodes[0].y += (targetRootY - nodes[0].y) * 0.25;

  // 2. 应用 Verlet 积分计算所有质点新位置
  for (let i = 1; i < nodes.length; i++) {
    const node = nodes[i];
    const vx = (node.x - node.oldX) * DAMPING;
    const vy = (node.y - node.oldY) * DAMPING;

    // 记录旧位置
    node.oldX = node.x;
    node.oldY = node.y;

    // 加上速度和重力
    node.x += vx;
    node.y += vy + GRAVITY_Y;
  }

  // 3. 约束消解 (保持相邻节点间的线段长度)
  for (let iter = 0; iter < NUM_ITERATIONS; iter++) {
    // 根节点固定不参与自由移动
    for (let i = 1; i < nodes.length; i++) {
      const p1 = nodes[i - 1];
      const p2 = nodes[i];

      const dx = p2.x - p1.x;
      const dy = p2.y - p1.y;
      const dist = Math.sqrt(dx * dx + dy * dy);
      const diff = SEGMENT_LENGTH - dist;
      
      // 距离误差按比例推回
      const percent = (diff / dist) * 0.5;
      const offsetX = dx * percent;
      const offsetY = dy * percent;

      // 根节点不需要回拉，只需拉动子节点
      if (i === 1) {
        p2.x += offsetX * 2;
        p2.y += offsetY * 2;
      } else {
        p1.x -= offsetX;
        p1.y -= offsetY;
        p2.x += offsetX;
        p2.y += offsetY;
      }
    }
  }

  // 4. 绘制骨架线段
  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';
  for (let i = 0; i < nodes.length - 1; i++) {
    const p1 = nodes[i];
    const p2 = nodes[i + 1];

    ctx.strokeStyle = `rgba(0, 180, 216, ${1.0 - i * 0.1})`;
    ctx.lineWidth = 12 - i * 1.2;

    ctx.beginPath();
    ctx.moveTo(p1.x, p1.y);
    ctx.lineTo(p2.x, p2.y);
    ctx.stroke();

    // 内部高亮核心骨骼线
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.4)';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(p1.x, p1.y);
    ctx.lineTo(p2.x, p2.y);
    ctx.stroke();
  }

  // 5. 绘制关节圆点
  nodes.forEach((node, index) => {
    const radius = 6 - index * 0.5;
    ctx.fillStyle = index === nodes.length - 1 ? 'var(--primaryColor)' : '#1c202e';
    ctx.strokeStyle = 'var(--primaryColor)';
    ctx.lineWidth = 1.5;

    ctx.beginPath();
    ctx.arc(node.x, node.y, radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.stroke();
  });

  // 绘制调试状态标签
  ctx.font = '10px JetBrains Mono';
  ctx.fillStyle = 'rgba(255, 255, 255, 0.25)';
  ctx.fillText('VERLET_SOLVER_DBONES', 15, 20);

  animationFrameId = requestAnimationFrame(updateVerlet);
};

// 交互事件监听方法，采用 camelCase
const handleMouseMove = (e: MouseEvent) => {
  mousePos.value = { x: e.clientX, y: e.clientY };
};

const handleMouseEnter = () => {
  isHovered.value = true;
};

const handleMouseLeave = () => {
  isHovered.value = false;
};

onMounted(() => {
  const canvas = canvasRef.value;
  if (canvas) {
    initNodes(canvas.width / 2, 30);
  }

  observer = new IntersectionObserver((entries) => {
    entries.forEach((entry) => {
      const wasVisible = isVisible.value;
      isVisible.value = entry.isIntersecting;
      if (entry.isIntersecting && !wasVisible) {
        updateVerlet();
      }
    });
  }, { threshold: 0.01 });

  if (canvasRef.value) {
    observer.observe(canvasRef.value);
  }
});

onUnmounted(() => {
  if (observer) {
    observer.disconnect();
  }
  cancelAnimationFrame(animationFrameId);
});
</script>

<template>
  <div 
    class="BoneCard GlassCard"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
    @mousemove="handleMouseMove"
  >
    <div class="CardVisual">
      <canvas ref="canvasRef" width="220" height="220" class="VisualCanvas"></canvas>
    </div>
    <div class="CardContent">
      <h3 class="CardTitle">Dynamic Bones 动态骨骼</h3>
      <p class="CardDescription">
        使用 Verlet 积分与多级长度硬约束求值器，构造带惯性延迟的动态悬挂骨链，展示柔软尾发或摇摆物的实时物理模拟。
      </p>
    </div>
  </div>
</template>

<style scoped>
.BoneCard {
  padding: 24px;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  gap: 16px;
  height: 100%;
}

.CardVisual {
  width: 220px;
  height: 220px;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
}

.VisualCanvas {
  width: 220px;
  height: 220px;
  display: block;
}

.CardContent {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.CardTitle {
  font-size: 20px;
  color: var(--textHighlightColor);
  font-weight: 600;
}

.CardDescription {
  font-size: 13px;
  color: var(--textColor);
  line-height: 1.5;
}
</style>
