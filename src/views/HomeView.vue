<template>
  <div class="wheel-wrapper">
    <div class="wheel-container">
      <!-- 添加旋转背景图 -->
      <div
        class="wheel-bg"
        :style="{ '--rotate-direction': isReverseRotation ? 'reverse' : 'normal' }"
      ></div>
      <canvas ref="wheelCanvas" width="400" height="400" style="background: none"></canvas>
      <!-- 中心指针 -->
      <img
        :src="isSpinning ? POINTER : POINTER_CLICK"
        class="pointer"
        @click="clickSpin"
        :style="{ cursor: isSpinning ? 'default' : 'pointer' }"
      />
    </div>
    <div class="winner">恭喜你抽中：{{ winner ? prizes[winner] : '' }}</div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'
import POINTER from '@/assets/images/pointer.png'
import POINTER_CLICK from '@/assets/images/click-pointer.png'
// import WHEEL_BG from '@/assets/images/turnplate-bg.png'
const prizes = [
  '一等奖',
  '二等奖',
  '三等奖',
  '四等奖',
  '五等奖',
  '六等奖',
  '七等奖',
  '八等奖',
  '九等奖',
  '十等奖',
  '谢谢参与',
]

// 常量定义
// 画布的宽度（单位：像素）
const CANVAS_WIDTH = 400
// 画布的高度（单位：像素）
const CANVAS_HEIGHT = 400
// 画布中心的 X 坐标
const CENTER_X = CANVAS_WIDTH * 0.5
// 画布中心的 Y 坐标
const CENTER_Y = CANVAS_HEIGHT * 0.5
// 转盘的半径（留出 20 像素的边距）
const WHEEL_RADIUS = CANVAS_WIDTH * 0.5 - 30
// 扇形区域的交替颜色
const SECTOR_COLORS = ['#ffecb3', '#ffe0b2']
// 文字距离转盘中心的半径
const TEXT_RADIUS = WHEEL_RADIUS * 0.75
// 中心圆的半径
const CENTER_CIRCLE_RADIUS = 30

// 状态定义
const wheelCanvas = ref<HTMLCanvasElement | null>(null)
const isSpinning = ref(false)
const isReverseRotation = ref(true) // 控制旋转方向，false为正向，true为逆向
const winner = ref<number | null>(null)
const currentAngle = ref<number>(270 - 360 / prizes.length / 2) // 动态计算初始角度，确保指针指向第一个奖项中心
let animationFrameId: number | null = null

function getCanvasContext() {
  const canvas = wheelCanvas.value
  if (!canvas) {
    console.error('Canvas element is not available.')
    return null
  }

  const ctx = canvas.getContext('2d')
  if (!ctx) {
    console.error('Failed to get 2D rendering context.')
    return null
  }

  return ctx
}

/**
 * 绘制转盘
 * 1. 清除画布
 * 2. 绘制扇形区域（交替使用两种颜色）
 * 3. 在每个扇形区域中心绘制奖项文字
 * 4. 绘制中心圆
 */
function drawWheel() {
  const ctx = getCanvasContext()
  if (!ctx) return

  const num = prizes.length
  const angle = (2 * Math.PI) / num

  // 清除画布
  ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT)

  // 绘制扇形区域（从右侧开始，顺时针）
  for (let i = 0; i < num; i++) {
    ctx.beginPath()
    ctx.moveTo(CENTER_X, CENTER_Y)
    ctx.arc(CENTER_X, CENTER_Y, WHEEL_RADIUS, i * angle, (i + 1) * angle)
    ctx.closePath()
    ctx.fillStyle = SECTOR_COLORS[i % 2]
    ctx.fill()
    ctx.stroke()

    // 绘制文字
    ctx.save()
    ctx.translate(CENTER_X, CENTER_Y)
    ctx.rotate(i * angle + angle / 2)
    ctx.textAlign = 'right'
    ctx.fillStyle = '#333'
    ctx.font = 'bold 14px Arial'
    ctx.fillText(prizes[i], TEXT_RADIUS, 10)
    ctx.restore()
  }

  // 绘制中心圆
  ctx.beginPath()
  ctx.arc(CENTER_X, CENTER_Y, CENTER_CIRCLE_RADIUS, 0, 2 * Math.PI)
  ctx.fillStyle = '#fff'
  ctx.fill()
  ctx.stroke()
}

function updateWheel() {
  const ctx = getCanvasContext()
  if (!ctx) return

  ctx.save()
  ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT)
  ctx.translate(CENTER_X, CENTER_Y)
  ctx.rotate((currentAngle.value * Math.PI) / 180)
  ctx.translate(-CENTER_X, -CENTER_Y)
  drawWheel()
  ctx.restore()
}

