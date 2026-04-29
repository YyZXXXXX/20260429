<template>
  <div class="interactive-tool">
    <h1>⏱️ 課堂互動工具箱 - 小組報告計時器</h1>

    <!-- 計時器模式切換 -->
    <div class="mode-switch card">
      <button :class="{ active: timerMode === 'countdown' }" @click="switchMode('countdown')">⏳ 倒數計時</button>
      <button :class="{ active: timerMode === 'stopwatch' }" @click="switchMode('stopwatch')">⏱️ 正向計時</button>
      <button :class="{ active: timerMode === 'pomodoro' }" @click="switchMode('pomodoro')">🍅 番茄鐘</button>
    </div>

    <div class="settings card">
      <!-- 倒數計時設定 -->
      <template v-if="timerMode === 'countdown'">
        <div class="theme-setting">
          <div class="input-wrapper">
            <input type="text" v-model="timerTheme" placeholder="📝 輸入倒數計時主題 (例如：第一組上台報告)..." class="theme-input" />
            <button v-if="timerTheme" class="clear-text-btn" @click="timerTheme = ''" title="清除文字">✖</button>
          </div>
        </div>
        <h2>🕒 快速時間設定</h2>
        <div class="quick-set">
          <button @click="setTime(3)">3 分鐘</button>
          <button @click="setTime(5)">5 分鐘</button>
          <button @click="setTime(10)">10 分鐘</button>
          <button @click="addTime(30)">+30 秒</button>
          <button @click="addTime(-30)" :disabled="timeRemaining < 30">-30 秒</button>
        </div>
        
        <div class="custom-set">
          <span class="custom-label">自訂時間：</span>
          <input type="number" v-model="customHours" min="0" placeholder="0" class="time-input" @keyup.enter="applyCustomTime" /> <span class="unit">時</span>
          <input type="number" v-model="customMinutes" min="0" placeholder="0" class="time-input" @keyup.enter="applyCustomTime" /> <span class="unit">分</span>
          <input type="number" v-model="customSeconds" min="0" max="59" placeholder="0" class="time-input" @keyup.enter="applyCustomTime" /> <span class="unit">秒</span>
          <button class="btn-apply" @click="applyCustomTime">設定</button>
          <button class="btn-clear-time" @click="clearCustomTime" v-if="customHours || customMinutes || customSeconds" title="清除輸入框">清除</button>
        </div>
      </template>

      <!-- 番茄鐘設定 -->
      <template v-else-if="timerMode === 'pomodoro'">
        <h2 class="flex-between">🍅 番茄鐘設定 <span class="phase-badge">{{ pomodoroPhase === 'work' ? '💻 專注時間 (25分)' : '☕ 休息時間 (5分)' }}</span></h2>
        <div class="quick-set">
          <button @click="setPomodoroPhase('work')" :class="{ 'active-btn': pomodoroPhase === 'work' }">切換至 25 分鐘專注</button>
          <button @click="setPomodoroPhase('break')" :class="{ 'active-btn': pomodoroPhase === 'break' }">切換至 5 分鐘休息</button>
        </div>
      </template>

      <!-- 正向計時提示 -->
      <template v-else>
        <h2>⏱️ 正向計時 (Stopwatch)</h2>
        <p class="desc-text">按下開始後，計時器將會從 0 秒開始往上遞增，適合用來測量活動所花費的總時間。</p>
      </template>
    </div>

    <div class="timer-area card">
      <h2 v-if="timerMode === 'countdown' && timerTheme" class="timer-theme-title">{{ timerTheme }}</h2>
      <div 
        class="display-screen" 
        :class="{ 
          'is-pomodoro-style': timerMode === 'pomodoro',
          'is-warning': timerMode === 'countdown' && timeRemaining <= 10 && timeRemaining > 0 && isRunning,
          'is-time-up': timerMode === 'countdown' && timeRemaining === 0 && hasStarted 
        }"
      >
        <div v-if="timerMode === 'pomodoro'" class="huge-tomato" :class="{ 'tomato-warning': timeRemaining <= 10 && timeRemaining > 0 && isRunning, 'tomato-time-up': timeRemaining === 0 && hasStarted }">🍅</div>
        <span class="time-display" :class="{ 'is-long': isLongTime }">{{ formattedTime }}</span>
      </div>

      <div class="controls">
        <button class="btn-start" v-if="!isRunning" @click="startTimer" :disabled="timerMode !== 'stopwatch' && timeRemaining === 0">
          ▶️ 開始
        </button>
        <button class="btn-pause" v-else @click="pauseTimer">
          ⏸️ 暫停
        </button>
        <button class="btn-lap" v-if="isRunning && timerMode === 'stopwatch'" @click="recordLap">
          ⏱️ 紀錄
        </button>
        <button class="btn-reset" @click="resetTimer">
          ⏹️ 重設
        </button>
      </div>
    </div>

    <!-- 計時紀錄區 -->
    <div class="results card" v-if="timerMode === 'stopwatch' && laps.length > 0">
      <div class="flex-between">
        <h2>📋 計時紀錄</h2>
        <button class="remove-btn" @click="clearLaps" title="清除紀錄">🗑️ 刪除紀錄</button>
      </div>
      <ul class="laps-list">
        <li v-for="(lap, index) in laps" :key="index">
          <span class="lap-number">第 {{ index + 1 }} 次</span> {{ formatLapTime(lap) }}
        </li>
      </ul>
    </div>

    <!-- 倒數計時歷史紀錄區 -->
    <div class="results card" v-if="timerMode === 'countdown' && timerHistory.length > 0">
      <div class="flex-between">
        <h2>📜 計時設定紀錄</h2>
        <button class="remove-btn" @click="clearAllHistory" title="清空所有紀錄">🗑️ 一鍵清空</button>
      </div>
      <ul class="history-list">
        <li v-for="record in timerHistory" :key="record.id">
          <div class="history-info">
            <span class="history-time">{{ formatHistoryTime(record.time) }}</span>
            <span class="history-theme">{{ record.theme }}</span>
          </div>
          <div class="history-actions">
            <button class="btn-reapply" @click="applyHistory(record)">🔄 重新套用</button>
            <button class="remove-btn" @click="removeHistory(record.id)" title="刪除此紀錄">✖</button>
          </div>
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onUnmounted, onMounted, watch } from 'vue';

