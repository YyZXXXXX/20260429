<template>
  <div class="interactive-tool">
    <h1>📊 課堂互動工具箱 - 小組競賽記分板</h1>

    <!-- p5.js 慶祝動畫容器 -->
    <div ref="p5Container" class="p5-canvas-container"></div>

    <!-- 小組設定區 -->
    <div class="settings card">
      <h2>👥 隊伍管理</h2>
      <div class="input-group">
        <input 
          v-model="newGroupName" 
          @keyup.enter="addGroup"
          placeholder="輸入新小組名稱，按 Enter 新增..." 
        />
        <button @click="addGroup">新增小組</button>
      </div>
    </div>

    <!-- 記分板展示區 -->
    <div class="scoreboard card">
      <h2 class="flex-between">
        <span>🏆 競賽排名</span>
        <button class="reset-btn" @click="resetAllScores" v-if="groups.length > 0" title="將所有分數歸零">↺ 分數清零</button>
      </h2>
      
      <div class="list-header" v-if="groups.length > 0">
        <span class="header-rank">排名 / 隊名</span>
        <span class="header-score">分數</span>
        <span class="header-controls">操作</span>
      </div>

      <!-- 使用 TransitionGroup 達成即時動態排序動畫 -->
      <TransitionGroup name="list" tag="div" class="group-list">
        <div 
          v-for="(group, index) in sortedGroups" 
          :key="group.id" 
          class="group-item"
          :class="{ 'is-first': group._rank === 1 && group.score > 0 && groups.length > 1 }"
        >
          <div class="group-info">
            <span class="rank" :class="{ 'gold': group._rank === 1 && group.score > 0 }">
              {{ (group._rank === 1 && group.score > 0) ? '👑 1' : group._rank }}
            </span>
            <span class="name">{{ group.name }}</span>
            <button class="remove-btn" @click="removeGroup(group.id)" title="移除此小組">×</button>
          </div>
          
          <Transition name="pop" mode="out-in">
            <div class="score-display" :key="group.score">
              {{ group.score }}
            </div>
          </Transition>
          
          <div class="controls">
            <button class="btn-minus" @click="updateScore(group.id, -1)">-1</button>
            <button class="btn-plus" @click="updateScore(group.id, 1)">+1</button>
            <button class="btn-plus-five" @click="updateScore(group.id, 5)">+5</button>
          </div>
        </div>
      </TransitionGroup>
      
      <div v-if="groups.length === 0" class="empty-text">目前沒有小組，請先新增小組名單。</div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, onUnmounted } from 'vue';

// 預設分組狀態
const groups = ref([]);
const newGroupName = ref('');
let nextId = 4; // 給予新小組唯一的 ID，確保動畫正確追蹤

const p5Container = ref(null);
let p5Instance = null;

