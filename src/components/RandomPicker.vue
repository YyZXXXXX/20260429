<template>
  <div class="interactive-tool">
    <h1>🎲 課堂互動工具箱 - 隨機抽籤機</h1>

    <!-- p5.js 慶祝動畫容器 -->
    <div ref="p5Container" class="p5-canvas-container"></div>

    <!-- 抽籤模式切換 -->
    <div class="mode-switch card">
      <button :class="{ active: drawMode === 'individual' }" @click="drawMode = 'individual'">👤 個人抽籤</button>
      <button :class="{ active: drawMode === 'group' }" @click="drawMode = 'group'">👥 小組抽籤</button>
    </div>

    <!-- 名單設定區 -->
    <div class="settings card">
      <!-- 個人名單設定 -->
      <template v-if="drawMode === 'individual'">
        <h2>📋 個人名單設定</h2>
        <div class="input-group">
          <input 
            v-model="newParticipant" 
            @keyup.enter="addParticipant"
            placeholder="輸入學生姓名，按 Enter 新增..." 
          />
          <button @click="addParticipant">新增</button>
        </div>
        
        <div class="participant-list">
          <span v-for="(person, index) in participants" :key="index" class="badge">
            {{ person }}
            <button class="remove-btn" @click="removeParticipant(index)">×</button>
          </span>
          <span v-if="participants.length === 0" class="empty-text">目前名單為空，請先新增名單。</span>
        </div>
      </template>

      <!-- 小組名單設定 -->
      <template v-else>
        <div class="flex-between" style="margin-bottom: 15px;">
          <h2 style="margin-bottom: 0; border: none; padding: 0;">👥 小組抽籤池</h2>
          <button class="sync-btn" @click="syncGroups" title="從記分板載入最新小組">🔄 同步記分板</button>
        </div>
        
        <div class="participant-list">
          <span v-for="(group, index) in groupPool" :key="index" class="badge group-badge">
            {{ group }}
            <button class="remove-btn" @click="groupPool.splice(index, 1)">×</button>
          </span>
          <span v-if="groupPool.length === 0" class="empty-text">目前抽籤池為空，請點擊「同步記分板」載入。</span>
        </div>
      </template>
      
      <div class="options">
        <label>
          <input type="checkbox" v-model="excludeAfterDraw" />
          ✅ 抽中後從名單中排除（避免重複中獎）
        </label>
      </div>
    </div>

    <!-- 抽籤視覺區 -->
    <div class="drawing-area card">
      <div 
        class="display-screen" 
        :class="{ 'is-drawing': isDrawing, 'has-winner': !isDrawing && drawnWinner }"
      >
        <span class="name-display">{{ currentName || '等待抽籤...' }}</span>
      </div>
      
      <button 
        class="draw-btn" 
        :disabled="isDrawing || (drawMode === 'individual' ? participants.length === 0 : groupPool.length === 0)" 
        @click="startDraw"
      >
        {{ isDrawing ? '🔄 抽籤中...' : '🎉 開始抽籤' }}
      </button>
    </div>

    <!-- 中獎紀錄區 -->
    <div class="results card" v-if="winners.length > 0">
      <div class="flex-between">
        <h2>🏆 中獎紀錄</h2>
        <button class="remove-btn" @click="clearWinners" title="清空紀錄">🗑️</button>
      </div>
      <ul>
        <li v-for="(winner, index) in winners" :key="index">
          <span class="draw-number">第 {{ index + 1 }} 抽</span> {{ winner }}
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import { ref, onUnmounted, watch, onMounted } from 'vue';

// 狀態管理
const participants = ref([]); 
const newParticipant = ref('');
const currentName = ref('');
const isDrawing = ref(false);
const drawnWinner = ref('');
const winners = ref([]);
const excludeAfterDraw = ref(true);

const drawMode = ref('individual'); // 'individual' 或 'group'
const groupPool = ref([]);

let drawInterval = null;

const p5Container = ref(null);
let p5Instance = null;

// --- 音效系統 ---
let audioCtx = null;
const initAudio = () => {
  try {
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    if (audioCtx.state === 'suspended') audioCtx.resume();
  } catch (e) {}
};

const playTickSound = () => {
  try {
    if (!audioCtx) return;
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.connect(gain); gain.connect(audioCtx.destination);
    osc.type = 'triangle'; osc.frequency.setValueAtTime(800, audioCtx.currentTime);
    gain.gain.setValueAtTime(0.05, audioCtx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.05);
    osc.start(); osc.stop(audioCtx.currentTime + 0.05);
  } catch(e) {}
};