const timerMode = ref('countdown'); // 'countdown', 'stopwatch', 'pomodoro'
const timeRemaining = ref(180); // 預設 3 分鐘 (180秒)
const timeElapsed = ref(0); // 正向計時使用
const pomodoroPhase = ref('work'); // 'work' 或 'break'

const isRunning = ref(false);
const hasStarted = ref(false);
let timerInterval = null;

const customHours = ref('');
const customMinutes = ref('');
const customSeconds = ref('');
const timerTheme = ref('');
const laps = ref([]); // 計時紀錄
const timerHistory = ref([]); // 倒數計時歷史紀錄

// Local Storage：頁面載入時讀取歷史紀錄
onMounted(() => {
  const savedHistory = localStorage.getItem('toolbox-timer-history');
  if (savedHistory) {
    timerHistory.value = JSON.parse(savedHistory);
  }
});

// 當歷史紀錄變更時自動儲存
watch(timerHistory, (newVal) => {
  localStorage.setItem('toolbox-timer-history', JSON.stringify(newVal));
}, { deep: true });

// 切換模式
const switchMode = (mode) => {
  stopTimer();
  timerMode.value = mode;
  hasStarted.value = false;
  if (mode === 'countdown') {
    timeRemaining.value = 180;
  } else if (mode === 'stopwatch') {
    timeElapsed.value = 0;
    laps.value = []; // 切換模式時清空紀錄
  } else if (mode === 'pomodoro') {
    setPomodoroPhase('work');
  }
};

// 番茄鐘階段切換
const setPomodoroPhase = (phase) => {
  stopTimer();
  pomodoroPhase.value = phase;
  hasStarted.value = false;
  timeRemaining.value = phase === 'work' ? 25 * 60 : 5 * 60;
};

// 將秒數格式化為 HH:MM:SS 或 MM:SS (根據當前模式判斷)
const formattedTime = computed(() => {
  let target = timerMode.value === 'stopwatch' ? timeElapsed.value : timeRemaining.value;
  const h = Math.floor(target / 3600).toString().padStart(2, '0');
  const m = Math.floor(target / 60).toString().padStart(2, '0');
  const s = (target % 60).toString().padStart(2, '0');
  return target >= 3600 ? `${h}:${m}:${s}` : `${m}:${s}`;
});

