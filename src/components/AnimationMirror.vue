<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue';

// 核心常量，采用 UPPER_SNAKE_CASE
const VIEWPORT_WIDTH = 640;
const VIEWPORT_HEIGHT = 320;
const ORIGINAL_CENTER_X = 160;
const MIRROR_CENTER_X = 480;
const GROUND_Y = 240;

const BONE_MAPPINGS = [
  { src: 'LeftShoulder', dst: 'RightShoulder', status: 'Mapped' },
  { src: 'LeftElbow', dst: 'RightElbow', status: 'Mapped' },
  { src: 'LeftHand', dst: 'RightHand', status: 'Mapped' },
  { src: 'LeftHip', dst: 'RightHip', status: 'Mapped' },
  { src: 'LeftKnee', dst: 'RightKnee', status: 'Mapped' },
  { src: 'LeftFoot', dst: 'RightFoot', status: 'Mapped' }
];

// 响应式状态定义，采用 camelCase
const canvasRef = ref<HTMLCanvasElement | null>(null);
const activeAnimation = ref<'walk' | 'wave'>('walk');
const activeAlgorithm = ref<'direct' | 'boneMapping'>('boneMapping');
const showSkeletonDebug = ref(true);
const playbackSpeed = ref(1.0);
const isPlaying = ref(true);
const isVisible = ref(false);

let animationFrameId: number;
let time = 0;
let observer: IntersectionObserver | null = null;

// 节点结构体，采用 PascalCase
interface Joint2D {
  name: string;
  x: number;
  y: number;
  parentName?: string;
}

// 骨骼连接线，用于渲染
const BONE_CONNECTIONS = [
  { from: 'Hip', to: 'Spine' },
  { from: 'Spine', to: 'Neck' },
  { from: 'Neck', to: 'Head' },
  // 胳膊
  { from: 'Neck', to: 'LeftShoulder' },
  { from: 'LeftShoulder', to: 'LeftElbow' },
  { from: 'LeftElbow', to: 'LeftHand' },
  { from: 'Neck', to: 'RightShoulder' },
  { from: 'RightShoulder', to: 'RightElbow' },
  { from: 'RightElbow', to: 'RightHand' },
  // 腿
  { from: 'Hip', to: 'LeftHip' },
  { from: 'LeftHip', to: 'LeftKnee' },
  { from: 'LeftKnee', to: 'LeftFoot' },
  { from: 'Hip', to: 'RightHip' },
  { from: 'RightHip', to: 'RightKnee' },
  { from: 'RightKnee', to: 'RightFoot' }
];

