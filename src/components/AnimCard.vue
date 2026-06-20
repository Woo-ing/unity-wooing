<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';

// 动画常量，采用 UPPER_SNAKE_CASE
const SEGMENT_COUNT = 3;
const SEGMENT_LENGTH = 45;
const BASE_WAVE_SPEED = 0.04;

// Canvas 与状态，采用 camelCase
const canvasRef = ref<HTMLCanvasElement | null>(null);
const isHovered = ref(false);
const mousePos = ref({ x: 0, y: 0 });
const isVisible = ref(false);

let animationFrameId: number;
let time = 0;
let observer: IntersectionObserver | null = null;

// 关节节点类定义，采用 PascalCase
interface Joint {
  x: number;
  y: number;
  angle: number;
}

// 渲染关节和骨骼的逻辑，采用 camelCase
const drawAnimation = () => {
  if (!isVisible.value) return;
  const canvas = canvasRef.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  ctx.clearRect(0, 0, canvas.width, canvas.height);

  const startX = canvas.width / 2;
  const startY = canvas.height - 40; // 底部作为骨骼基座
  
  // 更新时间
  time += isHovered.value ? BASE_WAVE_SPEED * 1.8 : BASE_WAVE_SPEED;

  // 存储各骨骼关节数据
  const joints: Joint[] = [{ x: startX, y: startY, angle: -Math.PI / 2 }];

  // 基础角度计算与波形偏移
  for (let i = 1; i <= SEGMENT_COUNT; i++) {
    const parent = joints[i - 1];
    
    // 用正弦波控制每个关节的局部旋转角 (实现波浪摆动)
    let localAngle = 0.3 * Math.sin(time - i * 0.8) / i;

    if (isHovered.value) {
      // 悬停时，计算关节指向鼠标方向的偏角 (简易 IK 朝向)
      const rect = canvas.getBoundingClientRect();
      const mx = mousePos.value.x - rect.left;
      const my = mousePos.value.y - rect.top;
      
      if (i === 1) {
        // 基底骨骼朝向鼠标
        const angleToMouse = Math.atan2(my - parent.y, mx - parent.x);
        // 平滑插值，避免突变
        const diff = angleToMouse - parent.angle;
        localAngle = diff * 0.1;
      } else {
        // 子骨骼受到惯性和鼠标拉力的扰动
        localAngle += (Math.sin(time * 1.5 - i) * 0.4);
      }
    }

    const globalAngle = parent.angle + localAngle;
    
    const nextX = parent.x + SEGMENT_LENGTH * Math.cos(globalAngle);
    const nextY = parent.y + SEGMENT_LENGTH * Math.sin(globalAngle);

    joints.push({
      x: nextX,
      y: nextY,
      angle: globalAngle
    });
  }

  // 绘制骨架线段 (Bones)
  ctx.lineCap = 'round';
  for (let i = 0; i < joints.length - 1; i++) {
    const p1 = joints[i];
    const p2 = joints[i + 1];

    // 骨骼渐变色
    ctx.strokeStyle = `rgba(0, 180, 216, ${1.0 - i * 0.25})`;
    ctx.lineWidth = 14 - i * 3.5;
    
    ctx.beginPath();
    ctx.moveTo(p1.x, p1.y);
    ctx.lineTo(p2.x, p2.y);
    ctx.stroke();

    // 绘制内部连接线 (类似关节骨髓线)
    ctx.strokeStyle = '#ffffff';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(p1.x, p1.y);
    ctx.lineTo(p2.x, p2.y);
    ctx.stroke();
  }

  // 绘制关节圆点 (Joints)
  joints.forEach((joint, index) => {
    // 根部关节和尖端关节大小不同
    const radius = 8 - index * 1.5;
    ctx.fillStyle = index === joints.length - 1 ? 'var(--primaryColor)' : '#1c202e';
    ctx.strokeStyle = 'var(--primaryColor)';
    ctx.lineWidth = 2;

    ctx.beginPath();
    ctx.arc(joint.x, joint.y, radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.stroke();
  });

  // 绘制装饰网格或骨骼标识符号
  ctx.font = '10px JetBrains Mono';
  ctx.fillStyle = 'rgba(255, 255, 255, 0.25)';
  ctx.fillText('BONE_CHAIN_LOD0', 15, 20);
  
  if (isHovered.value) {
    ctx.fillStyle = 'var(--primaryColor)';
    ctx.fillText('IK_ACTIVE', 15, 35);
  }

  animationFrameId = requestAnimationFrame(drawAnimation);
};

// 鼠标位置监听方法，采用 camelCase
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
        drawAnimation();
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
    class="AnimCard GlassCard"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
    @mousemove="handleMouseMove"
  >
    <div class="CardVisual">
      <canvas ref="canvasRef" width="220" height="220" class="VisualCanvas"></canvas>
    </div>
    <div class="CardContent">
      <h3 class="CardTitle">Animation 动画</h3>
      <p class="CardDescription">
        在 Canvas 中利用正弦级数控制关节层次旋转树，实现流畅的多关节摆动骨架，并结合简易反向动力学 (IK) 的鼠标朝向交互。
      </p>
    </div>
  </div>
</template>

<style scoped>
.AnimCard {
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
