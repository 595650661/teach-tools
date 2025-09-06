<template>
  <div class="wheel-wrapper">
    <div style="display: flex; flex-direction: column; width: 400px">
      <div>
        <h2>上传Excel文件</h2>
        <el-upload
          ref="uploadRef"
          class="upload-demo"
          :auto-upload="false"
          :show-file-list="true"
          :on-change="handleFileChange"
          :on-remove="handleFileRemove"
          :on-exceed="handleExceed"
          :limit="1"
          accept=".xlsx, .xls"
          :file-list="fileList"
        >
          <el-button type="primary">选择文件</el-button>
          <template #tip>
            <div class="el-upload__tip">请上传 xlsx/xls 格式的 Excel 文件，且只包含一列数据。</div>
          </template>
        </el-upload>
      </div>

      <div v-if="excelData.length">
        <h3>解析结果</h3>
        <div class="data-display">
          <el-select
            v-model="selectedColumn"
            placeholder="请选择数据列"
            style="margin-bottom: 10px"
          >
            <el-option
              v-for="(header, index) in excelHeaders"
              :key="index"
              :label="header"
              :value="index"
            />
          </el-select>
          <p>数组长度：{{ selectedColumnData.length }}</p>
          <pre>{{ selectedColumnData }}</pre>
        </div>
        <el-button type="primary" @click="importData" style="margin-top: 10px"
          >导入数据源</el-button
        >
      </div>
    </div>
    <div style="margin-left: 200px">
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
      <div class="winner">恭喜你抽中：{{ winner !== null ? prizes[winner] : '' }}</div>
      <div>(将要被抽中的是：{{ willBeRemoved }})</div>
      <el-button
        @click="removePrize"
        type="danger"
        :disabled="winner === null || prizes.length <= 2"
      >
        移除
      </el-button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, nextTick, type Ref, computed } from 'vue';
import POINTER from '@/assets/images/pointer.png';
import POINTER_CLICK from '@/assets/images/click-pointer.png';
// import WHEEL_BG from '@/assets/images/turnplate-bg.png'
import { genFileId, ElMessage } from 'element-plus';
import type { UploadInstance, UploadProps, UploadRawFile } from 'element-plus';
import * as XLSX from 'xlsx';

import WOOD_START from '@/assets/sounds/wood_start.wav';
import WOOD_END from '@/assets/sounds/wood_end.wav';
import WOOD_SPINNING from '@/assets/sounds/wood_click.wav';

// 创建音频对象
const startSound = new Audio(WOOD_START);
const endSound = new Audio(WOOD_END);
const spinningSound = new Audio(WOOD_SPINNING);

// 音效控制参数
const soundSettings = {
  spinningSoundInterval: 100, // 旋转音效播放间隔（毫秒）
  lastSpinningSoundTime: 0, // 上次播放旋转音效的时间
  spinningVolume: 0.5, // 旋转音效音量
  startVolume: 0.7, // 开始音效音量
  endVolume: 0.7, // 结束音效音量
};

// 预加载音频
startSound.preload = 'auto';
endSound.preload = 'auto';
spinningSound.preload = 'auto';

// 设置音量
startSound.volume = soundSettings.startVolume;
endSound.volume = soundSettings.endVolume;
spinningSound.volume = soundSettings.spinningVolume;

// 创建响应式引用
const startSoundRef = ref<HTMLAudioElement | null>(startSound);
const endSoundRef = ref<HTMLAudioElement | null>(endSound);
const spinningSoundRef = ref<HTMLAudioElement | null>(spinningSound);

// 存储解析后的 Excel 数据
const excelData = ref<any[][]>([]);
// 存储 Excel 文件的表头
const excelHeaders = ref<any[]>([]);
// 存储选中的列索引
const selectedColumn = ref(0);
// 存储上传的文件列表
const fileList = ref([]);
// el-upload组件引用
const uploadRef = ref<UploadInstance>();

// 缓存选中列的数据以提高性能
const selectedColumnData = computed(() => {
  if (excelData.value.length === 0 || excelHeaders.value.length === 0) {
    return [];
  }
  const selectedColumnIndex = selectedColumn.value;
  // 安全地获取选中列的数据，避免数组越界
  return excelData.value.map((row) => {
    return row.length > selectedColumnIndex ? row[selectedColumnIndex] : undefined;
  }).filter((item) => item !== undefined);
});

