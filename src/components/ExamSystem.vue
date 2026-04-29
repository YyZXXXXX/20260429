<template>
  <div class="interactive-tool">
    <h1>📝 課堂互動工具箱 - 數位測驗系統</h1>

    <!-- 系統模式切換區 -->
    <div class="mode-switch card">
      <button 
        :class="{ active: activeMode === 'exam' }" 
        @click="activeMode = 'exam'"
      >
        📝 互動測驗卷
      </button>
      <button 
        :class="{ active: activeMode === 'poll' }" 
        @click="activeMode = 'poll'"
      >
        🙋 即時反饋與投票
      </button>
    </div>

    <!-- ==================== 模式一：互動測驗卷 ==================== -->
    <div v-if="activeMode === 'exam'" class="exam-mode">
      
      <!-- 測驗結果展示 (提交後顯示) -->
      <div v-if="isSubmitted" class="result-banner card" :class="{ 'high-score': score >= 80 }">
        <h2>🏆 測驗結果</h2>
        <div class="score-display">
          <span class="score-number">{{ score }}</span> 分
        </div>
        <p class="score-message">{{ scoreMessage }}</p>
        <button class="action-btn mt-15" @click="resetExam">↻ 重新測驗</button>
      </div>

      <!-- 測驗題目列表 -->
      <div 
        v-for="(q, index) in questions" 
        :key="q.id" 
        class="question-card card" 
        :class="{ 
          'is-correct': isSubmitted && checkCorrect(q), 
          'is-wrong': isSubmitted && !checkCorrect(q) 
        }"
      >
        <h3>
          <span class="q-number">Q{{ index + 1 }}</span>
          {{ q.text }}
          <span class="q-type badge">{{ getTypeName(q.type) }}</span>
        </h3>
        
        <div class="options">
          <label 
            v-for="(opt, oIndex) in q.options" 
            :key="oIndex" 
            class="option-label" 
            :class="{ 
              'correct-ans': isSubmitted && isOptionCorrect(q, opt), 
              'wrong-ans': isSubmitted && isOptionWrong(q, opt),
              'is-selected': isSelected(q.id, opt, q.type)
            }"
          >
            <input 
              v-if="q.type === 'multiple'" 
              type="checkbox" 
              :value="opt" 
              v-model="userAnswers[q.id]" 
              :disabled="isSubmitted"
            >
            <input 
              v-else 
              type="radio" 
              :name="'q-' + q.id" 
              :value="opt" 
              v-model="userAnswers[q.id]" 
              :disabled="isSubmitted"
            >
            <span class="opt-text">{{ opt }}</span>
            
            <!-- 提交後在選項旁顯示小圖示 -->
            <span v-if="isSubmitted && isOptionCorrect(q, opt)" class="icon-check">✅</span>
            <span v-if="isSubmitted && isOptionWrong(q, opt)" class="icon-cross">❌</span>
          </label>
        </div>
        
        <!-- 解析區塊 (提交後顯示) -->
        <div v-if="isSubmitted" class="feedback">
          <p class="explanation">💡 <strong>解析：</strong> {{ q.explanation }}</p>
        </div>
      </div>

      <!-- 提交按鈕 -->
      <div v-if="!isSubmitted" class="submit-area">
        <button class="submit-btn" @click="submitExam">📝 提交測驗，觀看解析</button>
      </div>
    </div>

    <!-- ==================== 模式二：即時反饋 (Real-time Q&A) ==================== -->
    <div v-if="activeMode === 'poll'" class="poll-mode card">
      <h2>🙋 課堂即時反饋 (Real-time Q&A)</h2>
      
      <!-- 教師設定問題 -->
      <div v-if="pollState === 'setup'" class="setup-area">
        <label class="input-label">設定即時問答題目：</label>
        <div class="input-wrapper">
          <input v-model="pollQuestion" placeholder="例如：大家今天覺得上課進度如何？" class="poll-input" />
          <button v-if="pollQuestion" class="clear-text-btn" @click="pollQuestion = ''" title="清除文字">✖</button>
        </div>
        <button class="action-btn" @click="startPoll">🚀 發起即時投票</button>
      </div>

      <!-- 學生作答與結果展示區 -->
      <div v-if="pollState === 'active' || pollState === 'result'" class="active-area">
        <h3 class="poll-title">📋 {{ pollQuestion }}</h3>
        
        <div class="poll-options">
          <div v-for="(opt, index) in pollOptions" :key="index" class="poll-option-container">
            <button 
              class="vote-btn" 
              @click="votePoll(index)" 
              :disabled="pollState === 'result'"
            >
              {{ opt }}
            </button>
            
            <!-- 顯示投票結果的長條圖 -->
            <div v-if="pollState === 'result'" class="progress-container">
              <div class="progress-bar" :style="{ width: getVotePercentage(index) + '%' }"></div>
              <span class="vote-count">{{ pollVotes[index] }} 票 ({{ getVotePercentage(index) }}%)</span>
            </div>
          </div>
        </div>
        
        <button v-if="pollState === 'result'" class="action-btn mt-20 secondary" @click="resetPoll">
          ↻ 發起新投票
        </button>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue';

