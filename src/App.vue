<template>
  <div class="app-container" :class="{ 'dark': isDarkMode }">
    <!-- 背景動畫層 (置於最底層，不影響原有排版與組件) -->
    <div id="animated-bg">
      <!-- 白天雲朵層 -->
      <div class="day-layer">
        <div class="cloud cloud-1"></div>
        <div class="cloud cloud-2"></div>
        <div class="cloud cloud-3"></div>
        <div class="cloud cloud-4"></div>
        <div class="cloud cloud-5"></div>
        <div class="cloud cloud-6"></div>
      </div>
      <!-- 夜晚星空層 -->
      <div class="night-layer">
        <div v-for="(star, index) in stars" :key="index" class="star" :style="star"></div>
        <!-- 偶爾劃過的流星 -->
        <div class="meteor meteor-1"></div>
        <div class="meteor meteor-2"></div>
        <div class="meteor meteor-3"></div>
      </div>
    </div>

    <!-- 頂部導航列 -->
    <header class="navbar">
      
      <!-- 頂端控制列 (時間與主題切換) -->
      <div class="top-controls">
        <div class="current-time">🇹🇼 台灣時間：{{ currentTime }}</div>
        <div class="right-controls">
          <!-- 白噪音選單 -->
          <div class="noise-container">
            <button class="theme-toggle" @click="showNoiseMenu = !showNoiseMenu">
              🎧 {{ currentNoise === 'none' ? '白噪音' : noiseNames[currentNoise] }}
            </button>
            <div class="noise-menu" v-show="showNoiseMenu">
              <button :class="{ active: currentNoise === 'none' }" @click="setNoise('none')">🔇 關閉</button>
              <button :class="{ active: currentNoise === 'forest' }" @click="setNoise('forest')">🌲 森林</button>
              <button :class="{ active: currentNoise === 'water' }" @click="setNoise('water')">🌊 流水</button>
              <button :class="{ active: currentNoise === 'rain' }" @click="setNoise('rain')">🌧️ 下雨</button>
              <button :class="{ active: currentNoise === 'space' }" @click="setNoise('space')">🌌 空曠</button>
              <!-- 音量控制 -->
              <div class="volume-control" v-show="currentNoise !== 'none'">
                <label>🔈 音量</label>
                <input type="range" min="0" max="100" v-model="noiseVolume" @input="updateVolume" />
              </div>
            </div>
          </div>
          <button class="theme-toggle" @click="toggleTheme">
            {{ isDarkMode ? '☀️ 白天模式' : '🌙 夜間模式' }}
          </button>
        </div>
      </div>

      <h1>🛠️ 課堂互動工具箱</h1>
      <nav class="tabs">
        <button 
          :class="{ active: currentTab === 'picker' }" 
          @click="currentTab = 'picker'"
          data-tooltip="快速抽出幸運兒或小組，支援動畫特效"
        >
          🎲 隨機抽籤機
        </button>
        <button 
          :class="{ active: currentTab === 'scoreboard' }" 
          @click="currentTab = 'scoreboard'"
          data-tooltip="動態競賽排名，支援快速加減分"
        >
          📊 小組記分板
        </button>
        <button 
          :class="{ active: currentTab === 'timer' }" 
          @click="currentTab = 'timer'"
          data-tooltip="提供倒數、正向計時與番茄鐘功能"
        >
          ⏱️ 報告計時器
        </button>
        <button 
          :class="{ active: currentTab === 'exam' }" 
          @click="currentTab = 'exam'"
          data-tooltip="發起課堂即時投票或互動測驗卷"
        >
          📝 數位測驗系統
        </button>
        <button 
          :class="{ active: currentTab === 'schedule' }" 
          @click="currentTab = 'schedule'"
          data-tooltip="自訂專屬的每週班級課表"
        >
          📅 電子課表
        </button>
      </nav>
    </header>

    <!-- 工具展示區 (使用 v-show 保持組件狀態，避免切換時名單遺失) -->
    <main class="content">
      <RandomPicker v-show="currentTab === 'picker'" />
      <Scoreboard v-show="currentTab === 'scoreboard'" />
      <Timer v-show="currentTab === 'timer'" />
      <ExamSystem v-show="currentTab === 'exam'" />
      <ClassSchedule v-show="currentTab === 'schedule'" />
    </main>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onUnmounted } from 'vue';
