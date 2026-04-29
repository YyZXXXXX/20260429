<template>
  <div class="interactive-tool">
    <h1>📅 課堂互動工具箱 - 電子課表</h1>

    <div class="controls card">
      <div class="flex-between">
        <h2>⚙️ 課表設定</h2>
        <div class="actions">
          <button class="btn-add" @click="addRow">➕ 新增節次</button>
          <button class="btn-danger" @click="clearSchedule">🗑️ 清空所有內容</button>
          <button class="btn-reset" @click="resetToDefault">↻ 恢復預設格式</button>
        </div>
      </div>
    </div>

    <div class="schedule-container card">
      <div class="table-scroll">
        <table class="schedule-table">
          <thead>
            <tr>
              <th class="time-col">節次 / 時間</th>
              <th v-for="day in days" :key="day" class="day-col">{{ day }}</th>
              <th class="action-col">操作</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(row, rowIndex) in scheduleData" :key="row.id">
              <td class="time-col">
                <input v-model="row.periodName" placeholder="節次名稱" class="period-input" />
                <input v-model="row.timeRange" placeholder="時間 (例如 08:00-08:50)" class="time-input" />
              </td>
              <td v-for="(cell, colIndex) in row.days" :key="colIndex" class="cell-data">
                <div class="cell-content">
                  <input v-model="cell.subject" placeholder="科目" class="subject-input" />
                  <input v-model="cell.teacher" placeholder="老師" class="teacher-input" />
                </div>
              </td>
              <td class="action-col">
                <button class="remove-btn" @click="removeRow(rowIndex)" title="刪除此節次">✖</button>
              </td>
            </tr>
            <tr v-if="scheduleData.length === 0">
              <td colspan="7" class="empty-text">目前沒有任何節次，請點擊上方「新增節次」。</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted } from 'vue';

const days = ['星期一', '星期二', '星期三', '星期四', '星期五'];

// 預設的課表結構 (提供 8 節課 + 早自習與午休)
const getDefaultSchedule = () => [
  { id: Date.now() + 1, periodName: '早自習', timeRange: '07:30 - 08:00', days: Array.from({ length: 5 }, () => ({ subject: '', teacher: '' })) },
  { id: Date.now() + 2, periodName: '第 1 節', timeRange: '08:10 - 09:00', days: Array.from({ length: 5 }, () => ({ subject: '', teacher: '' })) },
  { id: Date.now() + 3, periodName: '第 2 節', timeRange: '09:10 - 10:00', days: Array.from({ length: 5 }, () => ({ subject: '', teacher: '' })) },
  { id: Date.now() + 4, periodName: '第 3 節', timeRange: '10:10 - 11:00', days: Array.from({ length: 5 }, () => ({ subject: '', teacher: '' })) },
  { id: Date.now() + 5, periodName: '第 4 節', timeRange: '11:10 - 12:00', days: Array.from({ length: 5 }, () => ({ subject: '', teacher: '' })) },
  { id: Date.now() + 6, periodName: '午休', timeRange: '12:00 - 13:00', days: Array.from({ length: 5 }, () => ({ subject: '午餐 / 休息', teacher: '' })) },
  { id: Date.now() + 7, periodName: '第 5 節', timeRange: '13:10 - 14:00', days: Array.from({ length: 5 }, () => ({ subject: '', teacher: '' })) },
  { id: Date.now() + 8, periodName: '第 6 節', timeRange: '14:10 - 15:00', days: Array.from({ length: 5 }, () => ({ subject: '', teacher: '' })) },
  { id: Date.now() + 9, periodName: '第 7 節', timeRange: '15:10 - 16:00', days: Array.from({ length: 5 }, () => ({ subject: '', teacher: '' })) },
  { id: Date.now() + 10, periodName: '第 8 節', timeRange: '16:10 - 17:00', days: Array.from({ length: 5 }, () => ({ subject: '', teacher: '' })) }
];

const scheduleData = ref([]);

// Local Storage：頁面載入時讀取課表資料
onMounted(() => {
  const savedSchedule = localStorage.getItem('toolbox-schedule');
  if (savedSchedule) {
    scheduleData.value = JSON.parse(savedSchedule);
  } else {
    scheduleData.value = getDefaultSchedule();
  }
});

// 當資料異動時自動儲存到 Local Storage
watch(scheduleData, (newVal) => {
  localStorage.setItem('toolbox-schedule', JSON.stringify(newVal));
}, { deep: true });

// 新增一行節次
const addRow = () => {
  scheduleData.value.push({
    id: Date.now(),
    periodName: `第 ${scheduleData.value.length + 1} 節`,
    timeRange: '',
    days: Array.from({ length: 5 }, () => ({ subject: '', teacher: '' }))
  });
};

// 刪除一行節次
const removeRow = (index) => {
  if (window.confirm('確定要刪除這個節次嗎？')) {
    scheduleData.value.splice(index, 1);
  }
};

// 清空所有格子內容 (保留節次列)
const clearSchedule = () => {
  if (window.confirm('確定要清空所有科目的內容嗎？(將保留節次與時間)')) {
    scheduleData.value.forEach(row => {
      row.days.forEach(day => {
        day.subject = '';
        day.teacher = '';
      });
    });
  }
};

// 恢復預設格式
const resetToDefault = () => {
  if (window.confirm('確定要捨棄目前的課表，恢復為預設的 8 節課格式嗎？')) {
    scheduleData.value = getDefaultSchedule();
  }
};
</script>

