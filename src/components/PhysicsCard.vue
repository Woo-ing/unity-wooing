<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';

// 物理模拟常量定义，采用 UPPER_SNAKE_CASE
const PARTICLE_COUNT = 12;
const MIN_RADIUS = 6;
const MAX_RADIUS = 12;
const FORCE_RADIUS = 55; // 鼠标排斥半径
const REPELLING_FORCE = 0.2; // 排斥力大小

// Canvas 与状态，采用 camelCase
const canvasRef = ref<HTMLCanvasElement | null>(null);
const isHovered = ref(false);
const mousePos = ref({ x: 0, y: 0 });
const isVisible = ref(false);

let animationFrameId: number;
let observer: IntersectionObserver | null = null;

// 粒子类型定义，采用 PascalCase
interface Particle {
  x: number;
  y: number;
  vx: number;
  vy: number;
  radius: number;
  color: string;
}

const particles: Particle[] = [];

// 初始化粒子列表
const initParticles = (width: number, height: number) => {
  particles.length = 0;
  for (let i = 0; i < PARTICLE_COUNT; i++) {
    const radius = Math.random() * (MAX_RADIUS - MIN_RADIUS) + MIN_RADIUS;
    const x = Math.random() * (width - radius * 2) + radius;
    const y = Math.random() * (height - radius * 2) + radius;
    const speed = Math.random() * 1.5 + 0.5;
    const angle = Math.random() * Math.PI * 2;
    particles.push({
      x,
      y,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed,
      radius,
      // 粒子颜色在橙色和青蓝色之间变化
      color: Math.random() > 0.5 ? 'var(--primaryColor)' : 'var(--secondaryColor)'
    });
  }
};

// 弹性碰撞与移动算法，采用 camelCase
const updatePhysics = () => {
  if (!isVisible.value) return;
  const canvas = canvasRef.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  ctx.clearRect(0, 0, canvas.width, canvas.height);

  const rect = canvas.getBoundingClientRect();
  const mx = mousePos.value.x - rect.left;
  const my = mousePos.value.y - rect.top;

  // 1. 位置更新与边界碰撞
  particles.forEach((p) => {
    // 鼠标排斥效应
    if (isHovered.value) {
      const dx = p.x - mx;
      const dy = p.y - my;
      const dist = Math.sqrt(dx * dx + dy * dy);
      if (dist < FORCE_RADIUS) {
        // 计算排斥加速度并施加到速度上
        const force = (FORCE_RADIUS - dist) / FORCE_RADIUS * REPELLING_FORCE;
        const angle = Math.atan2(dy, dx);
        p.vx += Math.cos(angle) * force;
        p.vy += Math.sin(angle) * force;
      }
    }

    // 限制速度大小以防穿墙
    const currentSpeed = Math.sqrt(p.vx * p.vx + p.vy * p.vy);
    const maxSpeed = 3.5;
    if (currentSpeed > maxSpeed) {
      p.vx = (p.vx / currentSpeed) * maxSpeed;
      p.vy = (p.vy / currentSpeed) * maxSpeed;
    }

    // 移动
    p.x += p.vx;
    p.y += p.vy;

    // 边界碰撞
    if (p.x - p.radius < 0) {
      p.x = p.radius;
      p.vx *= -1;
    } else if (p.x + p.radius > canvas.width) {
      p.x = canvas.width - p.radius;
      p.vx *= -1;
    }

    if (p.y - p.radius < 0) {
      p.y = p.radius;
      p.vy *= -1;
    } else if (p.y + p.radius > canvas.height) {
      p.y = canvas.height - p.radius;
      p.vy *= -1;
    }
  });

  // 2. 粒子间碰撞 (双重循环球形碰撞消解)
  for (let i = 0; i < particles.length; i++) {
    for (let j = i + 1; j < particles.length; j++) {
      const p1 = particles[i];
      const p2 = particles[j];

      const dx = p2.x - p1.x;
      const dy = p2.y - p1.y;
      const dist = Math.sqrt(dx * dx + dy * dy);
      const minDist = p1.radius + p2.radius;

      if (dist < minDist) {
        // 重叠消解 (推开避免穿透)
        const overlap = minDist - dist;
        const nx = dx / dist; // 法向量
        const ny = dy / dist;

        p1.x -= nx * overlap * 0.5;
        p1.y -= ny * overlap * 0.5;
        p2.x += nx * overlap * 0.5;
        p2.y += ny * overlap * 0.5;

        // 弹性碰撞冲量公式
        // 法线上的速度分量
        const kx = p1.vx - p2.vx;
        const ky = p1.vy - p2.vy;
        const vn = kx * nx + ky * ny;

        if (vn > 0) {
          // 两个粒子正在相向运动才需要处理
          // 假设质量等同于半径大小的平方
          const m1 = p1.radius * p1.radius;
          const m2 = p2.radius * p2.radius;
          const impulse = (2 * vn) / (m1 + m2);

          p1.vx -= impulse * m2 * nx;
          p1.vy -= impulse * m2 * ny;
          p2.vx += impulse * m1 * nx;
          p2.vy += impulse * m1 * ny;
        }
      }
    }
  }

  // 3. 绘制粒子
  particles.forEach((p) => {
    ctx.fillStyle = p.color;
    ctx.shadowBlur = 4;
    ctx.shadowColor = p.color;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.shadowBlur = 0; // 重置阴影以保证性能
  });

  // 4. 绘制排斥圈提示线
  if (isHovered.value) {
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.08)';
    ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.arc(mx, my, FORCE_RADIUS, 0, Math.PI * 2);
    ctx.stroke();
  }

  // 绘制调试状态标签
  ctx.font = '10px JetBrains Mono';
  ctx.fillStyle = 'rgba(255, 255, 255, 0.25)';
  ctx.fillText('COLLISION_RESOLVER_AABB', 15, 20);

  animationFrameId = requestAnimationFrame(updatePhysics);
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
    initParticles(canvas.width, canvas.height);
  }

  observer = new IntersectionObserver((entries) => {
    entries.forEach((entry) => {
      const wasVisible = isVisible.value;
      isVisible.value = entry.isIntersecting;
      if (entry.isIntersecting && !wasVisible) {
        updatePhysics();
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
    class="PhysicsCard GlassCard"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
    @mousemove="handleMouseMove"
  >
    <div class="CardVisual">
      <canvas ref="canvasRef" width="220" height="220" class="VisualCanvas"></canvas>
    </div>
    <div class="CardContent">
      <h3 class="CardTitle">Collisions 碰撞</h3>
      <p class="CardDescription">
        用 2D Canvas 实现多圆体刚性弹性碰撞，模拟物理冲量传递和重叠校正，并结合鼠标斥力场进行交互反馈。
      </p>
    </div>
  </div>
</template>

<style scoped>
.PhysicsCard {
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