import RandomPicker from './components/RandomPicker.vue';
import Scoreboard from './components/Scoreboard.vue';
import Timer from './components/Timer.vue';
import ExamSystem from './components/ExamSystem.vue';
import ClassSchedule from './components/ClassSchedule.vue';

// 預設顯示報告計時器（可依需求修改）
const currentTab = ref('timer');

// 台灣時間狀態管理
const currentTime = ref('');
let timeInterval = null;

const updateTime = () => {
  const now = new Date();
  // 設定以台灣時區 (Asia/Taipei) 顯示 24 小時制時間
  currentTime.value = now.toLocaleString('zh-TW', { 
    timeZone: 'Asia/Taipei',
    hour12: false,
    year: 'numeric', month: '2-digit', day: '2-digit',
    hour: '2-digit', minute: '2-digit', second: '2-digit'
  });
};

// 日夜模式狀態管理
const isDarkMode = ref(false);
const toggleTheme = () => {
  isDarkMode.value = !isDarkMode.value;
  localStorage.setItem('toolbox-theme', isDarkMode.value ? 'dark' : 'light');
};

// 星空資料狀態 (用於夜間模式)
const stars = ref([]);

// --- 白噪音系統 (Web Audio API 程式化生成) ---
// 使用演算法合成聲音，不依賴外部音檔，可確保永遠不會有讀取錯誤或空白
const showNoiseMenu = ref(false);
const currentNoise = ref('none');
const noiseNames = {
  none: '關閉', forest: '🌲 森林', water: '🌊 流水', rain: '🌧️ 下雨', space: '🌌 空曠'
};
const noiseVolume = ref(50); // 預設音量 50%

let noiseCtx = null;
let noiseSource = null;
let noiseFilter = null;
let noiseGain = null;

// 配置不同環境的頻率與音量特性
const applyNoiseSettings = (type) => {
  if (!noiseFilter) return;
  if (type === 'rain') {
    noiseFilter.type = 'lowpass'; noiseFilter.frequency.value = 800; noiseFilter.Q.value = 0.5;
  } else if (type === 'water') {
    noiseFilter.type = 'bandpass'; noiseFilter.frequency.value = 400; noiseFilter.Q.value = 0.2;
  } else if (type === 'forest') {
    noiseFilter.type = 'bandpass'; noiseFilter.frequency.value = 600; noiseFilter.Q.value = 0.8; // 調整為更自然的森林環境音
  } else if (type === 'space') {
    noiseFilter.type = 'lowpass'; noiseFilter.frequency.value = 100; noiseFilter.Q.value = 1;
  }
};

const getVolumeFor = (type) => {
  let base = 0;
  if (type === 'rain') base = 0.8;
  if (type === 'water') base = 1.2; // 稍微調整流水音量
  if (type === 'forest') base = 0.7; // 調整森林音量
  if (type === 'space') base = 3.0; // 空曠低頻需要較大音量增益
  return base * (noiseVolume.value / 100);
};

const updateVolume = () => {
  if (noiseCtx && noiseGain && currentNoise.value !== 'none') {
    // 平滑過渡音量，避免拖拉時產生雜音或爆音
    noiseGain.gain.setValueAtTime(noiseGain.gain.value, noiseCtx.currentTime);
    noiseGain.gain.linearRampToValueAtTime(getVolumeFor(currentNoise.value), noiseCtx.currentTime + 0.1);
  }
};

