<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
import DeviceStats from './components/DeviceStats.vue';
import DeviceList from './components/DeviceList.vue';
import config from './config.js'
import GiscusComments from './components/GiscusComments.vue';
import Footer from "./components/Footer.vue";
import DateSelector from "./components/DateSelector.vue";
import { pageConfig } from "./composables/pageConfig.js"
import Header from "./components/Header.vue";

// ===== 配置管理 =====
const { pageConfigs, isLoading: configLoading, fetchFlags } = pageConfig();

// 整体加载状态
const isPageReady = ref(false);

const showDeviceCount = computed(() =>
    isPageReady.value && (pageConfigs.value?.config?.WEB_DEVICE_COUNT ?? false)
);

const showComments = computed(() =>
    isPageReady.value && (pageConfigs.value?.config?.WEB_COMMENT ?? true)
);

const showAISummary = computed(() =>
    isPageReady.value && (pageConfigs.value?.config?.WEB_AI_SUMMARY ?? true)
);

const serverTzOffset = computed(() =>
    pageConfigs.value?.tzOffset ?? 8
);

const showSummary = computed(() =>
    isPageReady.value && (pageConfigs.value?.config?.WEB_SUMMARY ?? true)
);

const hasRealDevices = computed(() => {
  return devices.value.some(d => d.device !== 'summary');
});

// ===== 其他状态 =====
const API_BASE = config.API_BASE;
const devices = ref([]);
const selectedDevice = ref(null);
const clientIp = ref('获取中...');
const refreshInterval = ref(null);
const isRefreshing = ref(false); // 添加刷新状态
const toast = ref({
  show: false,
  message: '',
  type: 'error'
});

// 背景配置
const backgroundOptions = [
  {
    label: '晨曦渐变',
    value: 'linear-gradient(135deg, #d8e8ff 0%, #f7e6ff 45%, #d4f1ff 100%)'
  },
  {
    label: '极光蓝紫',
    value: 'linear-gradient(135deg, #2F80ED 0%, #6C5CE7 50%, #a88bff 100%)'
  },
  {
    label: '薄荷清爽',
    value: 'linear-gradient(135deg, #d8fff2 0%, #b3ffe0 40%, #c4e1ff 100%)'
  },
  {
    label: '星空深色',
    value: 'linear-gradient(135deg, #0b1224 0%, #111827 50%, #1f2937 100%)'
  }
];

const selectedBackground = ref(backgroundOptions[0].value);
const customBackground = ref('');

const backgroundStyle = computed(() => ({
  backgroundImage: selectedBackground.value,
  backgroundSize: 'cover',
  backgroundRepeat: 'no-repeat',
  backgroundAttachment: 'fixed',
  backgroundPosition: 'center'
}));

const OVERVIEW_DEVICE = {
  device: 'summary',
  currentApp: '不告诉你'
};

// 获取本地日期字符串
const getLocalDateString = (date = new Date()) => {
  const offset = date.getTimezoneOffset() * 60000;
  const localDate = new Date(date - offset);
  return localDate.toISOString().split('T')[0];
};

const selectedDate = ref(getLocalDateString());
const statsType = ref('daily');
const timeOffset = ref(0);
const stats = ref(null);

// Toast提示
const showToast = (message, type = 'error') => {
  toast.value = { show: true, message, type };
  setTimeout(() => {
    toast.value.show = false;
  }, 3000);
};

// 设置背景
const setBackground = (value) => {
  selectedBackground.value = value;
};

const applyCustomBackground = () => {
  if (!customBackground.value.trim()) return;

  const input = customBackground.value.trim();
  // 如果用户直接输入了 url 地址，自动补全 url() 格式
  if (input.startsWith('http')) {
    setBackground(`url('${input}')`);
  } else {
    setBackground(input);
  }
};

const resetBackground = () => {
  selectedBackground.value = backgroundOptions[0].value;
  customBackground.value = '';
};

// 获取客户端IP
const fetchClientIp = async () => {
  try {
    const response = await fetch(`${API_BASE}/ip`);
    const data = await response.json();
    clientIp.value = data.ip || '未知';
  } catch (err) {
    clientIp.value = '获取失败';
    console.error('获取IP地址失败:', err);
  }
};