<style scoped>
.interactive-tool { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; max-width: 1000px; margin: 0 auto; padding: 20px; color: #333; }
h1 { text-align: center; color: #2c3e50; margin-bottom: 30px; line-height: 1.4; word-break: break-word; }
.card { background: rgba(255, 255, 255, 0.85); backdrop-filter: blur(12px); -webkit-backdrop-filter: blur(12px); border-radius: 12px; padding: 24px; margin-bottom: 24px; box-shadow: 0 8px 32px rgba(31, 38, 135, 0.07); border: 1px solid rgba(255, 255, 255, 0.5); }
h2 { margin-top: 0; font-size: 1.2rem; color: #34495e; margin-bottom: 0; }

.flex-between { display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 15px; border-bottom: 2px solid #eee; padding-bottom: 15px; }
.actions { display: flex; gap: 10px; flex-wrap: wrap; }
.actions button { padding: 8px 16px; font-weight: bold; border: none; border-radius: 8px; cursor: pointer; color: white; transition: background 0.2s, transform 0.1s; }
.actions button:active { transform: translateY(1px); }
.btn-add { background: #3498db; }
.btn-add:hover { background: #2980b9; }
.btn-danger { background: #e74c3c; }
.btn-danger:hover { background: #c0392b; }
.btn-reset { background: #95a5a6; }
.btn-reset:hover { background: #7f8c8d; }

/* 表格樣式 */
.table-scroll { overflow-x: auto; border-radius: 8px; border: 1px solid rgba(0,0,0,0.1); }
.schedule-table { width: 100%; border-collapse: collapse; min-width: 800px; background: rgba(255, 255, 255, 0.5); }
.schedule-table th { background: #34495e; color: white; padding: 12px; font-size: 1.1rem; text-align: center; border: 1px solid rgba(0,0,0,0.1); }
.schedule-table td { border: 1px solid rgba(0,0,0,0.1); vertical-align: middle; transition: background 0.2s; }
.schedule-table tbody tr:hover td { background: rgba(236, 240, 241, 0.5); }

.time-col { width: 140px; padding: 8px; background: rgba(236, 240, 241, 0.3); }
.day-col { width: 15%; }
.action-col { width: 50px; text-align: center; padding: 0; }

/* 儲存格內部輸入框 */
.cell-content { display: flex; flex-direction: column; height: 100%; padding: 4px; }
.schedule-table input { 
  width: 100%; border: none; background: transparent; text-align: center; 
  font-family: inherit; transition: background 0.2s; box-sizing: border-box; 
}
.schedule-table input:focus { outline: none; background: rgba(255, 255, 255, 0.8); border-radius: 4px; }

.period-input { font-weight: bold; font-size: 1.1rem; color: #2c3e50; margin-bottom: 4px; padding: 4px; }
.time-input { font-size: 0.85rem; color: #7f8c8d; padding: 4px; }

.subject-input { font-weight: bold; font-size: 1.05rem; color: #2980b9; margin-bottom: 2px; padding: 8px 4px; }
.teacher-input { font-size: 0.9rem; color: #16a085; padding: 4px; }

/* 刪除按鈕 */
.remove-btn { background: none; border: none; color: #e74c3c; cursor: pointer; font-size: 1.2rem; opacity: 0.5; transition: opacity 0.2s; width: 100%; height: 100%; padding: 10px 0; }
.remove-btn:hover { opacity: 1; }

.empty-text { text-align: center; padding: 30px; color: #7f8c8d; font-style: italic; }

/* =========================================
   響應式設計 (RWD)：手機版表格轉為「卡片式排列」
   ========================================= */
@media (max-width: 768px) {
  .table-scroll {
    border: none;
    overflow-x: hidden; /* 取消橫向捲軸 */
  }
  .schedule-table {
    min-width: 100%;
    background: transparent;
  }
  .schedule-table thead {
    display: none; /* 在手機上隱藏上方表頭 */
  }
  .schedule-table tbody tr {
    display: block;
    margin-bottom: 20px;
    border: 1px solid rgba(0, 0, 0, 0.1);
    border-radius: 12px;
    overflow: hidden;
    background: rgba(255, 255, 255, 0.6);
    box-shadow: 0 4px 10px rgba(0, 0, 0, 0.05);
  }
  .schedule-table td {
    display: block;
    width: 100%;
    border: none;
    border-bottom: 1px solid rgba(0, 0, 0, 0.05);
    box-sizing: border-box;
  }
  .schedule-table td:last-child { border-bottom: none; }
  
  .time-col {
    width: 100%;
    background: rgba(0, 0, 0, 0.05);
    padding: 15px 10px;
  }
  
  .cell-data {
    position: relative;
    padding: 12px 12px 12px 90px;
    min-height: 50px;
  }
  /* 利用虛擬元素在手機版自動補上星期標題 */
  .cell-data::before {
    position: absolute; left: 15px; top: 50%; transform: translateY(-50%);
    font-weight: bold; color: #7f8c8d; font-size: 1rem;
  }
  .schedule-table td:nth-child(2)::before { content: '星期一'; }
  .schedule-table td:nth-child(3)::before { content: '星期二'; }
  .schedule-table td:nth-child(4)::before { content: '星期三'; }
  .schedule-table td:nth-child(5)::before { content: '星期四'; }
  .schedule-table td:nth-child(6)::before { content: '星期五'; }

  .cell-content { flex-direction: row; align-items: center; gap: 10px; }
  .schedule-table input { text-align: left; }
  .period-input, .time-input { text-align: center; }
  .subject-input { margin-bottom: 0; flex: 2; font-size: 1.05rem; }
  .teacher-input { flex: 1; }
  
  .action-col { width: 100%; background: rgba(231, 76, 60, 0.05); }
  .remove-btn { padding: 12px 0; font-weight: bold; }
  .remove-btn::after { content: ' 刪除此節次'; font-size: 1rem; }
}
</style>