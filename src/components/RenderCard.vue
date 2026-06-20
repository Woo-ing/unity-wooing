<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';

// 3D 投影常量定义，采用 UPPER_SNAKE_CASE
const SPHERE_RADIUS = 70;
const D_PROJECTION = 200; // 投影视距
const POINT_COUNT = 80;
const BASE_ROTATION_SPEED = 0.005;

// 响应式与 Canvas 引用，采用 camelCase
const canvasRef = ref<HTMLCanvasElement | null>(null);
const isHovered = ref(false);
const mousePos = ref({ x: 0, y: 0 });
const isVisible = ref(false);

let animationFrameId: number;
let angleX = 0;
let angleY = 0;
let observer: IntersectionObserver | null = null;

// 生成球体表面上的 3D 点
interface Point3D {
  x: number;
  y: number;
  z: number;
}

const points: Point3D[] = [];
for (let i = 0; i < POINT_COUNT; i++) {
  const theta = Math.acos(Math.random() * 2 - 1);
  const phi = Math.random() * Math.PI * 2;
  points.push({
    x: SPHERE_RADIUS * Math.sin(theta) * Math.cos(phi),
    y: SPHERE_RADIUS * Math.sin(theta) * Math.sin(phi),
    z: SPHERE_RADIUS * Math.cos(theta)
  });
}

// 旋转点算法，采用 camelCase
const rotateX = (point: Point3D, angle: number): Point3D => {
  const cos = Math.cos(angle);
  const sin = Math.sin(angle);
  return {
    x: point.x,
    y: point.y * cos - point.z * sin,
    z: point.y * sin + point.z * cos
  };
};

const rotateY = (point: Point3D, angle: number): Point3D => {
  const cos = Math.cos(angle);
  const sin = Math.sin(angle);
  return {
    x: point.x * cos + point.z * sin,
    y: point.y,
    z: -point.x * sin + point.z * cos
  };
};

// 绘制循环方法，采用 camelCase
const drawFrame = () => {
  if (!isVisible.value) return;
  const canvas = canvasRef.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  // 清除画布
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // 计算旋转角度
  let speedX = BASE_ROTATION_SPEED;
  let speedY = BASE_ROTATION_SPEED;

  if (isHovered.value) {
    // 鼠标在卡片上时，根据鼠标位置调整旋转速度
    const rect = canvas.getBoundingClientRect();
    const cx = rect.left + rect.width / 2;
    const cy = rect.top + rect.height / 2;
    speedX = (mousePos.value.y - cy) * 0.0001;
    speedY = (mousePos.value.x - cx) * 0.0001;
  }

  angleX += speedX;
  angleY += speedY;

  // 投影并绘制点和连线
  const projectedPoints: { x: number; y: number; depth: number }[] = [];
  const cx = canvas.width / 2;
  const cy = canvas.height / 2;

  points.forEach((p) => {
    let rotated = rotateX(p, angleX);
    rotated = rotateY(rotated, angleY);

    // 透视投影公式
    const scale = D_PROJECTION / (D_PROJECTION + rotated.z);
    projectedPoints.push({
      x: rotated.x * scale + cx,
      y: rotated.y * scale + cy,
      depth: rotated.z // 缓存深度用于绘制颜色和连线透明度
    });
  });

  // 绘制连线：对临近点进行连线
  ctx.lineWidth = 0.5;
  for (let i = 0; i < projectedPoints.length; i++) {
    for (let j = i + 1; j < projectedPoints.length; j++) {
      const p1 = projectedPoints[i];
      const p2 = projectedPoints[j];
      const dx = p1.x - p2.x;
      const dy = p1.y - p2.y;
      const dist = Math.sqrt(dx * dx + dy * dy);

      // 距离较近时绘制半透明连线
      if (dist < 45) {
        const opacity = (1 - dist / 45) * 0.15;
        ctx.strokeStyle = `rgba(0, 180, 216, ${opacity})`;
        ctx.beginPath();
        ctx.moveTo(p1.x, p1.y);
        ctx.lineTo(p2.x, p2.y);
        ctx.stroke();
      }
    }
  }

  // 绘制点
  projectedPoints.forEach((p) => {
    // 根据深度调节亮度和大小 (Z 轴越靠近屏幕，点越大越亮)
    const alpha = (SPHERE_RADIUS - p.depth) / (SPHERE_RADIUS * 2) * 0.8 + 0.2;
    const radius = (SPHERE_RADIUS - p.depth) / (SPHERE_RADIUS * 2) * 3 + 1;
    ctx.fillStyle = `rgba(245, 128, 38, ${alpha})`;
    ctx.beginPath();
    ctx.arc(p.x, p.y, radius, 0, Math.PI * 2);
    ctx.fill();
  });

  animationFrameId = requestAnimationFrame(drawFrame);
};

// 交互事件监听，采用 camelCase
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
  observer = new IntersectionObserver((entries) => {
    entries.forEach((entry) => {
      const wasVisible = isVisible.value;
      isVisible.value = entry.isIntersecting;
      if (entry.isIntersecting && !wasVisible) {
        drawFrame();
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
    class="RenderCard GlassCard"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
    @mousemove="handleMouseMove"
  >
    <div class="CardVisual">
      <canvas ref="canvasRef" width="220" height="220" class="VisualCanvas"></canvas>
    </div>
    <div class="CardContent">
      <h3 class="CardTitle">Rendering 渲染</h3>
      <p class="CardDescription">
        在 Canvas 中利用 3D 透视投影算法进行三维点云和拓扑线框的旋转投影，展现矩阵变换的视觉张力。
      </p>
    </div>
  </div>
</template>

<style scoped>
.RenderCard {
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