/**
 * 处理文件选择和数据解析
 * @param {object} file - el-upload 传递的文件对象
 */
const handleFileChange = (file: { raw: File; status: string }) => {
  if (file.status === 'ready') {
    // 使用 FileReader 读取文件内容
    const reader = new FileReader();

    // 读取成功后的回调
    reader.onload = (e) => {
      try {
        if (!e.target) {
          throw new Error('FileReader event target is null');
        }
        const data = e.target.result;
        // 将文件数据转换为二进制字符串
        const workbook = XLSX.read(data, { type: 'binary' });

        // 获取第一个工作表的名称
        const sheetName = workbook.SheetNames[0];
        // 获取第一个工作表
        const worksheet = workbook.Sheets[sheetName];

        // 将工作表数据转换为 JSON 数组
        // header: 1 表示将第一行作为表头（键名），不处理。我们只取值
        const jsonSheet: any[][] = XLSX.utils.sheet_to_json(worksheet, { header: 1 });
        // 提取表头（第一行）
        const headers = (jsonSheet[0] as any[]) || [];
        // 提取数据（排除表头）
        const rows = jsonSheet.slice(1);

        // 过滤掉表头为null或undefined的列（但保留空字符串）
        const validColumns = headers
          .map((header, index) => {
            return header !== null && header !== undefined ? index : -1;
          })
          .filter((index) => index !== -1);

        // 只保留有效列的数据
        const filteredRows = rows.map((row) => {
          return validColumns.map((index) => {
            // 保留所有值，包括空字符串，只过滤undefined和null
            const value = row[index];
            return (value !== undefined && value !== null) ? value : '';
          });
        });

        // 存储解析后的数据
        excelData.value = filteredRows;
        // 存储过滤后的表头（保留空字符串）
        excelHeaders.value = validColumns.map(index => headers[index]);

        selectedColumn.value = 0; // 重置选中的列索引

        ElMessage.success('Excel 文件解析成功！');
      } catch (error) {
        ElMessage.error('文件解析失败，请确保文件格式正确。');
        console.error('解析错误:', error);
      }
    };

    // 开始读取文件
    reader.readAsBinaryString(file.raw);
  }
};

/**
 * 处理文件移除
 */
const handleFileRemove = async () => {
  // 清空文件列表
  fileList.value = [];
  // 清空数据
  excelData.value = [];
  excelHeaders.value = [];
  selectedColumn.value = 0;

  // 强制刷新el-upload组件
  await nextTick();
  // 触发DOM更新
  uploadRef.value?.clearFiles();

  ElMessage.info('文件已移除，数据已清空。');
};

const handleExceed: UploadProps['onExceed'] = (files) => {
  uploadRef.value!.clearFiles();
  const file = files[0] as UploadRawFile;
  file.uid = genFileId();
  uploadRef.value!.handleStart(file);
};

const prizes = ref<string[]>([
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
]);
const importData = () => {
  // 先获取当前选中的列数据
  const selectedData = [...selectedColumnData.value];

  // 更新奖项数据
  prizes.value = selectedData;
  ElMessage.success('数据源已更新，转盘奖项已刷新！');
  // 重置winner
  winner.value = null;

  // 重新计算角度指向第一个奖项中心
  if (prizes.value.length > 0) {
    const sectorAngle = 360 / prizes.value.length;
    currentAngle.value = 270 - sectorAngle / 2;
  }

  // 更新转盘
  updateWheel();
};

// 获取选中的列数据
// 已通过计算属性selectedColumnData替代
const willBeRemoved = ref<string | null>(null);

// 画布的宽度（单位：像素）
const CANVAS_WIDTH = 400;
// 画布的高度（单位：像素）
const CANVAS_HEIGHT = 400;
// 画布中心的 X 坐标
const CENTER_X = CANVAS_WIDTH * 0.5;
// 画布中心的 Y 坐标
const CENTER_Y = CANVAS_HEIGHT * 0.5;
// 转盘的半径（留出 20 像素的边距）
const WHEEL_RADIUS = CANVAS_WIDTH * 0.5 - 30;
// 扇形区域的交替颜色
const SECTOR_COLORS = ['#ffecb3', '#ffe0b2'];
// 文字距离转盘中心的半径
const TEXT_RADIUS = WHEEL_RADIUS * 0.75;
// 中心圆的半径
const CENTER_CIRCLE_RADIUS = 30;