/**
 * 生成随机索引
 * @param max - 最大值（不包含）
 * @returns 随机索引值
 */
function getRandomIndex(max: number): number {
  if (typeof window !== 'undefined' && window.crypto && window.crypto.getRandomValues) {
    const array = new Uint32Array(1)
    window.crypto.getRandomValues(array)
    return array[0] % max
  } else {
    // 回退到Math.random()
    return Math.floor(Math.random() * max)
  }
}

/**
 * 启动转盘旋转动画
 * 1. 生成随机索引作为中奖结果
 * 2. 计算目标旋转角度（包含额外圈数）
 * 3. 使用缓动函数实现平滑动画
 * 4. 更新转盘角度并触发重绘
 */
function clickSpin() {
  if (isSpinning.value) return

  // 添加短暂延迟后再禁用按钮，避免用户快速点击导致动画冲突
  setTimeout(() => {
    isSpinning.value = true
  }, 100)

  winner.value = null

  // 使用工具函数生成随机索引
  const num = prizes.length
  const randomIndex = getRandomIndex(num)

  // 计算需要旋转的角度
  const sectorAngle = 360 / num
  // 每个奖项的中心角度
  const prizeCenterAngle = sectorAngle * randomIndex + sectorAngle / 2
  // 指针在顶部(270度方向)
  let targetAngle = 270 - prizeCenterAngle

  // 规范化角度
  while (targetAngle < 0) {
    targetAngle += 360
  }

  // 加上额外的旋转圈数（5圈）增加视觉效果
  const extraRotation = 5 * 360
  // 最终旋转角度
  const finalRotation = extraRotation + targetAngle - (currentAngle.value % 360)

  const startTime = performance.now()
  const duration = 4000 // 4秒动画

  const startAngle = currentAngle.value

  // 预计算缓动函数并缓存
  const easeOutCubic = (t: number) => 1 - Math.pow(1 - t, 3)
  const calculateRotation = (progress: number) =>
    startAngle + easeOutCubic(progress) * finalRotation

  /**
   * 动画帧回调函数
   * @param timestamp - 当前时间戳
   */
  const animate = (timestamp: number) => {
    const elapsed = timestamp - startTime
    const progress = Math.min(elapsed / duration, 1)

    currentAngle.value = calculateRotation(progress)
    updateWheel()

    if (progress < 1) {
      animationFrameId = requestAnimationFrame(animate)
    } else {
      isSpinning.value = false
      winner.value = randomIndex

      // 当剩余奖项大于2时才移除
      if (prizes.length > 2) {
        prizes.splice(randomIndex, 1)

        // 重新计算角度指向第一个奖项中心
        const sectorAngle = 360 / prizes.length
        currentAngle.value = 270 - sectorAngle / 2
      }

      // 立即更新转盘
      updateWheel()
    }
  }

  animationFrameId = requestAnimationFrame(animate)
}

onMounted(() => {
  // 初始绘制时确保转盘正确显示
  updateWheel()
})

onBeforeUnmount(() => {
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId)
  }
})
</script>

<style scoped>
.wheel-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
}
/* 添加旋转动画 */
@keyframes rotate {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}
/* 新增的背景元素 */
.wheel-bg {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: url('@/assets/images/turnplate-bg2.png');
  background-size: contain;
  background-repeat: no-repeat;
  background-position: center;
  animation: rotate 8s linear infinite;
  animation-direction: var(--rotate-direction, normal);
  z-index: -1; /* 确保背景在转盘下方 */
}

.wheel-container {
  position: relative;
  margin: 20px 0;
}

canvas {
  background: #fffbe6;
  border-radius: 50%;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

/* 中心指针样式 - 锚点圆在中心，箭头指向外围 */
.pointer {
  position: absolute;
  top: calc(50% - 18px);
  left: 50%;
  transform: translate(-50%, -50%) rotate(0deg);
  width: 100px;
  height: 100px;
  z-index: 10;
  object-fit: contain;
}

/* 中心的锚点圆 */
.pointer::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 30px;
  height: 30px;
  background-color: #f44336;
  border-radius: 50%;
}

/* 指向外围的箭头 */
.pointer::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -100%);
  width: 0;
  height: 0;
  border-left: 12px solid transparent;
  border-right: 12px solid transparent;
  border-bottom: 40px solid #f44336;
}

button {
  margin-top: 10px;
  padding: 10px 20px;
  font-size: 16px;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

button:hover:not(:disabled) {
  background-color: #45a049;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.winner {
  margin-top: 20px;
  padding: 10px 20px;
  background-color: #e8f5e9;
  border-radius: 4px;
  font-size: 18px;
  font-weight: bold;
  color: #2e7d32;
}
</style>