const playTadaSound = () => {
  try {
    if (!audioCtx) return;
    [523.25, 659.25, 783.99].forEach(freq => { // C大調和弦
      const osc = audioCtx.createOscillator();
      const gain = audioCtx.createGain();
      osc.connect(gain); gain.connect(audioCtx.destination);
      osc.type = 'sine'; osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
      gain.gain.setValueAtTime(0.15, audioCtx.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 1.5);
      osc.start(); osc.stop(audioCtx.currentTime + 1.5);
    });
  } catch(e) {}
};

// 從記分板同步小組名單
const syncGroups = () => {
  const savedGroups = localStorage.getItem('toolbox-groups');
  if (savedGroups) {
    groupPool.value = JSON.parse(savedGroups).map(g => g.name);
  }
};

// Local Storage：頁面載入時讀取名單與紀錄
onMounted(() => {
  const savedParticipants = localStorage.getItem('toolbox-participants');
  if (savedParticipants) {
    participants.value = JSON.parse(savedParticipants);
  } else {
    participants.value = ['小明', '小華', '小美', '大雄', '胖虎', '靜香']; // 預設
  }
  
  const savedWinners = localStorage.getItem('toolbox-winners');
  if (savedWinners) winners.value = JSON.parse(savedWinners);
  
  const savedGroupPool = localStorage.getItem('toolbox-group-pool');
  if (savedGroupPool) {
    groupPool.value = JSON.parse(savedGroupPool);
  } else {
    syncGroups();
  }

  // 動態載入 p5.js 並初始化紙屑噴發動畫
  const loadP5 = new Promise((resolve) => {
    if (window.p5) { resolve(window.p5); return; }
    const script = document.createElement('script');
    script.src = 'https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.9.0/p5.min.js';
    script.onload = () => resolve(window.p5);
    script.onerror = () => resolve(null);
    document.head.appendChild(script);
  });

  loadP5.then((p5) => {
    if (!p5 || !p5Container.value) return;
    
    let particles = [];
    const sketch = (p) => {
      p.setup = () => {
        const canvas = p.createCanvas(p.windowWidth, p.windowHeight);
        canvas.parent(p5Container.value);
        p.clear();
      };
      
      p.draw = () => {
        p.clear();
        for (let i = particles.length - 1; i >= 0; i--) {
          particles[i].update();
          particles[i].display();
          if (particles[i].isDead()) particles.splice(i, 1);
        }
      };
      
      p.windowResized = () => p.resizeCanvas(p.windowWidth, p.windowHeight);
      
      p.fireConfetti = () => {
        for (let i = 0; i < 120; i++) {
          // 抽籤的中獎動畫，從畫面中心向外噴發
          particles.push(new Particle(p, p.width / 2, p.height / 2));
        }
      };

      class Particle {
        constructor(pRef, x, y) {
          this.p = pRef;
          this.pos = this.p.createVector(x, y);
          this.vel = this.p.createVector(this.p.random(-12, 12), this.p.random(-15, 15));
          this.acc = this.p.createVector(0, 0.4); // 重力
          this.color = this.p.color(this.p.random(255), this.p.random(255), this.p.random(255));
          this.size = this.p.random(8, 18);
          this.rot = this.p.random(this.p.TWO_PI);
          this.rotSpeed = this.p.random(-0.2, 0.2);
          this.life = 255;
        }
        update() {
          this.vel.add(this.acc); this.pos.add(this.vel);
          this.rot += this.rotSpeed; this.life -= 3;
        }
        display() {
          this.p.push(); this.p.translate(this.pos.x, this.pos.y); this.p.rotate(this.rot);
          this.p.fill(this.color.levels[0], this.color.levels[1], this.color.levels[2], this.life);
          this.p.noStroke(); this.p.rectMode(this.p.CENTER);
          this.p.rect(0, 0, this.size, this.size / 1.5);
          this.p.pop();
        }
        isDead() { return this.life <= 0 || this.pos.y > this.p.height; }
      }
    };
    p5Instance = new p5(sketch);
  });
});