// 状态定义
const wheelCanvas = ref<HTMLCanvasElement | null>(null);
const isSpinning = ref(false);
const isReverseRotation = ref(true); // 控制旋转方向，false为正向，true为逆向
const winner = ref<number | null>(null);
const currentAngle = ref<number>(270 - 360 / prizes.value.length / 2); // 动态计算初始角度，确保指针指向第一个奖项中心
let animationFrameId: number | null = null;

function getCanvasContext() {
  const canvas = wheelCanvas.value;
  if (!canvas) {
    console.error('Canvas element is not available.');
    return null;
  }

  const ctx = canvas.getContext('2d');
  if (!ctx) {
    console.error('Failed to get 2D rendering context.');
    return null;
  }

  return ctx;
}

/**
 * 绘制转盘
 * 1. 清除画布
 * 2. 绘制扇形区域（交替使用两种颜色）
 * 3. 在每个扇形区域中心绘制奖项文字
 * 4. 绘制中心圆
 */
function drawWheel() {
  const ctx = getCanvasContext();
  if (!ctx) return;

  const num = prizes.value.length;
  const angle = (2 * Math.PI) / num;

  // 清除画布
  ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  // 绘制扇形区域（从右侧开始，顺时针）
  for (let i = 0; i < num; i++) {
    ctx.beginPath();
    ctx.moveTo(CENTER_X, CENTER_Y);
    ctx.arc(CENTER_X, CENTER_Y, WHEEL_RADIUS, i * angle, (i + 1) * angle);
    ctx.closePath();
    ctx.fillStyle = SECTOR_COLORS[i % 2];
    ctx.fill();
    ctx.stroke();

    // 绘制文字
    ctx.save();
    ctx.translate(CENTER_X, CENTER_Y);
    ctx.rotate(i * angle + angle / 2);
    ctx.textAlign = 'right';
    ctx.fillStyle = '#333';
    ctx.font = 'bold 14px Arial';
    ctx.fillText(prizes.value[i], TEXT_RADIUS, 10);
    ctx.restore();
  }

  // 绘制中心圆
  ctx.beginPath();
  ctx.arc(CENTER_X, CENTER_Y, CENTER_CIRCLE_RADIUS, 0, 2 * Math.PI);
  ctx.fillStyle = '#fff';
  ctx.fill();
  ctx.stroke();
}

function updateWheel() {
  const ctx = getCanvasContext();
  if (!ctx) return;

  ctx.save();
  ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
  ctx.translate(CENTER_X, CENTER_Y);
  ctx.rotate((currentAngle.value * Math.PI) / 180);
  ctx.translate(-CENTER_X, -CENTER_Y);
  drawWheel();
  ctx.restore();
}

/**
 * 生成随机索引
 * @param max - 最大值（不包含）
 * @returns 随机索引值
 */
function getRandomIndex(max: number): number {
  if (typeof window !== 'undefined' && window.crypto && window.crypto.getRandomValues) {
    const array = new Uint32Array(1);
    window.crypto.getRandomValues(array);
    return array[0] % max;
  } else {
    // 回退到Math.random()
    return Math.floor(Math.random() * max);
  }
}

// 播放音效函数 - 改进版本
const playSound = (sound: Ref<HTMLAudioElement | null>, forceRestart = false) => {
  if (!sound.value) return;

  try {
    // 如果强制重新播放或者音频已经播放完毕，则从头开始播放
    if (forceRestart || sound.value.currentTime >= sound.value.duration) {
      sound.value.currentTime = 0;
    }

    // 播放音频
    const playPromise = sound.value.play();

    if (playPromise !== undefined) {
      playPromise.catch((err) => {
        console.error('播放音效失败:', err);
        // 处理自动播放被阻止的情况
        if (err.name === 'NotAllowedError') {
          console.warn('自动播放被阻止，需要用户先与页面交互');
        }
      });
    }
  } catch (err) {
    console.error('播放音效异常:', err);
  }
};

// 播放旋转音效 - 专门用于旋转音效的函数
const playSpinningSound = () => {
  const now = Date.now();
  // 控制旋转音效播放频率，避免过于频繁播放
  if (now - soundSettings.lastSpinningSoundTime > soundSettings.spinningSoundInterval) {
    soundSettings.lastSpinningSoundTime = now;
    playSound(spinningSoundRef, true);
  }
};
/**
 * 启动转盘旋转动画
 * 1. 生成随机索引作为中奖结果
 * 2. 计算目标旋转角度（包含额外圈数）
 * 3. 使用缓动函数实现平滑动画
 * 4. 更新转盘角度并触发重绘
 */