// 获取设备基础信息
const fetchDevices = async () => {
  try {
    const response = await fetch(`${API_BASE}/devices`);
    if (!response.ok) {
      throw new Error(`请求失败: ${response.status}`);
    }
    const realDevices = await response.json();

    // 根据 WEB_SUMMARY 配置决定是否添加 summary 虚拟设备
    if (showSummary.value) {
      devices.value = [OVERVIEW_DEVICE, ...realDevices];
    } else {
      devices.value = realDevices;
    }

    if (devices.value.length > 0 && !selectedDevice.value) {
      selectedDevice.value = devices.value[0].device;
    }
  } catch (err) {
    showToast(`获取设备列表失败: ${err.message}`);
    console.error('获取设备列表错误:', err);
    devices.value = [];
  }
};

// 组件刷新key
const statsKey = ref(0);

// 刷新统计 - 不清空设备，使用key强制刷新组件
const refreshStats = () => {
  if (selectedDevice.value) {
    isRefreshing.value = true;
    // 通过改变key强制重新渲染组件
    statsKey.value++;
    // 延迟重置刷新状态
    setTimeout(() => {
      isRefreshing.value = false;
    }, 500);
  }
};

// 设置自动刷新
const setupAutoRefresh = () => {
  if (refreshInterval.value) {
    clearInterval(refreshInterval.value);
  }
  refreshInterval.value = setInterval(() => {
    if (selectedDevice.value) {
      fetchDevices();
    }
  }, 30000);
};

// 获取日期范围文本
const getDateRangeText = () => {
  if (!stats.value?.dateRange) return '';
  const { start, end } = stats.value.dateRange;
  return start === end ? start : `${start} 至 ${end}`;
};

// 处理统计数据更新
const handleStatsUpdate = (newStats) => {
  stats.value = newStats;
};

// 获取选中设备的信息
const getSelectedDevice = () => {
  if (!selectedDevice.value) return null;
  return devices.value.find(device => device.device === selectedDevice.value) || null;
};

// ===== 生命周期 =====
onMounted(async () => {
  try {
    await fetchFlags();
    // 标记页面准备就绪
    isPageReady.value = true;

    // 并行加载其他数据
    await Promise.all([
      fetchDevices(),
      fetchClientIp()
    ]);
  } catch (err) {
    console.error('❌ 初始化失败:', err);
    // 即使失败也标记为准备就绪，使用默认配置
    isPageReady.value = true;
  }

  setupAutoRefresh();
});

onUnmounted(() => {
  if (refreshInterval.value) {
    clearInterval(refreshInterval.value);
  }
});
</script>