watch(participants, (newVal) => localStorage.setItem('toolbox-participants', JSON.stringify(newVal)), { deep: true });
watch(winners, (newVal) => localStorage.setItem('toolbox-winners', JSON.stringify(newVal)), { deep: true });
watch(groupPool, (newVal) => localStorage.setItem('toolbox-group-pool', JSON.stringify(newVal)), { deep: true });

// 新增名單
const addParticipant = () => {
  const name = newParticipant.value.trim();
  if (name && !participants.value.includes(name)) {
    participants.value.push(name);
    newParticipant.value = '';
  }
};

// 移除單一名單
const removeParticipant = (index) => {
  participants.value.splice(index, 1);
};

// 清空中獎紀錄
const clearWinners = () => {
  winners.value = [];
};

// 開始抽籤 (包含動態跳轉效果)
const startDraw = () => {
  const pool = drawMode.value === 'individual' ? participants.value : groupPool.value;
  if (pool.length === 0) return;
  
  initAudio(); // 啟動音訊環境
  isDrawing.value = true;
  drawnWinner.value = '';
  
  let counter = 0;
  const speed = 60; // 文字跳轉切換速度 (毫秒)
  const duration = 2500; // 抽籤總時長 (毫秒)
  
  // 使用 setInterval 達成快速跳轉的視覺效果
  drawInterval = setInterval(() => {
    const randomIndex = Math.floor(Math.random() * pool.length);
    currentName.value = pool[randomIndex];
    playTickSound();
    counter += speed;
    
    // 時間到，結束抽籤
    if (counter >= duration) {
      clearInterval(drawInterval);
      stopDraw();
    }
  }, speed);
};

// 結束抽籤並處理中獎邏輯
const stopDraw = () => {
  isDrawing.value = false;
  const pool = drawMode.value === 'individual' ? participants.value : groupPool.value;
  
  // 隨機決定最終中獎者
  const finalIndex = Math.floor(Math.random() * pool.length);
  drawnWinner.value = pool[finalIndex];
  currentName.value = drawnWinner.value;
  
  // 記錄至中獎清單
  winners.value.push(drawnWinner.value);
  playTadaSound();
  
  // 觸發 p5 慶祝紙屑動畫
  if (p5Instance && p5Instance.fireConfetti) {
    p5Instance.fireConfetti();
  }
  
  // 根據設定決定是否從原始名單中剔除該名學生
  if (excludeAfterDraw.value) {
    pool.splice(finalIndex, 1);
  }
};

// 修復錯誤：確保元件被銷毀時清除 Interval 以避免 Memory Leak
onUnmounted(() => {
  if (drawInterval) {
    clearInterval(drawInterval);
  }
  if (p5Instance) p5Instance.remove();
});
</script>

<style scoped>
.interactive-tool {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  max-width: 700px;
  margin: 0 auto;
  padding: 20px;
  color: #333;
}

/* p5 畫布容器，固定於最上層且不阻擋點擊 */
.p5-canvas-container {
  position: fixed; top: 0; left: 0;
  width: 100vw; height: 100vh;
  pointer-events: none; z-index: 9999;
}