// 骨骼角度计算，基于时间与当前动画，返回原始骨骼节点位置，采用 camelCase
const computeOriginalJoints = (t: number): Record<string, Joint2D> => {
  const joints: Record<string, Joint2D> = {};

  // 基础躯干位置 (相对于局部原点)
  let hipX = 0;
  let hipY = -65;
  let spineY = -105;
  let neckY = -125;
  let headY = -145;

  if (activeAnimation.value === 'walk') {
    // 走路时臀部轻微起伏
    hipY += Math.sin(t * 2) * 3;
    spineY += Math.sin(t * 2) * 2;
    neckY += Math.sin(t * 2) * 1.5;
    headY += Math.sin(t * 2) * 1;
  }

  joints['Hip'] = { name: 'Hip', x: hipX, y: hipY };
  joints['Spine'] = { name: 'Spine', x: hipX, y: spineY, parentName: 'Hip' };
  joints['Neck'] = { name: 'Neck', x: hipX, y: neckY, parentName: 'Spine' };
  joints['Head'] = { name: 'Head', x: hipX, y: headY, parentName: 'Neck' };

  // 左右关节计算
  if (activeAnimation.value === 'walk') {
    // 走路动画：双腿和手臂呈反相位正弦运动
    const cycle = t;

    // 左腿
    const leftHipAngle = 0.5 * Math.sin(cycle);
    const leftKneeAngle = Math.sin(cycle) > 0 ? 0.6 * Math.sin(cycle) : 0.1;
    // 右腿
    const rightHipAngle = 0.5 * Math.sin(cycle + Math.PI);
    const rightKneeAngle = Math.sin(cycle + Math.PI) > 0 ? 0.6 * Math.sin(cycle + Math.PI) : 0.1;

    // 左臂
    const leftShoulderAngle = Math.PI/2 + 0.4 * Math.sin(cycle + Math.PI);
    const leftElbowAngle = 0.3 + 0.2 * Math.sin(cycle + Math.PI);
    // 右臂
    const rightShoulderAngle = Math.PI/2 + 0.4 * Math.sin(cycle);
    const rightElbowAngle = 0.3 + 0.2 * Math.sin(cycle);

    // 腿部关节坐标推算 (三角函数)
    const lhX = -12;
    const lhY = hipY;
    const lkX = lhX + 30 * Math.sin(leftHipAngle);
    const lkY = lhY + 30 * Math.cos(leftHipAngle);
    const lfX = lkX + 25 * Math.sin(leftHipAngle - leftKneeAngle);
    const lfY = lkY + 25 * Math.cos(leftHipAngle - leftKneeAngle);

    const rhX = 12;
    const rhY = hipY;
    const rkX = rhX + 30 * Math.sin(rightHipAngle);
    const rkY = rhY + 30 * Math.cos(rightHipAngle);
    const rfX = rkX + 25 * Math.sin(rightHipAngle - rightKneeAngle);
    const rfY = rkY + 25 * Math.cos(rightHipAngle - rightKneeAngle);

    // 手臂关节坐标推算
    const lsX = -14;
    const lsY = neckY + 5;
    const leX = lsX - 25 * Math.cos(leftShoulderAngle);
    const leY = lsY + 25 * Math.sin(leftShoulderAngle);
    const lhandX = leX - 20 * Math.cos(leftShoulderAngle + leftElbowAngle);
    const lhandY = leY + 20 * Math.sin(leftShoulderAngle + leftElbowAngle);

    const rsX = 14;
    const rsY = neckY + 5;
    const reX = rsX + 25 * Math.cos(rightShoulderAngle);
    const reY = rsY + 25 * Math.sin(rightShoulderAngle);
    const rhandX = reX + 20 * Math.cos(rightShoulderAngle + rightElbowAngle);
    const rhandY = reY + 20 * Math.sin(rightShoulderAngle + rightElbowAngle);

    joints['LeftHip'] = { name: 'LeftHip', x: lhX, y: lhY, parentName: 'Hip' };
    joints['LeftKnee'] = { name: 'LeftKnee', x: lkX, y: lkY, parentName: 'LeftHip' };
    joints['LeftFoot'] = { name: 'LeftFoot', x: lfX, y: lfY, parentName: 'LeftKnee' };

    joints['RightHip'] = { name: 'RightHip', x: rhX, y: rhY, parentName: 'Hip' };
    joints['RightKnee'] = { name: 'RightKnee', x: rkX, y: rkY, parentName: 'RightHip' };
    joints['RightFoot'] = { name: 'RightFoot', x: rfX, y: rfY, parentName: 'RightKnee' };

    joints['LeftShoulder'] = { name: 'LeftShoulder', x: lsX, y: lsY, parentName: 'Neck' };
    joints['LeftElbow'] = { name: 'LeftElbow', x: leX, y: leY, parentName: 'LeftShoulder' };
    joints['LeftHand'] = { name: 'LeftHand', x: lhandX, y: lhandY, parentName: 'LeftElbow' };

    joints['RightShoulder'] = { name: 'RightShoulder', x: rsX, y: rsY, parentName: 'Neck' };
    joints['RightElbow'] = { name: 'RightElbow', x: reX, y: reY, parentName: 'RightShoulder' };
    joints['RightHand'] = { name: 'RightHand', x: rhandX, y: rhandY, parentName: 'RightElbow' };

  } else {
    // 招手动画 (非对称动画)：左臂静止垂下，右臂高举快速挥舞
    const cycle = t * 2;

    const leftShoulderAngle = Math.PI / 2 + 0.1;
    const leftElbowAngle = 0.2;

    // 右臂举起并左右摆动
    const rightShoulderAngle = -Math.PI / 3 + 0.3 * Math.sin(cycle);
    const rightElbowAngle = 1.0 + 0.5 * Math.sin(cycle);

    // 静态腿部
    const lhX = -12;
    const lhY = hipY;
    const lkX = lhX;
    const lkY = lhY + 30;
    const lfX = lkX;
    const lfY = lkY + 25;

    const rhX = 12;
    const rhY = hipY;
    const rkX = rhX;
    const rkY = rhY + 30;
    const rfX = rkX;
    const rfY = rkY + 25;

    // 手臂坐标推算
    const lsX = -14;
    const lsY = neckY + 5;
    const leX = lsX - 25 * Math.cos(leftShoulderAngle);
    const leY = lsY + 25 * Math.sin(leftShoulderAngle);
    const lhandX = leX - 20 * Math.cos(leftShoulderAngle + leftElbowAngle);
    const lhandY = leY + 20 * Math.sin(leftShoulderAngle + leftElbowAngle);

    const rsX = 14;
    const rsY = neckY + 5;
    const reX = rsX + 25 * Math.cos(rightShoulderAngle);
    const reY = rsY + 25 * Math.sin(rightShoulderAngle);
    const rhandX = reX + 20 * Math.cos(rightShoulderAngle + rightElbowAngle);
    const rhandY = reY + 20 * Math.sin(rightShoulderAngle + rightElbowAngle);

    joints['LeftHip'] = { name: 'LeftHip', x: lhX, y: lhY, parentName: 'Hip' };
    joints['LeftKnee'] = { name: 'LeftKnee', x: lkX, y: lkY, parentName: 'LeftHip' };
    joints['LeftFoot'] = { name: 'LeftFoot', x: lfX, y: lfY, parentName: 'LeftKnee' };

    joints['RightHip'] = { name: 'RightHip', x: rhX, y: rhY, parentName: 'Hip' };
    joints['RightKnee'] = { name: 'RightKnee', x: rkX, y: rkY, parentName: 'RightHip' };
    joints['RightFoot'] = { name: 'RightFoot', x: rfX, y: rfY, parentName: 'RightKnee' };

    joints['LeftShoulder'] = { name: 'LeftShoulder', x: lsX, y: lsY, parentName: 'Neck' };
    joints['LeftElbow'] = { name: 'LeftElbow', x: leX, y: leY, parentName: 'LeftShoulder' };
    joints['LeftHand'] = { name: 'LeftHand', x: lhandX, y: lhandY, parentName: 'LeftElbow' };

    joints['RightShoulder'] = { name: 'RightShoulder', x: rsX, y: rsY, parentName: 'Neck' };
    joints['RightElbow'] = { name: 'RightElbow', x: reX, y: reY, parentName: 'RightShoulder' };
    joints['RightHand'] = { name: 'RightHand', x: rhandX, y: rhandY, parentName: 'RightElbow' };
  }

  return joints;
};