const isLongTime = computed(() => {
  let target = timerMode.value === 'stopwatch' ? timeElapsed.value : timeRemaining.value;
  return target >= 3600;
});

// 設定指定分鐘數
const setTime = (minutes) => {
  stopTimer();
  timeRemaining.value = minutes * 60;
  hasStarted.value = false;
};

// 套用自訂時間
const applyCustomTime = () => {
  const h = Math.max(0, parseInt(customHours.value) || 0);
  const m = Math.max(0, parseInt(customMinutes.value) || 0);
  const s = Math.max(0, parseInt(customSeconds.value) || 0);
  const total = h * 3600 + m * 60 + s;
  
  if (total > 0) {
    stopTimer();
    timeRemaining.value = total;
    hasStarted.value = false;
    customHours.value = ''; // 設定後清空輸入框，保持畫面乾淨
    customMinutes.value = '';
    customSeconds.value = '';
  }
};

// 清除自訂時間輸入框
const clearCustomTime = () => {
  customHours.value = '';
  customMinutes.value = '';
  customSeconds.value = '';
};

// 微調秒數
const addTime = (seconds) => {
  if (timeRemaining.value + seconds >= 0) {
    timeRemaining.value += seconds;
  }
};

// 將秒數格式化為 顯示用字串 (如 1小時 5分鐘 30秒)
const formatHistoryTime = (totalSeconds) => {
  const h = Math.floor(totalSeconds / 3600);
  const m = Math.floor((totalSeconds % 3600) / 60);
  const s = totalSeconds % 60;
  let parts = [];
  if (h > 0) parts.push(`${h} 小時`);
  if (m > 0) parts.push(`${m} 分鐘`);
  if (s > 0 || totalSeconds === 0) parts.push(`${s} 秒`);
  return parts.join(' ');
};

// 重新套用歷史紀錄
const applyHistory = (record) => {
  stopTimer();
  timerTheme.value = record.theme === '未命名計時' ? '' : record.theme;
  timeRemaining.value = record.time;
  hasStarted.value = false;
  clearCustomTime(); // 清除自訂欄位讓畫面更整潔
};

const removeHistory = (id) => { timerHistory.value = timerHistory.value.filter(r => r.id !== id); };
const clearAllHistory = () => { timerHistory.value = []; };

// 開始倒數
const startTimer = () => {
  if (isRunning.value) return;
  if (timerMode.value !== 'stopwatch' && timeRemaining.value <= 0) return;
  
  // 若為倒數計時且是第一次開始，自動記錄至歷史
  if (!hasStarted.value && timerMode.value === 'countdown') {
    const isDuplicate = timerHistory.value.length > 0 && 
      timerHistory.value[0].time === timeRemaining.value && 
      timerHistory.value[0].theme === (timerTheme.value || '未命名計時');
      
    if (!isDuplicate) {
      timerHistory.value.unshift({ id: Date.now(), theme: timerTheme.value || '未命名計時', time: timeRemaining.value });
      // 最多保留最近 15 筆紀錄避免過於冗長
      if (timerHistory.value.length > 15) timerHistory.value.pop();
    }
  }
  
  isRunning.value = true;
  hasStarted.value = true;
  
  timerInterval = setInterval(() => {
    if (timerMode.value === 'stopwatch') {
      timeElapsed.value++;
    } else {
      if (timeRemaining.value > 0) {
        timeRemaining.value--;
        
        // 倒數最後 10 秒發出警告音
        if (timeRemaining.value <= 10 && timeRemaining.value > 0) {
          playBeep(400, 0.1, 'sine');
        }
        
        // 時間到
        if (timeRemaining.value === 0) {
          stopTimer();
          playTimeUpSound();
          
          // 番茄鐘自動切換階段
          if (timerMode.value === 'pomodoro') {
            setPomodoroPhase(pomodoroPhase.value === 'work' ? 'break' : 'work');
          }
        }
      }
    }
  }, 1000);
};

const pauseTimer = () => {
  isRunning.value = false;
  clearInterval(timerInterval);
};

const stopTimer = () => {
  isRunning.value = false;
  clearInterval(timerInterval);
};

