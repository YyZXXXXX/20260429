<template>
  <div class="app-container" :class="{ 'dark': isDarkMode }">
    <!-- 頂部導航列 -->
    <header class="navbar">
      
      <!-- 頂端控制列 (時間與主題切換) -->
      <div class="top-controls">
        <div class="current-time">🇹🇼 台灣時間：{{ currentTime }}</div>
        <button class="theme-toggle" @click="toggleTheme">
          {{ isDarkMode ? '☀️ 白天模式' : '🌙 夜間模式' }}
        </button>
      </div>

      <h1>🛠️ 課堂互動工具箱</h1>
      <nav class="tabs">
        <button 
          :class="{ active: currentTab === 'picker' }" 
          @click="currentTab = 'picker'"
        >
          🎲 隨機抽籤機
        </button>
        <button 
          :class="{ active: currentTab === 'scoreboard' }" 
          @click="currentTab = 'scoreboard'"
        >
          📊 小組記分板
        </button>
        <button 
          :class="{ active: currentTab === 'timer' }" 
          @click="currentTab = 'timer'"
        >
          ⏱️ 報告計時器
        </button>
        <button 
          :class="{ active: currentTab === 'exam' }" 
          @click="currentTab = 'exam'"
        >
          📝 數位測驗系統
        </button>
      </nav>
    </header>

    <!-- 工具展示區 (使用 v-show 保持組件狀態，避免切換時名單遺失) -->
    <main class="content">
      <RandomPicker v-show="currentTab === 'picker'" />
      <Scoreboard v-show="currentTab === 'scoreboard'" />
      <Timer v-show="currentTab === 'timer'" />
      <ExamSystem v-show="currentTab === 'exam'" />
    </main>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onUnmounted } from 'vue';
import RandomPicker from './components/RandomPicker.vue';
import Scoreboard from './components/Scoreboard.vue';
import Timer from './components/Timer.vue';
import ExamSystem from './components/ExamSystem.vue';

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

// 頁面載入時讀取上次停留的工具分頁
onMounted(() => {
  updateTime();
  timeInterval = setInterval(updateTime, 1000); // 每秒更新時間

  const savedTheme = localStorage.getItem('toolbox-theme');
  if (savedTheme === 'dark') isDarkMode.value = true;

  const savedTab = localStorage.getItem('toolbox-current-tab');
  if (savedTab) currentTab.value = savedTab;
});

// 組件解除掛載時清理計時器
onUnmounted(() => {
  if (timeInterval) clearInterval(timeInterval);
});

// 當分頁切換時，儲存至 Local Storage
watch(currentTab, (newTab) => {
  localStorage.setItem('toolbox-current-tab', newTab);
});
</script>

<style scoped>
.app-container {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  min-height: 100vh;
  background: linear-gradient(135deg, #fdf8f5 0%, #e6d3c0 100%);
  background-attachment: fixed;
  transition: background 0.5s ease, color 0.5s ease;
}

.navbar {
  background-color: rgba(255, 255, 255, 0.7);
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
}

.current-time, .theme-toggle {
  font-size: 0.9rem;
  font-weight: bold;
  color: #34495e;
  background: rgba(255, 255, 255, 0.6);
  padding: 6px 14px;
  border-radius: 20px;
  border: 1px solid rgba(0, 0, 0, 0.05);
  transition: all 0.3s;
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
}

.tabs {
  display: flex;
  gap: 15px;
}

.tabs button {
  padding: 10px 20px;
  font-size: 1.1rem;
  font-weight: bold;
  border: 2px solid #e0e0e0;
  background-color: transparent;
  color: #7f8c8d;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.tabs button:hover {
  background-color: rgba(255, 255, 255, 0.9);
  color: #34495e;
  transform: translateY(-2px);
}

.tabs button.active {
  border-color: #4facfe;
  background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
  color: #ffffff;
  box-shadow: 0 4px 15px rgba(79, 172, 254, 0.4);
}

.content {
  padding: 0 20px 40px;
}

/* =========================================================
   夜間模式 (Dark Mode) 全局深度覆蓋 (Deep Styling)
   利用 :deep() 讓夜間模式無縫套用至所有子組件，不須修改其他檔案！
   ========================================================= */
.app-container.dark {
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
  color: #e0e0e0;
}
.app-container.dark .navbar {
  background-color: rgba(20, 20, 30, 0.7);
  border-bottom-color: rgba(255, 255, 255, 0.1);
}
.app-container.dark .navbar h1 { color: #e0e0e0; }
.app-container.dark .current-time, .app-container.dark .theme-toggle {
  background: rgba(0, 0, 0, 0.3); color: #ecf0f1; border-color: rgba(255,255,255,0.1);
}
.app-container.dark .theme-toggle:hover { background: rgba(0, 0, 0, 0.6); }
.app-container.dark .tabs button { border-color: #555; color: #aaa; }
.app-container.dark .tabs button:hover { background-color: rgba(255, 255, 255, 0.1); color: #fff; }
.app-container.dark .tabs button.active { border-color: #4facfe; background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%); color: #fff; }

/* 對卡片、背景與所有文字自動適配暗色 */
.app-container.dark :deep(.card) { background: rgba(30, 30, 40, 0.85); border-color: rgba(255, 255, 255, 0.1); box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3); }
.app-container.dark :deep(h1), .app-container.dark :deep(h2), .app-container.dark :deep(h3) { color: #e0e0e0; border-bottom-color: rgba(255, 255, 255, 0.1); }
.app-container.dark :deep(.group-item) { background: rgba(40, 40, 50, 0.95); border-left-color: #555; box-shadow: none; }
.app-container.dark :deep(.group-item.is-first) { background: rgba(60, 50, 20, 0.95); border-left-color: #f1c40f; }
.app-container.dark :deep(.score-display), .app-container.dark :deep(.name), .app-container.dark :deep(.interactive-tool) { color: #e0e0e0; }
.app-container.dark :deep(.empty-text), .app-container.dark :deep(.list-header), .app-container.dark :deep(.rank) { color: #aaa; }
.app-container.dark :deep(.input-group input), .app-container.dark :deep(.poll-input), .app-container.dark :deep(.time-input) { background: rgba(20, 20, 30, 0.8); color: #fff; border-color: #555; }
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

/* 響應式設計 (RWD)：針對平板與手機裝置優化 */
@media (max-width: 768px) {
  .tabs {
    flex-wrap: wrap;
    justify-content: center;
  }
  .tabs button {
    flex: 1 1 calc(50% - 15px); /* 一排兩個按鈕 */
    font-size: 1rem;
  }
}
</style>