// 镜像算法实现，根据选择的镜像算法处理并计算镜像后的关节位置，采用 camelCase
const computeMirroredJoints = (originalJoints: Record<string, Joint2D>): Record<string, Joint2D> => {
  const mirrored: Record<string, Joint2D> = {};

  if (activeAlgorithm.value === 'direct') {
    // 算法1：直接几何体翻转 (Direct Geometry Flip)
    // 所有骨骼的 X 坐标取负号。不对左右侧骨骼做对称命名映射
    // 原来的右臂(快速摆动)在镜像后变成了在左侧(从左侧伸出来)的右臂，几何表现为简单的左右物理颠倒
    Object.keys(originalJoints).forEach((key) => {
      const orig = originalJoints[key];
      mirrored[key] = {
        name: orig.name,
        x: -orig.x, // X 坐标翻转
        y: orig.y,
        parentName: orig.parentName
      };
    });
  } else {
    // 算法2：对称骨骼名映射镜像 (Bone Mapping Symmetric)
    // 这是 AnimationMirror 的核心算法！
    // 1. 将原左侧的数据赋给镜像后的右侧；将原右侧的数据赋给镜像后的左侧。
    // 2. 同时翻转 X 轴偏移量，保证镜像后肢体位置朝向正确。
    // 例如：镜像后的 LeftShoulder 取的是原 RightShoulder 的数据翻转 X 轴。
    
    // 对称命名映射表
    const symmetryMap: Record<string, string> = {
      'LeftShoulder': 'RightShoulder', 'RightShoulder': 'LeftShoulder',
      'LeftElbow': 'RightElbow', 'RightElbow': 'LeftElbow',
      'LeftHand': 'RightHand', 'RightHand': 'LeftHand',
      'LeftHip': 'RightHip', 'RightHip': 'LeftHip',
      'LeftKnee': 'RightKnee', 'RightKnee': 'LeftKnee',
      'LeftFoot': 'RightFoot', 'RightFoot': 'LeftFoot'
    };

    Object.keys(originalJoints).forEach((key) => {
      const orig = originalJoints[key];
      const targetKey = symmetryMap[key] || key; // 如果有对称骨骼则映射，没有（如中轴骨骼）则保持原名

      // 取出数据，翻转 X
      mirrored[targetKey] = {
        name: targetKey,
        x: -orig.x,
        y: orig.y,
        parentName: orig.parentName ? (symmetryMap[orig.parentName!] || orig.parentName) : undefined
      };
    });
  }

  return mirrored;
};