// 紀錄當下計次時間
const recordLap = () => {
  laps.value.push(timeElapsed.value);
};

// 刪除所有計時紀錄
const clearLaps = () => {
  laps.value = [];
};

// 將單筆紀錄秒數轉為格式
const formatLapTime = (target) => {
  const h = Math.floor(target / 3600).toString().padStart(2, '0');
  const m = Math.floor((target % 3600) / 60).toString().padStart(2, '0');
  const s = (target % 60).toString().padStart(2, '0');
  return target >= 3600 ? `${h}:${m}:${s}` : `${m}:${s}`;
};

const resetTimer = () => {
  stopTimer();
  hasStarted.value = false;
  if (timerMode.value === 'countdown') {
    timeRemaining.value = 180;
  } else if (timerMode.value === 'stopwatch') {
    timeElapsed.value = 0;
    laps.value = []; // 重設時也清空紀錄
  } else if (timerMode.value === 'pomodoro') {
    setPomodoroPhase('work');
  }
};

// 運用 Web Audio API 產生提示音效
const playBeep = (freq, duration, type) => {
  try {
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    osc.type = type;
    osc.frequency.value = freq;
    gain.gain.setValueAtTime(0.5, audioCtx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + duration);
    osc.start(audioCtx.currentTime);
    osc.stop(audioCtx.currentTime + duration);
  } catch (err) {
    console.warn('瀏覽器阻擋或不支援音效播放');
  }
};

// 時間到的連續響鈴
const playTimeUpSound = () => {
  playBeep(800, 0.4, 'square');
  setTimeout(() => playBeep(800, 0.4, 'square'), 300);
  setTimeout(() => playBeep(800, 0.6, 'square'), 600);
};

// 確保組件卸載時清除計時器，避免 Memory Leak
onUnmounted(() => {
  stopTimer();
});
</script>

