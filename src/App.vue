<script setup>
import { ref, computed, onMounted, onUnmounted } from "vue";

// 1. 洗程設定（將耗時改為分鐘，對齊你的實際需求：2h20m=140, 3h=180, 3h40m=220）
const WASH_MODES = [
  { name: "精緻 / P3", timeSpent: 140, label: "2 小時 20 分" },
  { name: "標準 / P2", timeSpent: 180, label: "3 小時" },
  { name: "節能 / ECO", timeSpent: 220, label: "3 小時 40 分" },
];

const selectedWashMode = ref(WASH_MODES[0].name);

// 2. 當前時間（即時更新，精確到毫秒級時間戳記以供計算）
const now = ref(new Date());
let timer = null;

onMounted(() => {
  timer = setInterval(() => {
    now.value = new Date();
  }, 1000);
});

onUnmounted(() => {
  clearInterval(timer);
});

// 格式化目前時間顯示
const currentDateTimeStr = computed(() => {
  return now.value.toLocaleTimeString("zh-TW", {
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
    hour12: false,
  });
});

// 3. 目標完成時間設定（預設最常用的 07:30 與 08:00 快捷鍵）
const PRESET_TIMES = [
  { label: "早上 07:30", hour: 7, minute: 30 },
  { label: "早上 08:00", hour: 8, minute: 0 },
];

const targetHour = ref(7);
const targetMinute = ref(30);

// 快速設定目標時間
const setPresetTime = (hour, minute) => {
  targetHour.value = hour;
  targetMinute.value = minute;
};

// 4. 核心計算邏輯 (透過 Computed 實現免點擊、即時更新)
// 4. 核心計算邏輯 (透過 Computed 實現免點擊、即時更新)
const result = computed(() => {
  const currentMode = WASH_MODES.find((m) => m.name === selectedWashMode.value);
  if (!currentMode)
    return { hours: 0, launchTime: "", clearBefore: "", status: "error" };

  // 建立目標時間的 Date 物件（預設為明天）
  const targetDate = new Date(now.value);
  targetDate.setHours(targetHour.value, targetMinute.value, 0, 0);

  // 如果設定的目標時間小於當前時間（例如現在是深夜 23:00，目標是 07:30），自動算作明天
  if (targetDate <= now.value) {
    targetDate.setDate(targetDate.getDate() + 1);
  }

  // 計算總可用時間 (毫秒)
  const totalAvailableMs = targetDate.getTime() - now.value.getTime();
  const totalAvailableMins = Math.floor(totalAvailableMs / (1000 * 60));

  // 扣除洗程時間 = 最大可等待（預約）的分鐘數
  const maxDelayMins = totalAvailableMins - currentMode.timeSpent;

  if (maxDelayMins < 0) {
    return {
      hours: 0,
      launchTime: "-",
      clearBefore: "糟了！現在立刻洗也會超時！",
      status: "danger",
    };
  }

  // 機器 step 為 1 小時，為了保證「在指定時間前洗完」，必須無條件捨去 (Math.floor)
  const computedHours = Math.floor(maxDelayMins / 60);
  const isOverMaxLimit = computedHours > 24;
  const delayHours = Math.min(computedHours, 24);

  // 1️⃣ 計算【預計啟動時間】點：現在時間 + 預約的小時數
  const actualLaunchDate = new Date(
    now.value.getTime() + delayHours * 60 * 60 * 1000
  );
  const launchTimeStr = actualLaunchDate.toLocaleTimeString("zh-TW", {
    hour: "2-digit",
    minute: "2-digit",
    hour12: false,
  });

  // 2️⃣ 計算【實際洗完時間】點：啟動時間 + 洗程耗時
  const actualFinishDate = new Date(
    actualLaunchDate.getTime() + currentMode.timeSpent * 60 * 1000
  );
  const finishTimeStr = actualFinishDate.toLocaleTimeString("zh-TW", {
    hour: "2-digit",
    minute: "2-digit",
    hour12: false,
  });

  // 如果超過 24 小時，調整提示文字
  const clearBeforeMessage = isOverMaxLimit
    ? `時間太充裕了！洗碗機最高支援預約 24 小時（設定 H 24 將於 ${finishTimeStr} 洗完）`
    : `預計於 ${finishTimeStr} 洗完（符合於 ${
        targetHour.value
      }:${targetMinute.value.toString().padStart(2, "0")} 前完成）`;

  return {
    hours: delayHours,
    launchTime: `預計於 ${launchTimeStr} 啟動`, // 新增回傳欄位
    clearBefore: clearBeforeMessage,
    status: isOverMaxLimit ? "danger" : "success",
  };
});
</script>