const setNoise = (type) => {
  currentNoise.value = type;

  // 關閉聲音 (加上 1 秒平滑淡出，避免聲音突然斷掉產生爆音)
  if (type === 'none') {
    if (noiseGain) {
      noiseGain.gain.linearRampToValueAtTime(0, noiseCtx.currentTime + 1);
      setTimeout(() => {
        if (noiseSource && currentNoise.value === 'none') {
          noiseSource.stop(); noiseSource.disconnect(); noiseSource = null;
        }
      }, 1000);
    }
    return;
  }

  if (!noiseCtx) noiseCtx = new (window.AudioContext || window.webkitAudioContext)();
  if (noiseCtx.state === 'suspended') noiseCtx.resume();

  if (noiseSource) {
    // 如果已經在播放，則平滑過渡到新環境音
    applyNoiseSettings(type);
    noiseGain.gain.linearRampToValueAtTime(getVolumeFor(type), noiseCtx.currentTime + 1);
  } else {
    // 初始化白噪音雜訊緩衝區
    const bufferSize = noiseCtx.sampleRate * 2; // 2秒無縫循環
    const buffer = noiseCtx.createBuffer(1, bufferSize, noiseCtx.sampleRate);
    const data = buffer.getChannelData(0);
    for (let i = 0; i < bufferSize; i++) data[i] = Math.random() * 2 - 1;

    noiseSource = noiseCtx.createBufferSource();
    noiseSource.buffer = buffer; noiseSource.loop = true;
    noiseFilter = noiseCtx.createBiquadFilter();
    noiseGain = noiseCtx.createGain();

    noiseSource.connect(noiseFilter); noiseFilter.connect(noiseGain); noiseGain.connect(noiseCtx.destination);
    applyNoiseSettings(type);
    noiseGain.gain.setValueAtTime(0, noiseCtx.currentTime);
    noiseGain.gain.linearRampToValueAtTime(getVolumeFor(type), noiseCtx.currentTime + 1); // 漸入
    noiseSource.start();
  }
};

// 頁面載入時讀取上次停留的工具分頁
onMounted(() => {
  updateTime();
  timeInterval = setInterval(updateTime, 1000); // 每秒更新時間

  const savedTheme = localStorage.getItem('toolbox-theme');
  if (savedTheme === 'dark') isDarkMode.value = true;

  const savedTab = localStorage.getItem('toolbox-current-tab');
  if (savedTab) currentTab.value = savedTab;

  // 隨機生成星空資料 (120顆星星，對應稍微放大的背景空間)
  const generatedStars = [];
  for (let i = 0; i < 120; i++) {
    const size = Math.random() * 3 + 1; // 1px ~ 4px
    generatedStars.push({
      width: `${size}px`,
      height: `${size}px`,
      left: `${Math.random() * 130}vw`,
      top: `${Math.random() * 130}vh`,
      animationDuration: `${Math.random() * 2 + 1}s`,
      animationDelay: `${Math.random() * 2}s`
    });
  }
  stars.value = generatedStars;

  // 【自動化腳本】直接掃描並清除子組件標題中的「課堂互動工具箱 - 」前綴文字
  setTimeout(() => {
    const componentTitles = document.querySelectorAll('.content h1, .content h2, .content h3');
    componentTitles.forEach(title => {
      const text = title.textContent;
      if (text.includes('課堂互動工具箱 -')) {
        title.textContent = text.replace('課堂互動工具箱 -', '').trim();
      }
    });
  }, 100); // 延遲 100 毫秒確保所有工具的組件都已經成功掛載到畫面上
});

// 組件解除掛載時清理計時器
onUnmounted(() => {
  if (timeInterval) clearInterval(timeInterval);
  if (noiseCtx) {
    noiseCtx.close();
  }
});

// 當分頁切換時，儲存至 Local Storage，並保持網頁標題為「課堂互動工具箱」
watch(currentTab, (newTab) => {
  localStorage.setItem('toolbox-current-tab', newTab);
  document.title = '課堂互動工具箱';
}, { immediate: true });
</script>

<style scoped>
.app-container {
  font-family: 'Nunito', 'Noto Sans TC', 'PingFang TC', 'Apple LiGothic Medium', 'Microsoft JhengHei', sans-serif;
  font-weight: 600;
  letter-spacing: 0.5px;
  min-height: 100vh;
  background: transparent; /* 背景移交給獨立的動畫層處理 */
  transition: background 0.5s ease, color 0.5s ease;
}

/* =========================================
   背景動畫層設定 (日夜模式共用)
   ========================================= */
#animated-bg {
  position: fixed;
  /* 加大背景範圍並稍微往上偏移，防止手機版滑動回彈或網址列伸縮時出現黑色缺塊 */
  top: -15vh;
  left: -15vw;
  width: 130vw;
  height: 130vh;
  z-index: -1;
  overflow: hidden;
  background: linear-gradient(to bottom, #74ebd5, #ACB6E5); /* 白天天空 */
  transition: background 1.5s ease-in-out;
  /* 開啟硬體加速，防止部分手機瀏覽器滾動時閃爍 */
  transform: translateZ(0);
  -webkit-transform: translateZ(0);
}