// --- 狀態管理 ---
const activeMode = ref('exam'); // 'exam' 或 'poll'

// --- 音效系統 ---
let audioCtx = null;
const initAudio = () => {
  try {
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    if (audioCtx.state === 'suspended') audioCtx.resume();
  } catch (e) {}
};

const playResultSound = (isPass) => {
  try {
    if (!audioCtx) return;
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.connect(gain); gain.connect(audioCtx.destination);
    osc.type = 'square';
    if (isPass) {
      osc.frequency.setValueAtTime(400, audioCtx.currentTime);
      osc.frequency.setValueAtTime(600, audioCtx.currentTime + 0.1);
      osc.frequency.setValueAtTime(800, audioCtx.currentTime + 0.2);
    } else {
      osc.frequency.setValueAtTime(400, audioCtx.currentTime);
      osc.frequency.setValueAtTime(300, audioCtx.currentTime + 0.2);
    }
    gain.gain.setValueAtTime(0.05, audioCtx.currentTime);
    gain.gain.linearRampToValueAtTime(0, audioCtx.currentTime + 0.4);
    osc.start(); osc.stop(audioCtx.currentTime + 0.4);
  } catch(e) {}
};

// ==================== 測驗卷邏輯 (Exam Mode) ====================
const questions = ref([
  {
    id: 1,
    type: 'single', // 單選
    text: '下列哪一個是 Vue.js 的核心特性？',
    options: ['響應式資料綁定', '操作 DOM 樹', '後端路由處理', '資料庫連線'],
    correctAnswer: '響應式資料綁定',
    explanation: 'Vue.js 是一個前端框架，其最大的特色在於響應式資料綁定，讓開發者只需關注資料狀態，畫面會自動更新。'
  },
  {
    id: 2,
    type: 'multiple', // 複選
    text: '下列哪些屬於網頁前端技術？（請選擇所有正確答案）',
    options: ['HTML', 'MySQL', 'CSS', 'JavaScript'],
    correctAnswer: ['HTML', 'CSS', 'JavaScript'],
    explanation: 'MySQL 是關聯式資料庫管理系統，屬於後端技術領域。'
  },
  {
    id: 3,
    type: 'tf', // 是非
    text: 'Vue 3 推薦使用 Composition API 作為組織元件邏輯的方式。',
    options: ['是', '否'],
    correctAnswer: '是',
    explanation: 'Composition API 是 Vue 3 的重要新特性，提供更好的程式碼覆用性與邏輯組織。'
  }
]);

const userAnswers = ref({});
const isSubmitted = ref(false);
const score = ref(0);

// Local Storage：讀取先前的作答狀態
onMounted(() => {
  const savedAnswers = localStorage.getItem('toolbox-exam-answers');
  if (savedAnswers) userAnswers.value = JSON.parse(savedAnswers);
  
  const savedScore = localStorage.getItem('toolbox-exam-score');
  if (savedScore !== null) {
    score.value = Number(savedScore);
    isSubmitted.value = true;
  }
});

// 初始化作答資料結構
const initAnswers = () => {
  questions.value.forEach(q => {
    userAnswers.value[q.id] = q.type === 'multiple' ? [] : '';
  });
};
initAnswers();

// 自動儲存當前作答至 Local Storage
watch(userAnswers, (newVal) => {
  localStorage.setItem('toolbox-exam-answers', JSON.stringify(newVal));
}, { deep: true });

const getTypeName = (type) => {
  if (type === 'single') return '單選';
  if (type === 'multiple') return '複選';
  return '是非';
};

// 判斷選項是否被選取（用於 UI 高亮）
const isSelected = (qId, opt, type) => {
  const ans = userAnswers.value[qId];
  if (type === 'multiple') return ans.includes(opt);
  return ans === opt;
};