function clickSpin() {
  if (isSpinning.value) return;

  // 添加短暂延迟后再禁用按钮，避免用户快速点击导致动画冲突
  setTimeout(() => {
    isSpinning.value = true;
  }, 100);

  playSound(startSoundRef);

  winner.value = null;

  // 使用工具函数生成随机索引
  const num = prizes.value.length;
  const randomIndex = getRandomIndex(num);

  // 在控制台提前输出将要被抽中的选项
  willBeRemoved.value = prizes.value[randomIndex];
  console.log('即将抽中的奖项:', willBeRemoved.value);

  // 计算需要旋转的角度
  const sectorAngle = 360 / num;
  // 每个奖项的中心角度
  const prizeCenterAngle = sectorAngle * randomIndex + sectorAngle / 2;
  // 指针在顶部(270度方向)
  let targetAngle = 270 - prizeCenterAngle;

  // 规范化角度
  while (targetAngle < 0) {
    targetAngle += 360;
  }

  // 加上额外的旋转圈数（5圈）增加视觉效果
  const extraRotation = 5 * 360;
  // 最终旋转角度
  const finalRotation = extraRotation + targetAngle - (currentAngle.value % 360);

  const startTime = performance.now();
  const duration = 4000; // 4秒动画

  const startAngle = currentAngle.value;

  // 预计算缓动函数并缓存
  const easeOutCubic = (t: number) => 1 - Math.pow(1 - t, 3);
  const calculateRotation = (progress: number) =>
    startAngle + easeOutCubic(progress) * finalRotation;

  /**
   * 动画帧回调函数
   * @param timestamp - 当前时间戳
   */
  const animate = (timestamp: number) => {
    const elapsed = timestamp - startTime;
    const progress = Math.min(elapsed / duration, 1);

    currentAngle.value = calculateRotation(progress);
    updateWheel();

    // 基于速度的动态音效播放（优化渐弱效果）
    if (progress < 1) {
      // 计算当前速度（基于缓动函数的导数）
      const currentSpeed = (3 * Math.pow(1 - progress, 2) * finalRotation) / duration;

      // 速度阈值触发音效（快速时更密集，慢速时更稀疏）
      if (currentSpeed > 0.1) {
        // 根据速度调整播放间隔 - 速度越快播放越频繁
        const speedFactor = Math.min(1, currentSpeed / 2);
        if (Math.random() < speedFactor) {
          playSpinningSound();
        }
      }
    }

    if (progress < 1) {
      animationFrameId = requestAnimationFrame(animate);
    } else {
      isSpinning.value = false;
      winner.value = randomIndex;

      // 播放结束音效
      playSound(endSoundRef);

      // 不再移除奖项，只更新转盘显示
      updateWheel();
    }
  };

  animationFrameId = requestAnimationFrame(animate);
}

/**
 * 移除当前中奖奖项
 */
function removePrize() {
  if (winner.value === null) return;

  // 移除奖项
  prizes.value.splice(winner.value, 1);

  // 重置winner
  winner.value = null;

  // 重新计算角度指向第一个奖项中心
  if (prizes.value.length > 0) {
    const sectorAngle = 360 / prizes.value.length;
    currentAngle.value = 270 - sectorAngle / 2;
  }

  // 更新转盘
  updateWheel();
}

onMounted(() => {
  // 初始绘制时确保转盘正确显示
  updateWheel();
});

onBeforeUnmount(() => {
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
});
</script>

<style scoped lang="scss">
.wheel-wrapper {
  display: flex;
  align-items: center;
  padding: 100px;
  width: 100%;
  height: 100%;
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

/* 中心指针样式 */
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

button {
  margin-top: 10px;
  padding: 10px 20px;
  font-size: 16px;
  color: white;
  border: none;
  border-radius: 4px;
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

.upload-demo {
  margin-bottom: 20px;
}
.data-display {
  background-color: #f5f5f5;
  padding: 15px;
  border-radius: 8px;
  max-height: 400px;
  overflow-y: auto;
}
pre {
  white-space: pre-wrap;
  word-wrap: break-word;
}
</style>