// 绘制骨骼连线方法，采用 camelCase
const drawSkeleton = (ctx: CanvasRenderingContext2D, joints: Record<string, Joint2D>, centerX: number, color: string) => {
  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';

  BONE_CONNECTIONS.forEach((conn) => {
    const fromJoint = joints[conn.from];
    const toJoint = joints[conn.to];

    if (fromJoint && toJoint) {
      ctx.beginPath();
      ctx.moveTo(centerX + fromJoint.x, GROUND_Y + fromJoint.y);
      ctx.lineTo(centerX + toJoint.x, GROUND_Y + toJoint.y);
      
      // 主骨架颜色
      ctx.strokeStyle = color;
      ctx.lineWidth = 4;
      ctx.stroke();

      // 内层高亮核心骨架线
      ctx.strokeStyle = '#ffffff';
      ctx.lineWidth = 1;
      ctx.stroke();
    }
  });

  // 绘制关节节点圈
  Object.keys(joints).forEach((key) => {
    const joint = joints[key];
    ctx.beginPath();
    ctx.arc(centerX + joint.x, GROUND_Y + joint.y, 4, 0, Math.PI * 2);
    ctx.fillStyle = '#1c202e';
    ctx.strokeStyle = color;
    ctx.lineWidth = 2;
    ctx.fill();
    ctx.stroke();
  });
};

// 整个沙盒场景绘制逻辑，采用 camelCase
const drawScene = () => {
  if (!isVisible.value) return;
  const canvas = canvasRef.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  // 清理背景
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // 1. 绘制网格背景 (编辑器感)
  if (showSkeletonDebug.value) {
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.02)';
    ctx.lineWidth = 1;
    for (let x = 0; x < canvas.width; x += 20) {
      ctx.beginPath();
      ctx.moveTo(x, 0);
      ctx.lineTo(x, canvas.height);
      ctx.stroke();
    }
    for (let y = 0; y < canvas.height; y += 20) {
      ctx.beginPath();
      ctx.moveTo(0, y);
      ctx.lineTo(canvas.width, y);
      ctx.stroke();
    }
  }

  // 2. 绘制中轴线（镜像平面）
  ctx.strokeStyle = 'var(--primaryColor)';
  ctx.lineWidth = 2;
  ctx.setLineDash([6, 4]);
  ctx.beginPath();
  ctx.moveTo(canvas.width / 2, 20);
  ctx.lineTo(canvas.width / 2, canvas.height - 20);
  ctx.stroke();
  ctx.setLineDash([]); // 重置

  // 镜像平面文字标识
  ctx.fillStyle = 'var(--primaryColor)';
  ctx.font = '10px JetBrains Mono';
  ctx.textAlign = 'center';
  ctx.fillText('MIRROR_PLANE (YZ)', canvas.width / 2, 15);

  // 3. 绘制地面线
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.1)';
  ctx.lineWidth = 1;
  ctx.beginPath();
  ctx.moveTo(10, GROUND_Y);
  ctx.lineTo(canvas.width - 10, GROUND_Y);
  ctx.stroke();

  // 4. 计算当前帧动画
  if (isPlaying.value) {
    time += 0.03 * playbackSpeed.value;
  }

  const originalJoints = computeOriginalJoints(time);
  const mirroredJoints = computeMirroredJoints(originalJoints);

  // 5. 绘制原骨骼小人 (左侧)
  drawSkeleton(ctx, originalJoints, ORIGINAL_CENTER_X, 'var(--secondaryColor)');
  
  // 6. 绘制镜像骨骼小人 (右侧)
  const mirrorColor = activeAlgorithm.value === 'boneMapping' ? 'var(--primaryColor)' : '#e0aaff';
  drawSkeleton(ctx, mirroredJoints, MIRROR_CENTER_X, mirrorColor);

  // 7. 绘制左右侧标识文字
  ctx.font = '12px JetBrains Mono';
  ctx.textAlign = 'center';
  ctx.fillStyle = 'rgba(255, 255, 255, 0.4)';
  ctx.fillText('ORIGINAL (SOURCE)', ORIGINAL_CENTER_X, GROUND_Y + 50);
  
  ctx.fillStyle = mirrorColor;
  const algoLabel = activeAlgorithm.value === 'boneMapping' ? 'BONE_MAPPING (SYMMETRIC)' : 'DIRECT_GEOMETRY_FLIP';
  ctx.fillText(`MIRRORED (${algoLabel})`, MIRROR_CENTER_X, GROUND_Y + 50);

  animationFrameId = requestAnimationFrame(drawScene);
};