.app-container.dark #animated-bg {
  background: linear-gradient(to bottom, #0f2027, #203a43, #2c5364); /* 夜晚星空 */
}

.day-layer, .night-layer {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  transition: opacity 1.5s ease-in-out;
}

.night-layer {
  opacity: 0;
  pointer-events: none;
}

.app-container.dark .day-layer { opacity: 0; }
.app-container.dark .night-layer { opacity: 1; pointer-events: auto; }

/* 雲朵動畫 */
.cloud {
  --scale: 1; /* 透過 CSS 變數修正動畫覆蓋 scale 的問題 */
  position: absolute;
  background: rgba(255, 255, 255, 0.8);
  width: 120px;
  height: 40px;
  border-radius: 50px;
  animation: floatClouds linear infinite;
}
.cloud::before, .cloud::after {
  content: '';
  position: absolute;
  background: rgba(255, 255, 255, 0.8);
  border-radius: 50%;
}
.cloud::before { width: 60px; height: 60px; top: -30px; left: 20px; }
.cloud::after { width: 40px; height: 40px; top: -15px; left: 70px; }

.cloud-1 { top: 15%; left: -150px; animation-duration: 45s; animation-delay: -15s; --scale: 0.9; }
.cloud-2 { top: 35%; left: -150px; animation-duration: 30s; animation-delay: -5s; --scale: 0.6; }
.cloud-3 { top: 50%; left: -150px; animation-duration: 60s; animation-delay: -35s; --scale: 1.2; }
.cloud-4 { top: 65%; left: -150px; animation-duration: 40s; animation-delay: -20s; --scale: 0.8; }
.cloud-5 { top: 25%; left: -150px; animation-duration: 55s; animation-delay: -45s; --scale: 1.1; }
.cloud-6 { top: 80%; left: -150px; animation-duration: 35s; animation-delay: -10s; --scale: 0.7; }

@keyframes floatClouds {
  0% { transform: translateX(-150px) scale(var(--scale)); }
  100% { transform: translateX(140vw) scale(var(--scale)); }
}

/* 星星閃爍動畫 */
.star {
  position: absolute;
  background-color: white;
  border-radius: 50%;
  animation: twinkle ease-in-out infinite alternate;
}

@keyframes twinkle {
  0% { opacity: 0.2; transform: scale(0.8); box-shadow: 0 0 2px rgba(255,255,255,0.2); }
  100% { opacity: 1; transform: scale(1.2); box-shadow: 0 0 8px rgba(255,255,255,0.8); }
}

/* 流星動畫 */
.meteor {
  position: absolute;
  width: 2px;
  height: 80px;
  background: linear-gradient(to bottom, rgba(255,255,255,1), rgba(255,255,255,0));
  transform: rotate(45deg);
  opacity: 0;
}
.meteor-1 { top: -10vh; left: 30vw; animation: shootingStar 10s infinite linear; animation-delay: 2s; }
.meteor-2 { top: 10vh; left: -10vw; animation: shootingStar 15s infinite linear; animation-delay: 8s; }
.meteor-3 { top: -20vh; left: 70vw; animation: shootingStar 12s infinite linear; animation-delay: 5s; }

@keyframes shootingStar {
  0% { opacity: 0; transform: translate(0, 0) rotate(45deg); }
  10% { opacity: 1; transform: translate(15vw, 15vh) rotate(45deg); }
  20%, 100% { opacity: 0; transform: translate(45vw, 45vh) rotate(45deg); }
}

.navbar {
  background-color: rgba(255, 255, 255, 0.4);
  backdrop-filter: blur(15px);
  -webkit-backdrop-filter: blur(15px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.5);
  padding: 20px 20px;
  box-shadow: 0 4px 30px rgba(0, 0, 0, 0.05);
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 20px;
  transition: background-color 0.5s ease, border-color 0.5s ease;
}

.top-controls {
  width: 100%;
  max-width: 800px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
  gap: 10px;
}

.right-controls {
  display: flex;
  gap: 10px;
  align-items: center;
}