/* 模式切換 */
.mode-switch { display: flex; gap: 15px; padding: 15px; justify-content: center; flex-wrap: wrap; }
.mode-switch button { padding: 12px 24px; font-size: 1.1rem; font-weight: bold; border: 2px solid #e0e0e0; background: rgba(255,255,255,0.5); color: #7f8c8d; border-radius: 8px; cursor: pointer; transition: all 0.3s; flex: 1;}
.mode-switch button.active { border-color: #4facfe; background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%); color: #fff; box-shadow: 0 4px 15px rgba(79, 172, 254, 0.4); }

h1 {
  text-align: center;
  color: #2c3e50;
  margin-bottom: 30px;
  line-height: 1.4;
  word-break: break-word;
}

.card {
  background: rgba(255, 255, 255, 0.85);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border-radius: 12px; padding: 24px; margin-bottom: 24px; 
  box-shadow: 0 8px 32px rgba(31, 38, 135, 0.07); 
  border: 1px solid rgba(255, 255, 255, 0.5);
}

h2 {
  margin-top: 0;
  font-size: 1.2rem;
  color: #34495e;
  border-bottom: 2px solid #eee;
  padding-bottom: 10px;
  margin-bottom: 15px;
}

/* 輸入框設計 */
.input-group {
  display: flex;
  gap: 10px;
  margin-bottom: 15px;
  flex-wrap: wrap;
}

.input-group input {
  flex: 1;
  padding: 12px;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  font-size: 16px;
  transition: border-color 0.3s;
}

.input-group input:focus {
  outline: none;
  border-color: #42b983;
}

.input-group button {
  padding: 10px 24px;
  background-color: #42b983;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
}

.input-group button:hover {
  background-color: #33a06f;
}

/* 名單標籤 */
.participant-list {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-bottom: 20px;
  min-height: 40px;
}

.badge {
  background: #e8f5e9;
  color: #2e7d32;
  padding: 8px 14px;
  border-radius: 20px;
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 500;
}

.remove-btn {
  background: none;
  border: none;
  color: #2e7d32;
  cursor: pointer;
  font-weight: bold;
  font-size: 16px;
  padding: 0;
  display: flex;
  align-items: center;
}

.group-badge { background: #e3f2fd; color: #1565c0; border: 1px solid #bbdefb; }
.group-badge .remove-btn { color: #1565c0; }

.empty-text {
  color: #999;
  font-style: italic;
  padding-top: 8px;
}

/* 選項設定 */
.options {
  font-size: 16px;
  color: #555;
  background: #f8f9fa;
  padding: 12px;
  border-radius: 8px;
}

.options label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  word-break: break-word;
  line-height: 1.4;
}

/* 抽籤視覺展示區 */
.display-screen {
  height: 180px;
  background: #34495e;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  margin-bottom: 20px;
  transition: all 0.3s ease;
  box-shadow: inset 0 4px 10px rgba(0,0,0,0.3);
}

.name-display {
  font-size: 3rem;
  font-weight: 900;
  color: white;
  letter-spacing: 2px;
  text-align: center;
  padding: 0 10px;
  word-break: break-word;
  line-height: 1.2;
}

/* 抽籤時的動畫 (垂直滾動感模擬) */
.is-drawing .name-display {
  animation: roll 0.12s linear infinite;
  color: #ecf0f1;
  opacity: 0.8;
}
@keyframes roll {
  0% { transform: translateY(-25px); opacity: 0.5; }
  50% { opacity: 1; }
  100% { transform: translateY(25px); opacity: 0.5; }
}

/* 抽出得主時的動畫 (高亮與彈出) */
.has-winner {
  background: linear-gradient(135deg, #f6d365 0%, #fda085 100%);
  animation: popOut 0.8s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  box-shadow: 0 0 30px rgba(241, 196, 15, 0.6);
}

.has-winner .name-display {
  color: #ffffff;
  text-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

.draw-btn {
  width: 100%;
  font-size: 1.2rem;
  font-weight: bold;
  padding: 16px;
  background: linear-gradient(135deg, #3498db, #2980b9);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  box-shadow: 0 4px 6px rgba(52, 152, 219, 0.3);
  transition: transform 0.1s, box-shadow 0.1s;
}

.draw-btn:active:not(:disabled) {
  transform: translateY(2px);
  box-shadow: 0 2px 4px rgba(52, 152, 219, 0.3);
}

.draw-btn:disabled {
  background: #bdc3c7;
  box-shadow: none;
  cursor: not-allowed;
}

/* 中獎清單 */
.results ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.results li {
  padding: 12px 16px;
  border-bottom: 1px solid #f0f0f0;
  font-size: 1.1rem;
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
  word-break: break-word;
}

.results li:last-child {
  border-bottom: none;
}

.draw-number {
  background: #e74c3c;
  color: white;
  font-size: 0.85rem;
  padding: 4px 8px;
  border-radius: 4px;
  font-weight: bold;
}

.flex-between {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 10px;
}

.sync-btn {
  background-color: #3498db; color: white; border: none; padding: 8px 14px;
  border-radius: 8px; font-size: 0.95rem; cursor: pointer; font-weight: bold;
  transition: background-color 0.2s, transform 0.1s; box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}
.sync-btn:hover { background-color: #2980b9; }
.sync-btn:active { transform: translateY(2px); box-shadow: none; }

@keyframes popOut {
  0% { transform: scale(0.9); }
  50% { transform: scale(1.05); }
  100% { transform: scale(1); }
}

/* 響應式設計 (RWD) */
@media (max-width: 600px) {
  h1 {
    font-size: 1.5rem;
  }
  .name-display {
    font-size: 2.2rem;
  }
  .mode-switch {
    flex-direction: column;
  }
  .input-group input, .input-group button {
    width: 100%;
    flex: none;
  }
}
</style>