<template>
  <div class="bg-slate-900 min-h-screen">
    <div class="max-w-sm mx-auto p-5 text-slate-100 font-sans">
      <div class="text-center mb-6 border-b border-slate-800 pb-4">
        <h1 class="text-lg font-bold text-teal-400 mb-1">🍽️ 洗碗機預約助手</h1>
        <p class="text-xs text-slate-400">
          現在時間：<span class="font-mono text-slate-200">{{
            currentDateTimeStr
          }}</span>
        </p>
      </div>

      <div class="mb-5">
        <label
          class="block text-xs font-semibold text-slate-400 uppercase tracking-wider mb-2"
          >1. 選擇洗碗模式</label
        >
        <div class="grid grid-cols-1 gap-2">
          <button
            v-for="mode in WASH_MODES"
            :key="mode.name"
            @click="selectedWashMode = mode.name"
            :class="[
              'w-full p-3 text-left rounded-xl border transition-all duration-200 flex justify-between items-center',
              selectedWashMode === mode.name
                ? 'bg-teal-500/10 border-teal-500 text-teal-300 font-medium shadow-lg shadow-teal-500/5'
                : 'bg-slate-800/50 border-slate-700/60 text-slate-300 hover:bg-slate-800',
            ]"
          >
            <span>{{ mode.name }}</span>
            <span class="text-xs opacity-80">{{ mode.label }}</span>
          </button>
        </div>
      </div>

      <div class="mb-6">
        <label
          class="block text-xs font-semibold text-slate-400 uppercase tracking-wider mb-2"
          >2. 希望幾點前洗完？</label
        >

        <div class="grid grid-cols-2 gap-2 mb-3">
          <button
            v-for="preset in PRESET_TIMES"
            :key="preset.label"
            @click="setPresetTime(preset.hour, preset.minute)"
            :class="[
              'p-2 text-xs rounded-lg border transition-all text-center',
              targetHour === preset.hour && targetMinute === preset.minute
                ? 'bg-blue-500/10 border-blue-500 text-blue-300 font-medium'
                : 'bg-slate-800 border-slate-700 text-slate-400',
            ]"
          >
            {{ preset.label }}
          </button>
        </div>

        <div
          class="flex items-center justify-center gap-2 bg-slate-800/40 border border-slate-700/60 p-3 rounded-xl"
        >
          <div class="flex items-center gap-1 text-center">
            <input
              type="number"
              v-model.number="targetHour"
              min="0"
              max="23"
              class="w-12 bg-slate-800 text-center text-lg font-mono rounded p-1 border border-slate-700 focus:outline-none focus:border-teal-500"
            />
            <span class="text-xs text-slate-400">時</span>
          </div>
          <span class="text-lg font-bold text-slate-500">:</span>
          <div class="flex items-center gap-1 text-center">
            <input
              type="number"
              v-model.number="targetMinute"
              min="0"
              max="59"
              class="w-12 bg-slate-800 text-center text-lg font-mono rounded p-1 border border-slate-700 focus:outline-none focus:border-teal-500"
            />
            <span class="text-xs text-slate-400">分</span>
          </div>
        </div>
      </div>

      <div
        class="p-4 rounded-xl border text-center transition-all duration-300"
        :class="{
          'bg-teal-500/10 border-teal-500/30 text-teal-300':
            result.status === 'success',
          'bg-rose-500/10 border-rose-500/30 text-rose-300':
            result.status === 'danger',
        }"
      >
        <div class="text-xs uppercase tracking-wider opacity-70 mb-1">
          洗碗機預約面板請按
        </div>
        <div class="text-3xl font-black font-mono my-1 tracking-tight">
          H <span class="text-4xl text-amber-400">{{ result.hours }}</span>
        </div>
        <div class="text-xs font-medium text-slate-300 mt-2 font-sans">
          {{ result.launchTime }}
        </div>

        <div class="text-[11px] opacity-60 mt-1 font-sans">
          {{ result.clearBefore }}
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* 移除 Chrome/Safari/Edge/Firefox 的 input number 上下箭頭 */
input::-webkit-outer-spin-button,
input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}
</style>