/* 白噪音選單樣式 */
.noise-container {
  position: relative;
}
.noise-menu {
  position: absolute;
  top: 120%;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(255, 255, 255, 0.9);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border-radius: 12px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
  padding: 8px;
  display: flex;
  flex-direction: column;
  gap: 5px;
  z-index: 100;
  min-width: 120px;
  border: 1px solid rgba(255, 255, 255, 0.5);
}
.noise-menu button {
  background: transparent;
  border: none;
  padding: 8px 12px;
  text-align: left;
  font-size: 1rem;
  font-weight: bold;
  color: #34495e;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.2s;
}
.noise-menu button:hover { background: rgba(0, 0, 0, 0.05); }
.noise-menu button.active { background: rgba(52, 152, 219, 0.2); color: #2980b9; }

.volume-control {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 10px 5px 5px;
  border-top: 1px solid rgba(0, 0, 0, 0.1);
  margin-top: 5px;
}
.volume-control label { font-size: 0.85rem; color: #34495e; font-weight: bold; }
.volume-control input[type="range"] { width: 100%; cursor: pointer; accent-color: #3498db; }

.current-time, .theme-toggle {
  font-size: 0.9rem;
  font-weight: bold;
  color: #34495e;
  background: rgba(255, 255, 255, 0.6);
  padding: 6px 14px;
  border-radius: 20px;
  border: 1px solid rgba(0, 0, 0, 0.05);
  transition: all 0.3s;
  text-align: center;
}

.theme-toggle {
  cursor: pointer;
}
.theme-toggle:hover {
  background: rgba(255, 255, 255, 0.9);
}

.navbar h1 {
  margin: 0 0 15px 0;
  color: #2c3e50;
  font-size: 1.8rem;
  font-weight: 900;
  text-align: center;
  word-break: break-word;
  line-height: 1.4;
}

.tabs {
  display: flex;
  gap: 15px;
}

.tabs button {
  padding: 10px 20px;
  font-size: 1.1rem;
  font-weight: bold;
  border: 1px solid rgba(255, 255, 255, 0.5);
  background-color: rgba(255, 255, 255, 0.3);
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
  color: #34495e;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05), inset 0 2px 5px rgba(255, 255, 255, 0.8), inset 0 -2px 5px rgba(0, 0, 0, 0.05);
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
  position: relative; /* 必須設定為相對定位，提示框才能對齊 */
}

.tabs button:hover {
  background-color: rgba(255, 255, 255, 0.5);
  color: #34495e;
  transform: translateY(-2px);
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.1), inset 0 2px 5px rgba(255, 255, 255, 1), inset 0 -2px 5px rgba(0, 0, 0, 0.05);
  z-index: 10; /* 提升層級，避免在手機版折行時提示框被下排按鈕擋住 */
}

.tabs button.active {
  border-color: rgba(79, 172, 254, 0.6);
  background-color: rgba(79, 172, 254, 0.4);
  color: #0056b3;
  box-shadow: 0 4px 15px rgba(79, 172, 254, 0.3), inset 0 3px 6px rgba(255, 255, 255, 0.6), inset 0 -3px 6px rgba(0, 0, 0, 0.1);
  transform: translateY(1px);
}

/* =========================================
   導覽列 Hover 提示方框 (Tooltip) 樣式
   ========================================= */
/* 提示文字框本體 */
.tabs button::after {
  content: attr(data-tooltip);
  position: absolute;
  top: calc(100% + 12px);
  left: 50%;
  transform: translateX(-50%) translateY(10px);
  background: rgba(44, 62, 80, 0.95);
  color: #fff;
  padding: 8px 12px;
  border-radius: 8px;
  font-size: 0.95rem;
  white-space: pre-wrap; /* 讓文字在手機等小螢幕上可以自然換行避免超出畫面 */
  max-width: 220px;
  pointer-events: none;
  opacity: 0;
  visibility: hidden;
  transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 100;
  font-weight: normal;
}
/* 提示框上方的小三角形 */
.tabs button::before {
  content: '';
  position: absolute;
  top: calc(100% + 2px);
  left: 50%;
  transform: translateX(-50%) translateY(10px);
  border-width: 5px;
  border-style: solid;
  border-color: transparent transparent rgba(44, 62, 80, 0.95) transparent;
  pointer-events: none;
  opacity: 0;
  visibility: hidden;
  transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
  z-index: 100;
}
/* 滑鼠碰到時淡入並微微往上浮動 */
.tabs button:hover::after,
.tabs button:hover::before {
  opacity: 1;
  visibility: visible;
  transform: translateX(-50%) translateY(0);
}

.content {
  padding: 0 20px 40px;
}

/* =========================================================
   夜間模式 (Dark Mode) 全局深度覆蓋 (Deep Styling)
   利用 :deep() 讓夜間模式無縫套用至所有子組件，不須修改其他檔案！
   ========================================================= */
.app-container.dark {
  background: transparent; /* 背景移交給獨立的動畫層處理 */
  color: #e0e0e0;
}
.app-container.dark .navbar {
  background-color: rgba(20, 20, 30, 0.4);
  border-bottom-color: rgba(255, 255, 255, 0.1);
}
.app-container.dark .navbar h1 { color: #e0e0e0; }
.app-container.dark .current-time, .app-container.dark .theme-toggle {
  background: rgba(0, 0, 0, 0.3); color: #ecf0f1; border-color: rgba(255,255,255,0.1);
}
.app-container.dark .theme-toggle:hover { background: rgba(0, 0, 0, 0.6); }
.app-container.dark .tabs button { border-color: rgba(255, 255, 255, 0.1); background-color: rgba(255, 255, 255, 0.05); color: #ccc; box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2), inset 0 2px 5px rgba(255, 255, 255, 0.1), inset 0 -2px 5px rgba(0, 0, 0, 0.2); }
.app-container.dark .tabs button:hover { background-color: rgba(255, 255, 255, 0.15); color: #fff; transform: translateY(-2px); box-shadow: 0 6px 12px rgba(0, 0, 0, 0.3), inset 0 2px 5px rgba(255, 255, 255, 0.2), inset 0 -2px 5px rgba(0, 0, 0, 0.2); }
.app-container.dark .tabs button.active { border-color: rgba(79, 172, 254, 0.5); background-color: rgba(79, 172, 254, 0.3); color: #80d0ff; box-shadow: 0 4px 15px rgba(79, 172, 254, 0.3), inset 0 3px 6px rgba(255, 255, 255, 0.2), inset 0 -3px 6px rgba(0, 0, 0, 0.2); transform: translateY(1px); }
/* 暗色模式時提示方框調整配色 */
.app-container.dark .tabs button::after { background: rgba(220, 220, 230, 0.95); color: #1a1a2e; }
.app-container.dark .tabs button::before { border-color: transparent transparent rgba(220, 220, 230, 0.95) transparent; }

/* 對卡片、背景與所有文字自動適配暗色 */
.app-container.dark :deep(.card) { background: rgba(30, 30, 40, 0.85); border-color: rgba(255, 255, 255, 0.1); box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3); }
.app-container.dark :deep(h1), .app-container.dark :deep(h2), .app-container.dark :deep(h3) { color: #e0e0e0; border-bottom-color: rgba(255, 255, 255, 0.1); }
.app-container.dark :deep(.group-item) { background: rgba(40, 40, 50, 0.95); border-left-color: #555; box-shadow: none; }
.app-container.dark :deep(.group-item.is-first) { background: rgba(60, 50, 20, 0.95); border-left-color: #f1c40f; }
.app-container.dark :deep(.score-display), .app-container.dark :deep(.name), .app-container.dark :deep(.interactive-tool) { color: #e0e0e0; }
.app-container.dark :deep(.empty-text), .app-container.dark :deep(.list-header), .app-container.dark :deep(.rank) { color: #aaa; }
.app-container.dark :deep(.input-group input), .app-container.dark :deep(.poll-input), .app-container.dark :deep(.time-input), .app-container.dark :deep(.theme-input) { background: rgba(20, 20, 30, 0.8); color: #fff; border-color: #555; }
.app-container.dark :deep(.clear-text-btn) { color: #888; }
.app-container.dark :deep(.clear-text-btn:hover) { color: #e74c3c; }
.app-container.dark :deep(.timer-theme-title) { color: #80d0ff; }
.app-container.dark :deep(.options), .app-container.dark :deep(.custom-set) { background: transparent; color: #e0e0e0; border-top-color: rgba(255, 255, 255, 0.1); }
.app-container.dark :deep(.custom-label), .app-container.dark :deep(.unit), .app-container.dark :deep(.input-label) { color: #ccc; }
.app-container.dark :deep(.quick-set button), .app-container.dark :deep(.vote-btn), .app-container.dark :deep(.option-label) { background: rgba(255, 255, 255, 0.1); color: #e0e0e0; border-color: #555; }
.app-container.dark :deep(.quick-set button:hover:not(:disabled)), .app-container.dark :deep(.vote-btn:hover:not(:disabled)), .app-container.dark :deep(.option-label:hover:not(:disabled)) { background: rgba(255, 255, 255, 0.2); }
.app-container.dark :deep(.option-label.is-selected) { background: rgba(52, 152, 219, 0.2); border-color: #3498db; }
.app-container.dark :deep(.badge) { background: rgba(46, 204, 113, 0.2); color: #2ecc71; }
.app-container.dark :deep(.result-banner) { background: rgba(40, 40, 50, 0.9); border-top-color: #555; }
.app-container.dark :deep(.result-banner.high-score) { background: rgba(20, 50, 30, 0.9); border-top-color: #2ecc71; }
.app-container.dark :deep(.score-number) { color: #4facfe; }
.app-container.dark :deep(.score-message) { color: #ccc; }
.app-container.dark :deep(.feedback) { background: rgba(241, 196, 15, 0.1); color: #f1c40f; }
.app-container.dark :deep(.results li) { border-bottom-color: rgba(255, 255, 255, 0.1); }
.app-container.dark :deep(.history-list li) { border-bottom-color: rgba(255, 255, 255, 0.1); }
.app-container.dark :deep(.history-theme) { color: #e0e0e0; }
/* 電子課表夜間模式 */
.app-container.dark :deep(.table-scroll) { border-color: rgba(255, 255, 255, 0.1); }
.app-container.dark :deep(.schedule-table) { background: rgba(30, 30, 40, 0.5); }
.app-container.dark :deep(.schedule-table th) { background: rgba(20, 20, 30, 0.8); border-color: rgba(255, 255, 255, 0.1); }
.app-container.dark :deep(.schedule-table td) { border-color: rgba(255, 255, 255, 0.05); }
.app-container.dark :deep(.schedule-table tbody tr:hover td) { background: rgba(255, 255, 255, 0.05); }
.app-container.dark :deep(.time-col) { background: rgba(0, 0, 0, 0.2); }
.app-container.dark :deep(.period-input), .app-container.dark :deep(.subject-input) { color: #80d0ff; }
.app-container.dark :deep(.time-input), .app-container.dark :deep(.teacher-input) { color: #aaa; }
/* 白噪音夜間模式 */
.app-container.dark .noise-menu { background: rgba(30, 30, 40, 0.9); border-color: rgba(255, 255, 255, 0.1); box-shadow: 0 8px 24px rgba(0, 0, 0, 0.5); }
.app-container.dark .noise-menu button { color: #e0e0e0; }
.app-container.dark .noise-menu button:hover { background: rgba(255, 255, 255, 0.1); }
.app-container.dark .noise-menu button.active { background: rgba(52, 152, 219, 0.3); color: #80d0ff; }
.app-container.dark .volume-control { border-top-color: rgba(255, 255, 255, 0.1); }
.app-container.dark .volume-control label { color: #aaa; }

/* 響應式設計 (RWD)：針對平板與手機裝置優化 */
@media (max-width: 768px) {
  .tabs {
    flex-wrap: wrap;
    justify-content: center;
  }
  .tabs button {
    flex: 1 1 calc(50% - 15px); /* 一排兩個按鈕 */
    font-size: 1rem;
    padding: 8px 10px;
  }
  .top-controls {
    flex-direction: column;
    justify-content: center;
  }
  .navbar h1 {
    font-size: 1.5rem;
  }
  /* 手機版電子課表夜間模式適配 */
  .app-container.dark :deep(.schedule-table tbody tr) {
    background: rgba(30, 30, 40, 0.8);
    border-color: rgba(255, 255, 255, 0.1);
  }
  .app-container.dark :deep(.cell-data::before) { color: #aaa; }
}
</style>