// 送出計算分數
const submitExam = () => {
  initAudio();
  let currentScore = 0;
  const pointsPerQuestion = 100 / questions.value.length;

  questions.value.forEach(q => {
    const ans = userAnswers.value[q.id];
    if (q.type === 'multiple') {
      if (Array.isArray(ans) && ans.length === q.correctAnswer.length && ans.every(val => q.correctAnswer.includes(val))) {
        currentScore += pointsPerQuestion;
      }
    } else {
      if (ans === q.correctAnswer) {
        currentScore += pointsPerQuestion;
      }
    }
  });
  
  score.value = Math.round(currentScore);
  isSubmitted.value = true;
  localStorage.setItem('toolbox-exam-score', score.value.toString());
  playResultSound(score.value >= 60);
};

// 重設測驗
const resetExam = () => {
  initAnswers();
  isSubmitted.value = false;
  score.value = 0;
  
  // 清除快取
  localStorage.removeItem('toolbox-exam-score');
  localStorage.removeItem('toolbox-exam-answers');
};

// 判斷邏輯（用於顯示解析的視覺回饋）
const checkCorrect = (q) => {
  const ans = userAnswers.value[q.id];
  if (q.type === 'multiple') {
    return Array.isArray(ans) && ans.length === q.correctAnswer.length && ans.every(val => q.correctAnswer.includes(val));
  }
  return ans === q.correctAnswer;
};
const isOptionCorrect = (q, opt) => q.type === 'multiple' ? q.correctAnswer.includes(opt) : q.correctAnswer === opt;
const isOptionWrong = (q, opt) => {
  const ans = userAnswers.value[q.id];
  if (q.type === 'multiple') return ans.includes(opt) && !q.correctAnswer.includes(opt);
  return ans === opt && q.correctAnswer !== opt;
};

const scoreMessage = computed(() => {
  if (score.value >= 100) return '太完美了！滿分！🎉';
  if (score.value >= 80) return '表現非常棒！繼續保持！👍';
  if (score.value >= 60) return '及格了，但還有進步空間喔！💪';
  return '別灰心，看看下方解析再試一次吧！📚';
});

// ==================== 即時投票邏輯 (Poll Mode) ====================
const pollState = ref('setup'); // 'setup', 'active', 'result'
const pollQuestion = ref('這堂課目前為止，大家都聽得懂嗎？');
const pollOptions = ref(['A. 完全聽懂', 'B. 大部分聽懂', 'C. 有點模糊', 'D. 完全聽不懂']);
const pollVotes = ref([0, 0, 0, 0]);

const startPoll = () => {
  if (pollQuestion.value.trim() === '') return;
  pollState.value = 'active';
};

const votePoll = (index) => {
  pollVotes.value[index]++;
  pollState.value = 'result';
};

const getVotePercentage = (index) => {
  const total = pollVotes.value.reduce((acc, curr) => acc + curr, 0);
  if (total === 0) return 0;
  return Math.round((pollVotes.value[index] / total) * 100);
};

const resetPoll = () => {
  pollVotes.value = [0, 0, 0, 0];
  pollState.value = 'setup';
};
</script>