<template>
  <!-- Toast 提示 -->
  <div v-if="toast.show" class="fixed top-4 right-4 z-50 w-auto transition-all duration-300">
    <div class="px-4 py-3 rounded-lg shadow-lg" :class="{
        'bg-red-50 border border-red-200 text-red-700': toast.type === 'error',
        'bg-green-50 border border-green-200 text-green-700': toast.type === 'success'
      }">
      <div class="flex items-start">
        <svg v-if="toast.type === 'error'" class="h-5 w-5 mr-2 mt-0.5 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path>
        </svg>
        <svg v-else class="h-5 w-5 mr-2 mt-0.5 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"></path>
        </svg>
        <span class="text-sm">{{ toast.message }}</span>
      </div>
    </div>
  </div>

  <!--  加载骨架屏 -->
  <div v-if="!isPageReady" class="bg-gray-100 min-h-screen rounded-lg dark:bg-[#1e2022]">
    <div class="max-w-7xl mx-auto px-4">
      <!-- 标题骨架 -->
      <div class="h-12 bg-gray-300 dark:bg-gray-700 rounded animate-pulse mx-auto w-96 mt-8 mb-8"></div>

      <div class="flex flex-col lg:flex-row gap-6 pb-6">
        <!-- 左侧骨架 -->
        <div class="space-y-6 w-full lg:w-96">
          <div class="h-32 bg-gray-300 dark:bg-gray-700 rounded animate-pulse"></div>
          <div class="h-64 bg-gray-300 dark:bg-gray-700 rounded animate-pulse"></div>
          <div class="h-48 bg-gray-300 dark:bg-gray-700 rounded animate-pulse"></div>
        </div>

        <!-- 右侧骨架 -->
        <div class="flex-1">
          <div class="h-96 bg-gray-300 dark:bg-gray-700 rounded animate-pulse"></div>
        </div>
      </div>
    </div>
  </div>

  <!-- 实际内容：只在配置加载完成后显示 -->
  <div v-else class="min-h-screen relative overflow-hidden" :style="backgroundStyle">
    <div class="absolute inset-0 bg-gradient-to-br from-white/50 via-white/20 to-blue-200/30 dark:from-slate-950/70 dark:via-slate-900/70 dark:to-indigo-950/60"></div>
    <div class="max-w-7xl mx-auto px-4 relative z-10 pb-10">
      <Header></Header>

      <!-- 背景自定义 -->
      <div class="glass-card p-4 md:p-5 mb-6 border border-white/30 dark:border-white/10">
        <div class="flex flex-col md:flex-row md:items-center md:justify-between gap-4">
          <div class="space-y-1">
            <p class="text-xs uppercase tracking-[0.08em] text-slate-500 dark:text-slate-300">界面风格</p>
            <h2 class="text-lg font-semibold text-slate-800 dark:text-slate-100">自定义背景</h2>
            <p class="text-sm text-slate-600 dark:text-slate-300">选择喜欢的底图，卡片会半透明并带有模糊效果。</p>
          </div>
          <div class="flex flex-wrap gap-2 justify-end">
            <button
                v-for="opt in backgroundOptions"
                :key="opt.label"
                @click="setBackground(opt.value)"
                class="relative w-28 h-16 rounded-xl overflow-hidden border border-white/40 dark:border-white/10 transition-transform hover:-translate-y-0.5 focus:outline-none focus:ring-2 focus:ring-white/60"
                :aria-label="`应用${opt.label}`"
                :style="{ backgroundImage: opt.value, backgroundSize: 'cover', backgroundPosition: 'center' }"
            >
              <span class="absolute inset-0 bg-white/15 dark:bg-black/30"></span>
              <span class="relative text-xs font-semibold text-white drop-shadow-md">{{ opt.label }}</span>
            </button>
          </div>
        </div>

        <div class="mt-4 flex flex-col sm:flex-row gap-3">
          <input
              v-model="customBackground"
              type="text"
              class="w-full rounded-lg glass-input px-4 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-200/80 dark:focus:ring-indigo-500/50"
              placeholder="输入图片链接或 CSS 渐变，例如 url(https://...) 或 linear-gradient(...)"
          >
          <div class="flex gap-2 sm:w-auto">
            <button @click="applyCustomBackground" class="px-4 py-2 rounded-lg bg-blue-600 text-white font-semibold shadow-md hover:bg-blue-700 transition-colors whitespace-nowrap">应用背景</button>
            <button @click="resetBackground" class="px-4 py-2 rounded-lg border border-white/50 text-slate-700 dark:text-slate-200 dark:border-white/10 hover:bg-white/40 dark:hover:bg-white/5 transition-colors whitespace-nowrap">重置</button>
          </div>
        </div>
      </div>

      <div class="flex flex-col lg:flex-row gap-6 pb-6">
        <!-- 左侧模块区 -->
        <div class="space-y-6">
          <!-- 设备统计卡片 -->
          <div v-if="showDeviceCount" class="glass-card p-6">
            <div class="grid grid-cols-2 gap-4">
              <div class="glass-tile p-4 text-center">
                <h3 class="text-gray-500 text-sm font-medium flex items-center justify-center gap-1">
                  <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4.354a4 4 0 110 5.292M15 21H3v-1a6 6 0 0112 0v1zm0 0h6v-1a6 6 0 00-9-5.197M13 7a4 4 0 11-8 0 4 4 0 018 0z" />
                  </svg>
                  总设备数
                </h3>
                <p class="text-2xl font-bold mt-2">{{ devices.filter(d => d.device !== 'summary').length }}</p>
              </div>
              <div class="glass-tile p-4 text-center">
                <h3 class="text-gray-500 text-sm font-medium flex items-center justify-center gap-1">
                  <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 text-green-500" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 12h14M5 12a2 2 0 01-2-2V6a2 2 0 012-2h14a2 2 0 012 2v4a2 2 0 01-2 2M5 12a2 2 0 00-2 2v4a2 2 0 002 2h14a2 2 0 002-2v-4a2 2 0 00-2-2m-2-4h.01M17 16h.01" />
                  </svg>
                  在线设备数
                </h3>
                <p class="text-2xl font-bold mt-2 text-green-600">
                  {{ devices.filter(d => d.running).length }}
                </p>
              </div>
            </div>
          </div>

          <!-- 评论区组件 -->
          <GiscusComments
              v-if="showComments"
              :repo="pageConfigs.config.GISCUS_REPO"
              :repo-id="pageConfigs.config.GISCUS_REPOID"
              :category="pageConfigs.config.GISCUS_CATEGORY"
              :category-id="pageConfigs.config.GISCUS_CATEGORYID"
              :mapping="pageConfigs.config.GISCUS_MAPPING"
              :reactions-enabled="pageConfigs.config.GISCUS_REACTIONSENABLED ? '1' : '0'"
              :emit-metadata="pageConfigs.config.GISCUS_EMITMETADATA ? '1' : '0'"
              :input-position="pageConfigs.config.GISCUS_INPUTPOSITION"
              :theme="pageConfigs.config.GISCUS_THEME"
              :lang="pageConfigs.config.GISCUS_LANG"
          />
          <!-- 设备列表 -->
          <div class="sticky top-4">
            <DeviceList
                :devices="devices"
                v-model:selected-device="selectedDevice"
                @refresh-devices="fetchDevices"
            />

            <!-- 日期筛选组件 -->
            <Transition name="slide-fade">
              <DateSelector
                  v-show="selectedDevice !== null"
                  v-model="statsType"
                  v-model:offset="timeOffset"
                  v-model:selected-date="selectedDate"
                  :date-range-text="getDateRangeText()"
                  :server-tz-offset="serverTzOffset"
              />
            </Transition>
          </div>
        </div>

        <!-- 右侧统计 -->
        <div class="flex-1 min-w-0">
          <div class="glass-card p-6 sticky top-40 relative overflow-hidden">
            <div class="flex justify-between items-center mb-4">
              <h2 class="text-xl font-semibold flex items-center gap-2">
                <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z" />
                </svg>
                <span v-if="selectedDevice!=='summary'">{{ selectedDevice }} 使用统计</span>
                <span v-else>总览</span>
              </h2>

              <button v-if="selectedDevice" @click="refreshStats" class="px-3 py-1 text-sm bg-gray-100 rounded-md hover:bg-gray-200 transition-colors dark:bg-gray-800 dark:text-gray-200 dark:hover:bg-gray-700">
                <span class="flex items-center">
                  <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-1 stroke-current" fill="none" viewBox="0 0 24 24" :class="{ 'animate-spin': isRefreshing }">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"/>
                  </svg>
                  刷新
                </span>
              </button>
            </div>

            <!-- 刷新时显示加载遮罩 - 添加过渡动画 -->
            <transition name="fade">
              <div v-if="isRefreshing" class="absolute inset-0 bg-white/10 dark:bg-[#181a1b]/60 backdrop-blur-sm flex flex-col items-center justify-center z-10 rounded-lg">
                <div class="flex flex-col items-center gap-3 pb-[90%]">
                  <svg class="animate-spin h-10 w-10 text-gray-600 dark:text-gray-400" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                  </svg>
                  <span class="text-sm font-medium text-gray-600 dark:text-gray-400">刷新中...</span>
                </div>
              </div>
            </transition>

            <DeviceStats
                v-if="hasRealDevices && selectedDevice"
                :key="statsKey"
                :device-id="selectedDevice"
                :device-info="getSelectedDevice()"
                :date="selectedDate"
                :stats-type="statsType"
                :time-offset="timeOffset"
                @stats-update="handleStatsUpdate"
                :showAiSummary="showAISummary"
                :refresh-trigger="statsKey"
            />

            <!-- 空状态提示 - 仅当没有设备时显示 -->
            <div v-else class="text-center py-8 text-gray-500">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-12 w-12 mx-auto mb-3 text-gray-400" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.75 17L9 20l-1 1h8l-1-1-.75-3M3 13h18M5 17h14a2 2 0 002-2V5a2 2 0 00-2-2H5a2 2 0 00-2 2v10a2 2 0 002 2z" />
              </svg>
              暂无设备数据
            </div>
          </div>
        </div>
      </div>

      <Footer :client-ip="clientIp"></Footer>
    </div>
  </div>
</template>

<style scoped>
/* 骨架屏动画 */
@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

.animate-pulse {
  animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

/* 旋转动画 */
@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.animate-spin {
  animation: spin 1s linear infinite;
}

/* 遮罩淡入淡出动画 */
.fade-enter-active {
  transition: opacity 0.2s ease-in;
}

.fade-leave-active {
  transition: opacity 0.3s ease-out;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.fade-enter-to,
.fade-leave-from {
  opacity: 1;
}

/* DateSelector 动画 - 方案四：使用 max-height 和 overflow */
.slide-fade-enter-active {
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  overflow: hidden;
}

.slide-fade-leave-active {
  transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
  overflow: hidden;
}

.slide-fade-enter-from {
  opacity: 0;
  max-height: 0;
  transform: translateY(-10px);
}

.slide-fade-enter-to {
  opacity: 1;
  max-height: 220px;
  transform: translateY(0);
}

.slide-fade-leave-from {
  opacity: 1;
  max-height: 300px;
  transform: translateY(0);
}

.slide-fade-leave-to {
  opacity: 0;
  max-height: 0;
  transform: translateY(-10px);
}
</style>