<style scoped>
.interactive-tool { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; color: #333; }
h1 { text-align: center; color: #2c3e50; margin-bottom: 30px; line-height: 1.4; word-break: break-word; }
.card { 
  background: rgba(255, 255, 255, 0.85); 
  backdrop-filter: blur(12px); -webkit-backdrop-filter: blur(12px);
  border-radius: 12px; padding: 24px; margin-bottom: 24px; 
  box-shadow: 0 8px 32px rgba(31, 38, 135, 0.07); 
  border: none; 
}
h2 { margin-top: 0; font-size: 1.2rem; color: #34495e; border-bottom: 2px solid #eee; padding-bottom: 10px; margin-bottom: 15px; }

/* 模式切換 */
.mode-switch { display: flex; gap: 15px; padding: 15px; justify-content: center; flex-wrap: wrap; }
.mode-switch button { padding: 12px 24px; font-size: 1.1rem; font-weight: bold; border: 2px solid #e0e0e0; background: rgba(255,255,255,0.5); color: #7f8c8d; border-radius: 8px; cursor: pointer; transition: all 0.3s; flex: 1;}
.mode-switch button.active { border-color: #4facfe; background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%); color: #fff; box-shadow: 0 4px 15px rgba(79, 172, 254, 0.4); }

.flex-between { display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 10px; border-bottom: 2px solid #eee; padding-bottom: 10px; margin-bottom: 15px; }
.flex-between h2 { border-bottom: none; padding-bottom: 0; margin-bottom: 0; }
.phase-badge { background: #eafaf1; color: #27ae60; padding: 4px 10px; border-radius: 12px; font-size: 0.9rem; font-weight: bold; border: 1px solid #2ecc71; display: inline-block;}
.desc-text { color: #7f8c8d; line-height: 1.5; font-size: 1.05rem; }

.quick-set { display: flex; gap: 10px; flex-wrap: wrap; margin-bottom: 10px; }
.quick-set button { flex: 1; padding: 12px 10px; background-color: #ecf0f1; color: #2c3e50; border: none; border-radius: 8px; cursor: pointer; font-size: 1rem; font-weight: bold; transition: background 0.2s; min-width: 80px; }
.quick-set button:hover:not(:disabled) { background-color: #bdc3c7; }
.quick-set button:disabled { opacity: 0.5; cursor: not-allowed; }
.quick-set button.active-btn { background-color: #e74c3c; color: white; }

/* 自訂時間區塊樣式 */
.custom-set { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; margin-top: 15px; padding-top: 15px; border-top: 1px dashed rgba(0,0,0,0.15); }
.custom-label { font-weight: bold; color: #34495e; font-size: 0.95rem; }
.time-input { width: 60px; padding: 8px; border: 2px solid #e0e0e0; border-radius: 8px; text-align: center; font-size: 1rem; background: rgba(255,255,255,0.8); transition: border-color 0.2s; }
.time-input:focus { outline: none; border-color: #3498db; background: #fff; }
.unit { color: #7f8c8d; font-size: 0.9rem; }
.btn-apply { padding: 8px 16px; background-color: #3498db; color: white; border: none; border-radius: 8px; cursor: pointer; font-weight: bold; transition: background 0.2s; margin-left: auto; }
.btn-apply:hover { background-color: #2980b9; }

.btn-clear-time { padding: 8px 16px; background-color: #95a5a6; color: white; border: none; border-radius: 8px; cursor: pointer; font-weight: bold; transition: background 0.2s; margin-left: 8px; }
.btn-clear-time:hover { background-color: #7f8c8d; }

.theme-setting { margin-bottom: 20px; }
.input-wrapper { position: relative; display: flex; align-items: center; width: 100%; }
.theme-input { width: 100%; padding: 12px; padding-right: 40px; border: 2px solid #e0e0e0; border-radius: 8px; font-size: 1.1rem; box-sizing: border-box; text-align: center; transition: border-color 0.2s; font-weight: bold; color: #34495e; background: rgba(255,255,255,0.8); }
.clear-text-btn { position: absolute; right: 12px; background: transparent; border: none; font-size: 1.2rem; color: #aaa; cursor: pointer; padding: 0; display: flex; align-items: center; justify-content: center; transition: color 0.2s; }
.clear-text-btn:hover { color: #e74c3c; }
.theme-input:focus { outline: none; border-color: #3498db; background: #fff; }
.timer-theme-title { text-align: center; color: #2980b9; font-size: 1.6rem; margin-bottom: 15px; border: none; padding-bottom: 0; word-break: break-word; }

.display-screen { position: relative; height: 180px; background: #34495e; border-radius: 12px; display: flex; align-items: center; justify-content: center; margin-bottom: 20px; transition: all 0.3s ease; box-shadow: inset 0 4px 10px rgba(0,0,0,0.3); overflow: hidden; }
.display-screen.is-pomodoro-style { background: transparent; box-shadow: none; overflow: visible; height: 280px; }

.time-display { position: relative; z-index: 2; font-size: 5.5rem; font-weight: 900; color: white; letter-spacing: 4px; font-variant-numeric: tabular-nums; }
.time-display.is-long { font-size: 4.2rem; } /* 時長較長時縮小字體以防破版 */
.display-screen.is-pomodoro-style .time-display { font-size: 4rem; text-shadow: 0 4px 12px rgba(0,0,0,0.6), 0 0 25px rgba(0,0,0,0.9); top: 20px; }

.huge-tomato { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); z-index: 1; font-size: 260px; user-select: none; pointer-events: none; filter: drop-shadow(0 8px 16px rgba(0,0,0,0.2)); transition: all 0.3s ease; line-height: 1; }
.huge-tomato.tomato-warning { animation: tomatoPulse 1s infinite alternate; }
.huge-tomato.tomato-time-up { animation: tomatoShake 0.2s infinite; filter: drop-shadow(0 0 30px rgba(231,76,60,0.8)); }

/* 視覺閃爍提示 */
.display-screen.is-warning { background: #e67e22; animation: pulseWarning 1s infinite alternate; }
.display-screen.is-time-up { background: #c0392b; animation: flashRed 0.3s infinite alternate; }

@keyframes pulseWarning {
  0% { background: #e67e22; box-shadow: inset 0 4px 10px rgba(0,0,0,0.3); }
  100% { background: #d35400; box-shadow: inset 0 4px 20px rgba(0,0,0,0.6), 0 0 15px rgba(230, 126, 34, 0.6); }
}
@keyframes flashRed {
  0% { background: #c0392b; transform: scale(1); }
  100% { background: #e74c3c; transform: scale(1.02); box-shadow: 0 0 25px rgba(231, 76, 60, 0.8); }
}
@keyframes tomatoPulse {
  0% { transform: translate(-50%, -50%) scale(1); filter: drop-shadow(0 8px 16px rgba(0,0,0,0.2)); }
  100% { transform: translate(-50%, -50%) scale(1.05); filter: drop-shadow(0 8px 25px rgba(231, 76, 60, 0.8)); }
}
@keyframes tomatoShake {
  0%, 100% { transform: translate(-50%, -50%) rotate(-5deg) scale(1.1); }
  50% { transform: translate(-50%, -50%) rotate(5deg) scale(1.1); }
}

.controls { display: flex; gap: 15px; }
.controls button { flex: 1; padding: 16px; font-size: 1.2rem; font-weight: bold; border: none; border-radius: 12px; cursor: pointer; color: white; box-shadow: 0 4px 6px rgba(0,0,0,0.1); transition: transform 0.1s, opacity 0.2s; }
.controls button:active:not(:disabled) { transform: translateY(2px); box-shadow: none; }
.controls button:disabled { opacity: 0.5; cursor: not-allowed; }

.btn-start { background: linear-gradient(135deg, #2ecc71, #27ae60); }
.btn-pause { background: linear-gradient(135deg, #f1c40f, #f39c12); }
.btn-reset { background: linear-gradient(135deg, #e74c3c, #c0392b); }
.btn-lap { background: linear-gradient(135deg, #9b59b6, #8e44ad); }

/* 計時紀錄列表 */
.results { margin-top: 20px; }
.results ul { list-style: none; padding: 0; margin: 0; }
.results li { padding: 12px 16px; border-bottom: 1px solid #f0f0f0; font-size: 1.1rem; display: flex; align-items: center; gap: 12px; flex-wrap: wrap; color: #333; }
.results li:last-child { border-bottom: none; }
.lap-number { background: #3498db; color: white; font-size: 0.85rem; padding: 4px 8px; border-radius: 4px; font-weight: bold; }
.remove-btn { background: none; border: none; color: #e74c3c; cursor: pointer; font-weight: bold; font-size: 1rem; display: flex; align-items: center; opacity: 0.7; transition: opacity 0.2s; padding: 4px 8px; }
.remove-btn:hover { opacity: 1; }

/* 歷史紀錄列表 */
.history-list { list-style: none; padding: 0; margin: 0; }
.history-list li { padding: 12px 16px; border-bottom: 1px solid #f0f0f0; display: flex; justify-content: space-between; align-items: center; gap: 12px; flex-wrap: wrap; color: #333; }
.history-list li:last-child { border-bottom: none; }
.history-info { display: flex; align-items: center; gap: 12px; flex: 1; min-width: 200px; }
.history-time { background: #34495e; color: white; font-size: 0.9rem; padding: 4px 10px; border-radius: 6px; font-weight: bold; white-space: nowrap; }
.history-theme { font-weight: bold; color: #2c3e50; word-break: break-word; }
.history-actions { display: flex; align-items: center; gap: 8px; }
.btn-reapply { background: #2ecc71; color: white; border: none; padding: 6px 12px; border-radius: 6px; cursor: pointer; font-weight: bold; font-size: 0.9rem; transition: background 0.2s; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
.btn-reapply:hover { background: #27ae60; transform: translateY(-1px); }
.btn-reapply:active { transform: translateY(1px); box-shadow: none; }

/* 響應式設計 (RWD) */
@media (max-width: 600px) {
  h1 {
    font-size: 1.5rem;
  }
  .mode-switch {
    flex-direction: column;
  }
  .time-display {
    font-size: 4.5rem;
  }
  .time-display.is-long {
    font-size: 3rem;
  }
  .display-screen.is-pomodoro-style { height: 220px; }
  .display-screen.is-pomodoro-style .time-display { font-size: 3.2rem; top: 15px; }
  .huge-tomato { font-size: 220px; }
  
  .quick-set button {
    flex: 1 1 30%;
  }
  .controls {
    flex-direction: column;
  }
  .custom-set {
    flex-wrap: wrap; justify-content: center;
  }
  .btn-apply, .btn-clear-time { margin-left: 0; width: 100%; margin-top: 5px; }
  
  .history-info { flex-direction: column; align-items: flex-start; gap: 6px; }
  .history-actions { width: 100%; justify-content: flex-end; }
}
</style>