// Local Storage：頁面載入時讀取名單
onMounted(() => {
  const saved = localStorage.getItem('toolbox-groups');
  if (saved) {
    groups.value = JSON.parse(saved);
    nextId = groups.value.length > 0 ? Math.max(...groups.value.map(g => g.id)) + 1 : 1;
  } else {
    groups.value = [
      { id: 1, name: '第一組', score: 0 },
      { id: 2, name: '第二組', score: 0 },
      { id: 3, name: '第三組', score: 0 }
    ];
  }

  // 動態載入 p5.js 並初始化紙屑噴發動畫
  const loadP5 = new Promise((resolve) => {
    if (window.p5) { resolve(window.p5); return; }
    const script = document.createElement('script');
    script.src = 'https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.9.0/p5.min.js';
    script.onload = () => resolve(window.p5);
    script.onerror = () => resolve(null); // 容斷處理，避免載入失敗時白畫面
    document.head.appendChild(script);
  });

  loadP5.then((p5) => {
    if (!p5 || !p5Container.value) return;
    
    let particles = [];
    const sketch = (p) => {
      p.setup = () => {
        const canvas = p.createCanvas(p.windowWidth, p.windowHeight);
        canvas.parent(p5Container.value);
        p.clear(); // 透明背景，不遮擋 UI
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
      
      // 對外暴露的噴發函式
      p.fireConfetti = () => {
        for (let i = 0; i < 80; i++) {
          particles.push(new Particle(p, p.random(p.width * 0.2, p.width * 0.8), p.height + 20));
        }
      };

      class Particle {
        constructor(pRef, x, y) {
          this.p = pRef;
          this.pos = this.p.createVector(x, y);
          this.vel = this.p.createVector(this.p.random(-6, 6), this.p.random(-15, -28));
          this.acc = this.p.createVector(0, 0.5);
          this.color = this.p.color(this.p.random(255), this.p.random(255), this.p.random(255));
          this.size = this.p.random(8, 16);
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

// Local Storage：當小組資料異動時自動儲存
watch(groups, (newGroups) => {
  localStorage.setItem('toolbox-groups', JSON.stringify(newGroups));
}, { deep: true });

// 即時排序：根據分數由高到低排序，並計算真實排名（處理同分並列情況）
const sortedGroups = computed(() => {
  const sorted = [...groups.value].sort((a, b) => {
    if (b.score !== a.score) return b.score - a.score;
    return a.id - b.id; // 分數相同時，依據 ID 排序以保持動畫與列表的穩定
  });
  
  let rank = 1;
  return sorted.map((group, index, arr) => {
    // 如果分數小於前一名，名次才遞增（這能讓同分的人保持同樣名次）
    if (index > 0 && group.score < arr[index - 1].score) rank = index + 1;
    return { ...group, _rank: rank };
  });
});

// --- 音效系統 (無須外部檔案，使用 Web Audio API) ---
let audioCtx = null;
const initAudio = () => {
  try {
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    if (audioCtx.state === 'suspended') audioCtx.resume();
  } catch (e) {}
};

const playScoreSound = (delta) => {
  try {
    initAudio();
    if (!audioCtx) return;
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    
    if (delta > 0) {
      osc.type = 'sine';
      osc.frequency.setValueAtTime(600, audioCtx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(1200, audioCtx.currentTime + 0.1);
    } else {
      osc.type = 'sawtooth';
      osc.frequency.setValueAtTime(300, audioCtx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(150, audioCtx.currentTime + 0.1);
    }
    
    gain.gain.setValueAtTime(0.1, audioCtx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.1);
    osc.start();
    osc.stop(audioCtx.currentTime + 0.1);
  } catch (e) {}
};

// 新增小組
const addGroup = () => {
  const name = newGroupName.value.trim();
  if (name) {
    groups.value.push({ id: nextId++, name, score: 0 });
    newGroupName.value = '';
  }
};

// 刪除小組
const removeGroup = (id) => {
  groups.value = groups.value.filter(g => g.id !== id);
};

// 所有分數清零防呆機制
const resetAllScores = () => {
  if (window.confirm('確定要將所有小組的分數歸零嗎？')) {
    groups.value.forEach(group => {
      group.score = 0;
    });
  }
};

// 更新分數
const updateScore = (id, delta) => {
  const group = groups.value.find(g => g.id === id);
  if (group) {
    group.score += delta;
    playScoreSound(delta);
    
    // 加分時觸發 p5 紙屑噴發
    if (delta > 0 && p5Instance && p5Instance.fireConfetti) {
      p5Instance.fireConfetti();
    }
  }
};

// 組件解除掛載時清理 p5 實例避免 memory leak
onUnmounted(() => { if (p5Instance) p5Instance.remove(); });
</script>

<style scoped>
.interactive-tool {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  max-width: 800px;
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

h1 { text-align: center; color: #2c3e50; margin-bottom: 30px; line-height: 1.4; word-break: break-word; }
.card {
  background: rgba(255, 255, 255, 0.85);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border-radius: 12px; padding: 24px; margin-bottom: 24px; 
  box-shadow: 0 8px 32px rgba(31, 38, 135, 0.07); 
  border: 1px solid rgba(255, 255, 255, 0.5);
}
h2 {
  margin-top: 0; font-size: 1.2rem; color: #34495e;
  border-bottom: 2px solid #eee; padding-bottom: 10px; margin-bottom: 15px;
}

.flex-between {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 10px;
}

.reset-btn {
  background-color: #95a5a6;
  color: white;
  border: none;
  padding: 6px 12px;
  border-radius: 6px;
  font-size: 0.85rem;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.2s, transform 0.1s;
}
.reset-btn:hover { background-color: #7f8c8d; }
.reset-btn:active { transform: translateY(2px); }

/* 輸入框樣式 */
.input-group { display: flex; gap: 10px; flex-wrap: wrap; }
.input-group input {
  flex: 1; padding: 12px; border: 2px solid #e0e0e0;
  border-radius: 8px; font-size: 16px; transition: border-color 0.3s;
}
.input-group input:focus { outline: none; border-color: #42b983; }
.input-group button {
  padding: 10px 24px; background-color: #42b983; color: white;
  border: none; border-radius: 8px; cursor: pointer; font-size: 16px; font-weight: bold;
}
.input-group button:hover { background-color: #33a06f; }

/* 記分板列表樣式 */
.list-header {
  display: flex; justify-content: space-between;
  padding: 0 16px 8px; font-weight: bold; color: #7f8c8d; font-size: 0.95rem;
  flex-wrap: wrap; gap: 5px;
}
.header-rank { flex: 1; }
.header-score { width: 80px; text-align: center; }
.header-controls { flex: 1; text-align: right; }

.group-list { position: relative; }
.group-item {
  display: flex; justify-content: space-between; align-items: center;
  background: rgba(255, 255, 255, 0.95); padding: 16px; margin-bottom: 12px;
  border-radius: 12px; border-left: 6px solid #bdc3c7;
  box-shadow: 0 4px 10px rgba(0,0,0,0.03); width: 100%; box-sizing: border-box;
  transition: all 0.3s ease;
  gap: 10px;
}
.group-item.is-first {
  border-left-color: #f1c40f; background: #fffdf5;
  /* 移除 transform 以避免與 Vue TransitionGroup 的 FLIP 移動動畫發生衝突導致閃爍 */
  box-shadow: 0 4px 12px rgba(241,196,15,0.2); z-index: 1;
}

.group-info { display: flex; align-items: center; gap: 12px; flex: 1; min-width: 0; }
.rank { font-size: 1.2rem; font-weight: bold; color: #7f8c8d; min-width: 45px; }
.rank.gold { color: #f39c12; }
.name { font-size: 1.25rem; font-weight: 600; word-break: break-word; line-height: 1.3; }
.remove-btn {
  background: none; border: none; color: #e74c3c; font-size: 1.2rem;
  cursor: pointer; opacity: 0.3; transition: opacity 0.2s; padding: 0 8px;
}
.remove-btn:hover { opacity: 1; }

.score-display {
  font-size: 2.2rem; font-weight: 900; color: #2c3e50;
  width: 80px; text-align: center;
}

.controls { display: flex; gap: 8px; flex: 1; justify-content: flex-end; flex-wrap: wrap; }
.controls button {
  padding: 10px 14px; font-size: 1rem; font-weight: bold; border: none;
  border-radius: 6px; cursor: pointer; color: white;
  transition: transform 0.1s, opacity 0.2s, box-shadow 0.1s; box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}
.controls button:active { transform: translateY(2px); box-shadow: none; }
.controls button:hover { opacity: 0.9; }
.btn-minus { background-color: #e74c3c; }
.btn-plus { background-color: #2ecc71; }
.btn-plus-five { background-color: #3498db; }

.empty-text { color: #999; font-style: italic; text-align: center; padding: 20px; }

/* Vue Transition Group 排序與進退場動畫 */
.list-move,
.list-enter-active,
.list-leave-active {
  transition: all 0.5s cubic-bezier(0.55, 0, 0.1, 1);
}
.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: scaleY(0.01) translate(30px, 0);
}
.list-leave-active {
  position: absolute;
}

/* 分數變更時的縮放動畫 (Pop effect) */
.pop-enter-active {
  animation: pop-in 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
.pop-leave-active {
  display: none; /* 隱藏舊分數，避免版面跳動 */
}

@keyframes pop-in {
  0% { transform: scale(0.5); opacity: 0; }
  60% { transform: scale(1.3); }
  100% { transform: scale(1); opacity: 1; }
}

/* 響應式設計 (RWD) */
@media (max-width: 600px) {
  h1 {
    font-size: 1.5rem;
  }
  .group-item {
    flex-direction: column;
    gap: 15px;
    align-items: flex-start;
  }
  .group-info {
    width: 100%;
  }
  .score-display {
    align-self: center;
  }
  .controls {
    width: 100%;
    justify-content: center;
  }
  .controls button { flex: 1; }
  .input-group input, .input-group button {
    width: 100%;
    flex: none;
  }
}
</style>