<style scoped>
.interactive-tool { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; color: #333; }
h1 { text-align: center; color: #2c3e50; margin-bottom: 30px; line-height: 1.4; word-break: break-word; }
.card { background: #ffffff; border-radius: 12px; padding: 24px; margin-bottom: 24px; box-shadow: 0 4px 12px rgba(0,0,0,0.08); border: 1px solid #eaeaea; }

/* 模式切換 */
.mode-switch { display: flex; gap: 15px; padding: 15px; justify-content: center; }
.mode-switch button { padding: 12px 24px; font-size: 1.1rem; font-weight: bold; border: 2px solid #e0e0e0; background: transparent; color: #7f8c8d; border-radius: 8px; cursor: pointer; transition: all 0.3s; }
.mode-switch button.active { border-color: #3498db; background: #3498db; color: #fff; }

/* 測驗結果 Banner */
.result-banner { text-align: center; background: #f8f9fa; border-top: 6px solid #95a5a6; }
.result-banner.high-score { border-top-color: #2ecc71; background: #f0fdf4; }
.score-display { font-size: 1.5rem; margin: 15px 0; font-weight: bold; }
.score-number { font-size: 4rem; color: #2c3e50; }
.score-message { color: #666; font-size: 1.2rem; }

/* 題目樣式 */
.question-card { border-left: 6px solid #bdc3c7; transition: all 0.3s; }
.question-card.is-correct { border-left-color: #2ecc71; background-color: #f9fffb; }
.question-card.is-wrong { border-left-color: #e74c3c; background-color: #fff9f9; }
.q-number { color: #3498db; font-weight: 900; margin-right: 8px; }
.badge { font-size: 0.8rem; background: #ecf0f1; color: #7f8c8d; padding: 4px 8px; border-radius: 4px; vertical-align: middle; margin-left: 10px; display: inline-block; white-space: nowrap; }
h3 { line-height: 1.5; word-break: break-word; }

/* 選項樣式 */
.options { display: flex; flex-direction: column; gap: 10px; margin-top: 15px; }
.option-label { display: flex; align-items: flex-start; padding: 12px 16px; border: 2px solid #ecf0f1; border-radius: 8px; cursor: pointer; transition: all 0.2s; background: #fff; word-break: break-word; line-height: 1.4; gap: 8px; }
.option-label:hover:not(:disabled) { border-color: #bdc3c7; }
.option-label.is-selected { border-color: #3498db; background: #ebf5fb; }
.option-label input { margin-top: 4px; transform: scale(1.2); cursor: pointer; flex-shrink: 0; }
.opt-text { flex: 1; font-size: 1.05rem; margin-top: 1px; }
.icon-check, .icon-cross { flex-shrink: 0; margin-top: 2px; }

/* 作答回饋高亮 */
.option-label.correct-ans { border-color: #2ecc71; background: #eafaf1; font-weight: bold; }
.option-label.wrong-ans { border-color: #e74c3c; background: #fdedec; text-decoration: line-through; color: #7f8c8d; }
.feedback { margin-top: 15px; padding: 12px; background: #fff3cd; border-radius: 8px; color: #856404; font-size: 0.95rem; }

/* 投票區塊 */
.input-label { display: block; margin-bottom: 8px; font-weight: bold; color: #34495e; }
.poll-input { width: 100%; padding: 12px; padding-right: 35px; border: 2px solid #e0e0e0; border-radius: 8px; font-size: 16px; margin-bottom: 0; box-sizing: border-box; }
.input-wrapper { position: relative; display: flex; align-items: center; margin-bottom: 15px; }
.clear-text-btn { position: absolute; right: 10px; background: transparent; border: none; color: #aaa; font-size: 1.2rem; cursor: pointer; transition: color 0.2s; padding: 0; }
.clear-text-btn:hover { color: #e74c3c; }
.poll-title { font-size: 1.4rem; color: #2c3e50; text-align: center; margin-bottom: 20px; word-break: break-word; line-height: 1.4; }
.poll-options { display: flex; flex-direction: column; gap: 12px; }
.vote-btn { width: 100%; padding: 14px; background: #ecf0f1; color: #2c3e50; border: none; border-radius: 8px; font-size: 1.1rem; cursor: pointer; text-align: left; transition: background 0.2s; font-weight: 500; }
.vote-btn:hover:not(:disabled) { background: #bdc3c7; }
.progress-container { margin-top: 8px; background: #ecf0f1; border-radius: 6px; height: 24px; position: relative; overflow: hidden; }
.progress-bar { height: 100%; background: #3498db; transition: width 0.8s cubic-bezier(0.4, 0, 0.2, 1); }
.vote-count { position: absolute; right: 10px; top: 50%; transform: translateY(-50%); font-size: 0.85rem; font-weight: bold; color: #2c3e50; z-index: 1; }

/* 通用按鈕 */
.submit-area { text-align: center; margin-top: 20px; }
.submit-btn { padding: 16px 40px; font-size: 1.2rem; font-weight: bold; background: linear-gradient(135deg, #2ecc71, #27ae60); color: white; border: none; border-radius: 12px; cursor: pointer; box-shadow: 0 4px 6px rgba(46, 204, 113, 0.3); transition: transform 0.1s; }
.submit-btn:active { transform: translateY(2px); }
.action-btn { padding: 10px 24px; font-size: 1rem; font-weight: bold; background: #3498db; color: white; border: none; border-radius: 8px; cursor: pointer; }
.action-btn.secondary { background: #95a5a6; }
.action-btn:hover { opacity: 0.9; }
.mt-15 { margin-top: 15px; }
.mt-20 { margin-top: 20px; }

/* 響應式設計 (RWD) */
@media (max-width: 600px) {
  h1 {
    font-size: 1.5rem;
  }
  .mode-switch {
    flex-direction: column;
  }
  .submit-btn {
    width: 100%;
  }
}
</style>