// 状态变动监听，采用 camelCase
watch([activeAnimation, activeAlgorithm, showSkeletonDebug], () => {
  // 如果当前是暂停状态，变动后重新刷一帧
  if (!isPlaying.value) {
    drawScene();
  }
});

// 播放/暂停控制，采用 camelCase
const togglePlayback = () => {
  isPlaying.value = !isPlaying.value;
};

// 切换动画类型，采用 camelCase
const selectAnimation = (type: 'walk' | 'wave') => {
  activeAnimation.value = type;
};

// 切换镜像算法，采用 camelCase
const selectAlgorithm = (algo: 'direct' | 'boneMapping') => {
  activeAlgorithm.value = algo;
};

onMounted(() => {
  observer = new IntersectionObserver((entries) => {
    entries.forEach((entry) => {
      const wasVisible = isVisible.value;
      isVisible.value = entry.isIntersecting;
      if (entry.isIntersecting && !wasVisible) {
        drawScene();
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
  <section class="AnimationMirrorSection ScrollReveal" id="AnimationMirror">
    <div class="SectionHeader">
      <h2 class="SectionTitle">核心作品展示: AnimationMirror</h2>
      <p class="SectionSubtitle">
        支持多种动画镜像算法，极速为通用骨骼（Generic Avatar）生成完美的左/右镜像动画剪辑资源。
      </p>
    </div>

    <!-- 模拟 Unity 编辑器容器，采用 PascalCase -->
    <div class="UnityEditorSandbox GlassCard">
      <!-- 编辑器顶部工具栏 -->
      <div class="EditorToolbar">
        <div class="TabGroup">
          <div class="EditorTab ActiveTab">
            <svg class="TabIcon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <polygon points="5 3 19 12 5 21 5 3" />
            </svg>
            <span>AnimationMirror_Sandbox.unity</span>
          </div>
        </div>
        <div class="PlaybackControls">
          <!-- 播放暂停按钮 -->
          <button class="ToolbarButton" @click="togglePlayback" :title="isPlaying ? '暂停' : '播放'">
            <svg v-if="isPlaying" viewBox="0 0 24 24" fill="currentColor" class="CtrlIcon">
              <rect x="6" y="4" width="4" height="16" />
              <rect x="14" y="4" width="4" height="16" />
            </svg>
            <svg v-else viewBox="0 0 24 24" fill="currentColor" class="CtrlIcon">
              <polygon points="8 5 5 5 5 19 8 19 8 5" style="display: none;" />
              <polygon points="6 4 20 12 6 20 6 4" />
            </svg>
          </button>
          <span class="ToolbarDivider"></span>
          <span class="SpeedIndicator">Speed: {{ playbackSpeed.toFixed(1) }}x</span>
        </div>
      </div>

      <!-- 编辑器工作区布局：左侧视口，右侧 Inspector -->
      <div class="EditorWorkspace">
        <div class="ViewportPanel">
          <canvas 
            ref="canvasRef" 
            :width="VIEWPORT_WIDTH" 
            :height="VIEWPORT_HEIGHT" 
            class="ViewportCanvas"
          ></canvas>
        </div>

        <!-- Unity Inspector 面板 -->
        <div class="InspectorPanel">
          <div class="InspectorHeader">
            <h3>Inspector</h3>
          </div>

          <div class="InspectorContent">
            <!-- 属性组件：动画选择 -->
            <div class="InspectorGroup">
              <label class="GroupLabel">Source Animation</label>
              <div class="ButtonGrid">
                <button 
                  :class="['SelectBtn', { 'ActiveBtn': activeAnimation === 'walk' }]" 
                  @click="selectAnimation('walk')"
                >
                  Walking Cycle (Symmetric)
                </button>
                <button 
                  :class="['SelectBtn', { 'ActiveBtn': activeAnimation === 'wave' }]" 
                  @click="selectAnimation('wave')"
                >
                  Waving Hand (Asymmetric)
                </button>
              </div>
            </div>

            <!-- 属性组件：算法选择 -->
            <div class="InspectorGroup">
              <label class="GroupLabel">Mirror Algorithm</label>
              <select 
                class="InspectorSelect" 
                v-model="activeAlgorithm" 
                @change="selectAlgorithm(activeAlgorithm)"
              >
                <option value="boneMapping">Symmetric Bone Mapping (Recommended)</option>
                <option value="direct">Direct X-Axis Coordinate Flip</option>
              </select>
            </div>

            <!-- 属性组件：骨架和播放参数 -->
            <div class="InspectorGroup">
              <label class="GroupLabel">Debug Settings</label>
              <div class="CheckboxRow">
                <input type="checkbox" id="showDebugLines" v-model="showSkeletonDebug" />
                <label for="showDebugLines">Show Skeleton Grid</label>
              </div>
              
              <div class="SliderRow">
                <label>Playback Speed</label>
                <input 
                  type="range" 
                  min="0.1" 
                  max="2.0" 
                  step="0.1" 
                  v-model.number="playbackSpeed" 
                  class="SpeedRange" 
                />
              </div>
            </div>

            <!-- 核心数据对齐列表（体现功能细节） -->
            <div class="InspectorGroup">
              <label class="GroupLabel">Active Bone Symmetry Mappings</label>
              <div class="MappingTable">
                <div class="TableHeader">
                  <span>Source Joint</span>
                  <span></span>
                  <span>Mirrored Joint</span>
                </div>
                <div class="TableBody">
                  <div 
                    v-for="(map, index) in BONE_MAPPINGS" 
                    :key="index"
                    class="TableRow"
                  >
                    <span class="JointName">{{ map.src }}</span>
                    <span class="MappingArrow">⇄</span>
                    <span class="JointName">{{ map.dst }}</span>
                  </div>
                </div>
              </div>
            </div>

            <div class="AssetDescription">
              <div class="Badge">Unity Asset</div>
              <p>
                <b>AnimationMirror</b> 自动分析骨骼拓扑结构并对齐命名。支持 .anim 的关节 Rotation 与 Position 数据转换。彻底解决手动制作镜像动画的繁琐过程。
              </p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
.AnimationMirrorSection {
  display: flex;
  flex-direction: column;
  gap: 40px;
  padding: 40px 0;
}

.SectionHeader {
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
}

.SectionTitle {
  font-size: 32px;
  font-weight: 700;
  letter-spacing: -0.02em;
}

.SectionSubtitle {
  font-size: 16px;
  color: var(--textColor);
  max-width: 600px;
  line-height: 1.6;
}

/* Unity 风格编辑器容器 */
.UnityEditorSandbox {
  display: flex;
  flex-direction: column;
  width: 100%;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 30px 60px rgba(0, 0, 0, 0.6);
  border: 1px solid rgba(255, 255, 255, 0.05);
}

/* 顶部工具栏 */
.EditorToolbar {
  background-color: #1a1c23;
  border-bottom: 1px solid #101216;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 16px;
}

.TabGroup {
  display: flex;
  height: 100%;
}

.EditorTab {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 0 16px;
  font-size: 12px;
  font-family: var(--fontMono);
  color: var(--textColor);
  height: 100%;
  background-color: #232834;
  border-right: 1px solid #101216;
  border-top: 2px solid transparent;
}

.ActiveTab {
  background-color: #2b313f;
  color: var(--textHighlightColor);
  border-top: 2px solid var(--primaryColor);
}

.TabIcon {
  width: 12px;
  height: 12px;
  fill: currentColor;
}

.PlaybackControls {
  display: flex;
  align-items: center;
  gap: 12px;
}

.ToolbarButton {
  background: transparent;
  border: none;
  color: var(--textColor);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border-radius: 4px;
  transition: all 0.2s ease;
}

.ToolbarButton:hover {
  background-color: rgba(255, 255, 255, 0.08);
  color: var(--textHighlightColor);
}

.CtrlIcon {
  width: 14px;
  height: 14px;
}

.ToolbarDivider {
  width: 1px;
  height: 16px;
  background-color: rgba(255, 255, 255, 0.1);
}

.SpeedIndicator {
  font-size: 11px;
  font-family: var(--fontMono);
  color: var(--textColor);
  opacity: 0.8;
}

/* 工作区区分布局 */
.EditorWorkspace {
  display: flex;
  width: 100%;
  background-color: #20242f;
  min-height: 320px;
}

.ViewportPanel {
  flex: 7;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #171a22;
  position: relative;
}

.ViewportCanvas {
  max-width: 100%;
  display: block;
  background-color: #15181f;
}

.InspectorPanel {
  flex: 3;
  border-left: 1px solid #101216;
  background-color: #2b313f;
  display: flex;
  flex-direction: column;
  max-height: 480px;
  overflow-y: auto;
}

.InspectorHeader {
  background-color: #232834;
  border-bottom: 1px solid #101216;
  padding: 10px 16px;
}

.InspectorHeader h3 {
  font-size: 14px;
  font-weight: 600;
  letter-spacing: 0.05em;
  color: var(--textColor);
  text-transform: uppercase;
}

.InspectorContent {
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.InspectorGroup {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.GroupLabel {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: var(--textColor);
  opacity: 0.7;
}

.ButtonGrid {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.SelectBtn {
  background-color: #1f232d;
  border: 1px solid rgba(255, 255, 255, 0.05);
  color: var(--textColor);
  font-size: 12px;
  padding: 8px 12px;
  border-radius: 6px;
  text-align: left;
  cursor: pointer;
  transition: all 0.2s ease;
}

.SelectBtn:hover {
  background-color: #262c39;
  color: var(--textHighlightColor);
}

.ActiveBtn {
  background-color: var(--primaryColorGlow);
  border-color: var(--primaryColor);
  color: var(--primaryColor);
}

.InspectorSelect {
  background-color: #1f232d;
  border: 1px solid rgba(255, 255, 255, 0.05);
  color: var(--textHighlightColor);
  font-size: 12px;
  padding: 8px 12px;
  border-radius: 6px;
  cursor: pointer;
  outline: none;
}

.CheckboxRow {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 12px;
  color: var(--textHighlightColor);
  cursor: pointer;
}

.CheckboxRow input {
  cursor: pointer;
}

.SliderRow {
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-size: 12px;
  color: var(--textHighlightColor);
  margin-top: 6px;
}

.SpeedRange {
  width: 100%;
  accent-color: var(--primaryColor);
  background: #1f232d;
  height: 4px;
  border-radius: 2px;
  outline: none;
}

/* 映射表格 */
.MappingTable {
  border: 1px solid rgba(255, 255, 255, 0.05);
  border-radius: 6px;
  overflow: hidden;
  background-color: #1f232d;
}

.TableHeader {
  display: flex;
  justify-content: space-between;
  background-color: #181b24;
  padding: 6px 12px;
  font-size: 10px;
  font-weight: 700;
  text-transform: uppercase;
  color: var(--textColor);
  opacity: 0.6;
}

.TableBody {
  display: flex;
  flex-direction: column;
  max-height: 120px;
  overflow-y: auto;
}

.TableRow {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 12px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.02);
  font-size: 11px;
  font-family: var(--fontMono);
}

.JointName {
  color: #c5c9db;
}

.MappingArrow {
  color: var(--primaryColor);
  font-weight: bold;
}

.AssetDescription {
  margin-top: 10px;
  background-color: rgba(245, 128, 38, 0.04);
  border: 1px dashed rgba(245, 128, 38, 0.2);
  border-radius: 6px;
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.AssetDescription .Badge {
  align-self: flex-start;
  background-color: var(--primaryColor);
  color: #ffffff;
  font-size: 9px;
  font-weight: 700;
  text-transform: uppercase;
  padding: 2px 6px;
  border-radius: 4px;
}

.AssetDescription p {
  font-size: 11px;
  line-height: 1.5;
  color: var(--textColor);
}

/* 适配移动端 */
@media (max-width: 1024px) {
  .EditorWorkspace {
    flex-direction: column;
  }
  .ViewportPanel {
    height: 320px;
  }
  .InspectorPanel {
    border-left: none;
    border-top: 1px solid #101216;
    max-height: none;
  }
  .SectionTitle {
    font-size: 26px;
  }
  .SectionSubtitle {
    font-size: 14px;
  }
}
</style>
