<template>
  <div class="dashboard">
    <!-- 顶部导航条 -->
    <header class="top-nav">
      <div class="nav-left">
        <div class="logo">LJ</div>
        <div class="title-group">
          <h1 class="main-title">灵境万象 · 疲劳岗位检测告警中心</h1>
          <p class="sub-title">智能穿戴体征 + 眼球检测 · 巴士司机疲劳风险监测</p>
        </div>
      </div>

      <div class="nav-center">
        <div class="filter-group">
          <div class="filter-item">
            <label>线路</label>
            <select v-model="filters.line">
              <option value="">全部</option>
              <option value="B12">B12</option>
              <option value="K23">K23</option>
              <option value="M05">M05</option>
              <option value="G08">G08</option>
            </select>
          </div>
          <div class="filter-item">
            <label>班次</label>
            <select v-model="filters.shift">
              <option value="">全部</option>
              <option value="早班">早班</option>
              <option value="白班">白班</option>
              <option value="夜班">夜班</option>
            </select>
          </div>
          <div class="filter-item">
            <label>风险级别</label>
            <select v-model="filters.risk">
              <option value="">全部</option>
              <option value="high">高危</option>
              <option value="medium">中等</option>
              <option value="low">低</option>
            </select>
          </div>
        </div>
      </div>

      <div class="nav-right">
        <div class="time-display">
          <div class="current-time">{{ currentTime }}</div>
          <div class="current-date">{{ currentDate }}</div>
        </div>
        <button class="ai-assistant-btn">AI 疲劳助手</button>
      </div>
    </header>

    <!-- 主体内容区（一屏布局） -->
    <main class="main-content">
      <!-- 左侧列 - 超紧凑司机榜 -->
      <aside class="left-column">
        <!-- 今日整体风险概况 -->
        <section class="card overview-card-compact">
          <h2 class="card-title-mini">今日风险概况</h2>
          <div class="stats-row">
            <div class="stat-mini">
              <span class="stat-num">{{ stats.total }}</span>
              <span class="stat-lbl">总数</span>
            </div>
            <div class="stat-mini high">
              <span class="stat-num">{{ stats.high }}</span>
              <span class="stat-lbl">高危</span>
            </div>
            <div class="stat-mini medium">
              <span class="stat-num">{{ stats.medium }}</span>
              <span class="stat-lbl">中等</span>
            </div>
          </div>
          <div class="distribution-bar-mini">
            <div class="bar-seg high" :style="{ width: highPercent + '%' }"></div>
            <div class="bar-seg medium" :style="{ width: mediumPercent + '%' }"></div>
            <div class="bar-seg low" :style="{ width: lowPercent + '%' }"></div>
          </div>
        </section>

        <!-- 司机风险榜（超紧凑版 - 47px高度）-->
        <section class="card driver-rank-card">
          <h2 class="card-title-mini">司机风险榜 Top {{ filteredDrivers.length }}</h2>
          <div class="driver-rank-list">
            <div
              v-for="driver in filteredDrivers"
              :key="driver.id"
              class="driver-rank-item"
              :class="{ active: currentDriver?.id === driver.id }"
              @click="selectDriver(driver)"
            >
              <!-- 头像 -->
              <div class="driver-avatar" :style="{ background: getAvatarGradient(driver.id) }">
                {{ getDriverInitial(driver.name) }}
              </div>

              <!-- 右侧信息 -->
              <div class="driver-rank-info">
                <!-- 第一行：姓名 + 线路 + 疲劳分 + 级别 -->
                <div class="rank-row-1">
                  <div class="rank-name-group">
                    <span class="rank-name">{{ driver.name }}</span>
                    <span class="rank-line">{{ driver.line }}</span>
                  </div>
                  <div class="rank-score-group">
                    <span class="rank-trend" :class="driver.trend">{{ getTrendIcon(driver.trend) }}</span>
                    <span class="rank-score" :class="driver.riskLevel">{{ driver.fatigueScore }}</span>
                    <span class="rank-badge" :class="driver.riskLevel">{{ riskLabelMap[driver.riskLevel] }}</span>
                  </div>
                </div>
                <!-- 第二行：驾驶时长 + 体征标签 + 告警次数 -->
                <div class="rank-row-2">
                  <span class="rank-drive-time">驾驶 {{ driver.driveHours }}h</span>
                  <span class="rank-vital">❤️{{ driver.vitals.heartRate }}</span>
                  <span class="rank-vital">HRV{{ driver.vitals.stressIndex }}</span>
                  <span class="rank-vital">眼{{ driver.eye.perclos }}%</span>
                  <span class="rank-alerts" v-if="driver.alertCount > 0">⚠️{{ driver.alertCount }}</span>
                </div>
              </div>
            </div>
          </div>
        </section>
      </aside>

      <!-- 中间列（4层专业图表叠加）-->
      <section class="center-column">
        <!-- 当前司机概要栏 -->
        <div class="driver-summary-bar-pro">
          <div class="summary-left-pro">
            <span class="focus-icon">👤</span>
            <span class="driver-name-pro">{{ currentDriver?.name }}</span>
            <span class="tag-pro">{{ currentDriver?.line }}</span>
            <span class="tag-pro">{{ currentDriver?.shift }}</span>
            <span class="tag-pro">连续{{ currentDriver?.driveHours }}h</span>
          </div>
          <div class="summary-right-pro">
            <span class="risk-badge-pro" :class="currentDriver?.riskLevel">
              {{ riskStatusMap[currentDriver?.riskLevel] }}
            </span>
          </div>
        </div>

        <!-- 三区图表：上部趋势 + 下部（疲劳构成 + 事件时间轴）-->
        <div class="professional-charts-pro">
          <!-- ① 上部：综合趋势图 + 预测（60%高度）-->
          <div class="chart-layer chart-main-trend">
            <div class="chart-header-pro">
              <h3 class="chart-title-pro">综合疲劳监控 · 最近30分钟 + 未来10分钟预测</h3>
              <div class="chart-legend-pro">
                <span class="legend-item-pro"><i class="dot fatigue"></i>疲劳评分</span>
                <span class="legend-item-pro"><i class="dot heart"></i>心率（标准化）</span>
                <span class="legend-item-pro"><i class="dot stress"></i>压力指数</span>
                <span class="legend-item-pro"><i class="dot predict"></i>未来预测</span>
              </div>
            </div>

            <div class="chart-canvas-pro">
              <!-- 三段色带背景（更明显）-->
              <div class="zone-bg-enhanced">
                <div class="zone-segment danger" title="高危区 >80">
                  <span class="zone-label">高危区 &gt;80</span>
                </div>
                <div class="zone-segment warning" title="警戒区 60-80">
                  <span class="zone-label">警戒区 60-80</span>
                </div>
                <div class="zone-segment safe" title="安全区 <60">
                  <span class="zone-label">安全区 &lt;60</span>
                </div>
              </div>

              <!-- SVG多线趋势图 -->
              <svg class="main-svg-pro" viewBox="0 0 900 200" preserveAspectRatio="none">
                <defs>
                  <linearGradient id="fatigueGrad" x1="0%" y1="0%" x2="0%" y2="100%">
                    <stop offset="0%" style="stop-color:#ff6b3d;stop-opacity:0.3" />
                    <stop offset="100%" style="stop-color:#ff6b3d;stop-opacity:0" />
                  </linearGradient>
                </defs>

                <!-- 压力指数曲线（紫色）-->
                <path :d="stressLinePath" fill="none" stroke="#9b5bff" stroke-width="2" opacity="0.6" />

                <!-- 心率曲线（青色）-->
                <path :d="heartRatePath" fill="none" stroke="#2de1ff" stroke-width="2" opacity="0.7" />

                <!-- 疲劳评分主曲线（橙红色，最粗）-->
                <path :d="fatigueAreaPath" fill="url(#fatigueGrad)" />
                <path :d="fatigueLinePath" fill="none" stroke="#ff6b3d" stroke-width="3.5" />

                <!-- 未来预测虚线（金色）-->
                <path :d="predictPath" fill="none" stroke="#ffd700" stroke-width="2.5" stroke-dasharray="6,4" opacity="0.85" />

                <!-- 疲劳评分数据点 -->
                <circle
                  v-for="(point, idx) in fatiguePoints"
                  :key="'f-'+idx"
                  :cx="point.x"
                  :cy="point.y"
                  r="4.5"
                  :fill="getPointColor(currentDriver?.fatigueTrend[idx])"
                  class="data-point-enhanced"
                />

                <!-- 异常事件标注（用图标）-->
                <g v-for="(event, idx) in currentDriver?.events" :key="'e-'+idx" class="event-icon-group">
                  <circle
                    :cx="event.x"
                    :cy="event.y - 5"
                    r="10"
                    fill="rgba(255, 61, 61, 0.2)"
                    stroke="#ff3d3d"
                    stroke-width="2"
                    class="event-marker-bg"
                  />
                  <text
                    :x="event.x"
                    :y="event.y - 1"
                    font-size="14"
                    text-anchor="middle"
                    class="event-icon"
                  >{{ event.icon }}</text>
                </g>
              </svg>
            </div>
          </div>

          <!-- ② ③ 下部：左右两栏（40%高度）-->
          <div class="chart-bottom-row">
            <!-- ② 左：疲劳构成分析 -->
            <div class="chart-layer chart-composition">
              <div class="chart-header-pro">
                <h3 class="chart-title-pro">疲劳评分构成分析 · 当前 {{ currentDriver?.fatigueScore }} 分</h3>
              </div>
              <div class="composition-bars">
                <div
                  v-for="(item, key) in currentDriver?.fatigueComposition"
                  :key="key"
                  class="composition-item"
                >
                  <div class="composition-label">{{ item.label }}</div>
                  <div class="composition-bar-wrapper">
                    <div
                      class="composition-bar-fill"
                      :class="getCompositionClass(key)"
                      :style="{ width: item.value + '%' }"
                    >
                      <span class="composition-value">{{ item.value }}%</span>
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <!-- ③ 右：事件时间轴 -->
            <div class="chart-layer chart-timeline">
              <div class="chart-header-pro">
                <h3 class="chart-title-pro">异常事件时间轴 · 最近30分钟</h3>
              </div>
              <div class="timeline-container">
                <div class="timeline-axis"></div>
                <div
                  v-for="(event, idx) in currentDriver?.eventsList"
                  :key="'tl-'+idx"
                  class="timeline-event-item"
                  :style="{ left: event.position + '%' }"
                  :class="event.severity"
                >
                  <div class="timeline-event-marker">
                    <span class="timeline-icon">{{ event.icon }}</span>
                  </div>
                  <div class="timeline-event-info">
                    <div class="timeline-event-time">{{ event.time }}</div>
                    <div class="timeline-event-label">{{ event.label }}</div>
                  </div>
                </div>
                <!-- 时间刻度 -->
                <div class="timeline-scale">
                  <span v-for="i in 7" :key="'ts-'+i" class="timeline-tick">{{ (i-1) * 5 }}min</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- 右侧列（分级告警流）-->
      <aside class="right-column">
        <section class="card alerts-card-pro">
          <div class="alerts-header-pro">
            <h2 class="card-title-mini">实时告警流</h2>
            <div class="alert-filter-pro">
              <button
                class="filter-btn-pro"
                :class="{ active: alertFilter === 'all' }"
                @click="alertFilter = 'all'"
              >全部</button>
              <button
                class="filter-btn-pro high"
                :class="{ active: alertFilter === 'high' }"
                @click="alertFilter = 'high'"
              >高危</button>
              <button
                class="filter-btn-pro medium"
                :class="{ active: alertFilter === 'medium' }"
                @click="alertFilter = 'medium'"
              >预警</button>
            </div>
          </div>

          <div class="alerts-stream-pro" ref="alertsContainer" @mouseenter="pauseScroll" @mouseleave="resumeScroll">
            <div class="alerts-scroll-wrapper" :style="{ transform: `translateY(-${scrollOffset}px)` }">
              <div
                v-for="alert in displayAlerts"
                :key="alert.uniqueId"
                class="alert-item-pro"
                :class="alert.severity"
              >
                <div class="alert-severity-stripe" :class="alert.severity"></div>
                <div class="alert-content-pro">
                  <div class="alert-header-row">
                    <span class="alert-badge-pro" :class="alert.severity">{{ severityMap[alert.severity] }}</span>
                    <span class="alert-time-pro">{{ alert.time }}</span>
                  </div>
                  <div class="alert-title-pro">{{ alert.title }}</div>
                  <div class="alert-meta-pro">
                    <span>{{ alert.driver }}</span>
                    <span class="sep">·</span>
                    <span>{{ alert.line }}</span>
                  </div>
                  <button class="alert-action-pro">查看详情</button>
                </div>
              </div>
            </div>
          </div>
        </section>
      </aside>
    </main>

    <!-- 底部三栏（增强版）-->
    <footer class="bottom-bar-pro">
      <!-- 班次风险分布（堆叠柱状图）-->
      <section class="bottom-card-pro">
        <h3 class="bottom-title-pro">班次风险分布</h3>
        <div class="shift-chart-pro">
          <div v-for="shift in shiftStats" :key="shift.name" class="shift-column-pro">
            <div class="shift-bars-pro">
              <div class="shift-bar-stack">
                <div class="bar-stack-item high" :style="{ height: (shift.high / shift.total * 100) + '%' }">
                  <span class="bar-value" v-if="shift.high > 0">{{ shift.high }}</span>
                </div>
                <div class="bar-stack-item medium" :style="{ height: (shift.medium / shift.total * 100) + '%' }">
                  <span class="bar-value" v-if="shift.medium > 1">{{ shift.medium }}</span>
                </div>
                <div class="bar-stack-item low" :style="{ height: (shift.low / shift.total * 100) + '%' }">
                  <span class="bar-value" v-if="shift.low > 2">{{ shift.low }}</span>
                </div>
              </div>
            </div>
            <div class="shift-name-pro">{{ shift.name }}</div>
            <div class="shift-total-pro">{{ shift.total }}人</div>
          </div>
        </div>
      </section>

      <!-- 设备在线情况（点阵图）-->
      <section class="bottom-card-pro">
        <h3 class="bottom-title-pro">设备在线情况</h3>
        <div class="device-grid-pro">
          <div class="device-category-pro">
            <div class="device-name-tag">穿戴设备 {{ devices.wearableOnline }}/{{ devices.wearableTotal }}</div>
            <div class="device-dots-pro">
              <span
                v-for="i in devices.wearableTotal"
                :key="'w-'+i"
                class="device-dot"
                :class="{ online: i <= devices.wearableOnline }"
              ></span>
            </div>
          </div>
          <div class="device-category-pro">
            <div class="device-name-tag">驾驶舱摄像头 {{ devices.cameraOnline }}/{{ devices.cameraTotal }}</div>
            <div class="device-dots-pro">
              <span
                v-for="i in devices.cameraTotal"
                :key="'c-'+i"
                class="device-dot"
                :class="{ online: i <= devices.cameraOnline }"
              ></span>
            </div>
          </div>
        </div>
      </section>

      <!-- AI运营建议（卡片式）-->
      <section class="bottom-card-pro">
        <h3 class="bottom-title-pro">AI 运营建议</h3>
        <div class="ai-cards-pro">
          <div class="ai-card-item">
            <span class="ai-icon">🧠</span>
            <div class="ai-content">
              <div class="ai-title">高危区夜班集中</div>
              <div class="ai-desc">建议提前更换司机</div>
            </div>
          </div>
          <div class="ai-card-item">
            <span class="ai-icon">📌</span>
            <div class="ai-content">
              <div class="ai-title">B12/K23高风险</div>
              <div class="ai-desc">建议强制休息4小时</div>
            </div>
          </div>
        </div>
      </section>
    </footer>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

// 过滤器
const filters = ref({
  line: '',
  shift: '',
  risk: ''
})

// 告警过滤
const alertFilter = ref('all')

// 当前时间
const currentTime = ref('')
const currentDate = ref('')
let timeInterval = null

// 更新时间
const updateTime = () => {
  const now = new Date()
  currentTime.value = now.toLocaleTimeString('zh-CN', { hour12: false })

  const weekdays = ['星期日', '星期一', '星期二', '星期三', '星期四', '星期五', '星期六']
  const year = now.getFullYear()
  const month = now.getMonth() + 1
  const date = now.getDate()
  const weekday = weekdays[now.getDay()]
  currentDate.value = `${year}年${month}月${date}日 · ${weekday}`
}

// Mock 数据 - 司机列表（增强版）
const drivers = ref([
  {
    id: 1,
    name: '张师傅',
    position: '驾驶员',
    line: 'B12',
    shift: '夜班',
    riskLevel: 'high',
    fatigueScore: 87,
    visualFatigueScore: 84,
    physiologicalFatigueScore: 86,
    driveHours: 5.5,
    trend: 'up',
    alertCount: 3,
    vitals: { heartRate: 104, heartRateBaseline: 72, heartRateDeviation: 32, spo2: 95, temp: 36.9, stressIndex: 78 },
    eye: { perclos: 34, gazeAway: 3.2, yawnCount: 5 },
    sleep: { lastNightDuration: 4.5, deepSleepRatio: 22, sleepQuality: 'poor' },
    fatigueTrend: [45, 58, 65, 72, 81, 85, 87],
    visualFatigueTrend: [43, 56, 63, 70, 79, 83, 84],
    physioFatigueTrend: [46, 60, 67, 74, 83, 87, 86],
    heartRateTrend: [75, 82, 88, 95, 100, 102, 104],
    stressTrend: [45, 52, 58, 65, 70, 75, 78],
    perclosTrend: [12, 18, 22, 28, 31, 33, 34],
    predictTrend: [87, 89, 91],
    fatigueComposition: {
      drivingDuration: { value: 35, label: '连续驾驶5.5h' },
      perclos: { value: 30, label: '眼部疲劳（PERCLOS 34%）' },
      heartRate: { value: 20, label: '心率异常（104 bpm）' },
      environment: { value: 10, label: '夜间驾驶' },
      historyFatigue: { value: 5, label: '前48h基础疲劳' }
    },
    events: [
      { x: 500, y: 40, time: '08:40', label: '打盹检测', type: 'sleep', icon: '😴' },
      { x: 700, y: 30, time: '09:12', label: '闭眼率飙升', type: 'eye', icon: '👁️' }
    ],
    eventsList: [
      { position: 30, time: '08:05', label: '心率>100', severity: 'medium', type: 'heart', icon: '❤️', status: 'confirmed' },
      { position: 55, time: '08:40', label: '打盹检测', severity: 'high', type: 'sleep', icon: '😴', status: 'processing' },
      { position: 78, time: '09:12', label: '闭眼率>30%', severity: 'high', type: 'eye', icon: '👁️', status: 'resolved' }
    ]
  },
  {
    id: 2,
    name: '李师傅',
    position: '驾驶员',
    line: 'K23',
    shift: '夜班',
    riskLevel: 'high',
    fatigueScore: 82,
    visualFatigueScore: 79,
    physiologicalFatigueScore: 81,
    driveHours: 4.8,
    trend: 'up',
    alertCount: 2,
    vitals: { heartRate: 98, heartRateBaseline: 70, heartRateDeviation: 28, spo2: 96, temp: 37.1, stressIndex: 72 },
    eye: { perclos: 28, gazeAway: 2.8, yawnCount: 4 },
    sleep: { lastNightDuration: 4.8, deepSleepRatio: 25, sleepQuality: 'poor' },
    fatigueTrend: [42, 55, 62, 70, 76, 80, 82],
    visualFatigueTrend: [40, 53, 60, 68, 74, 78, 79],
    physioFatigueTrend: [43, 57, 64, 72, 78, 82, 81],
    heartRateTrend: [72, 78, 84, 90, 94, 96, 98],
    stressTrend: [42, 48, 55, 62, 68, 70, 72],
    perclosTrend: [10, 15, 19, 23, 26, 27, 28],
    predictTrend: [82, 84, 85],
    fatigueComposition: {
      drivingDuration: { value: 33, label: '连续驾驶4.8h' },
      perclos: { value: 28, label: '眼部疲劳（PERCLOS 28%）' },
      heartRate: { value: 22, label: '心率异常（98 bpm）' },
      environment: { value: 12, label: '夜间驾驶' },
      historyFatigue: { value: 5, label: '前48h基础疲劳' }
    },
    events: [
      { x: 600, y: 50, time: '08:55', label: '方向偏移', type: 'direction', icon: '🔄' }
    ],
    eventsList: [
      { position: 45, time: '08:30', label: '哈欠', severity: 'medium', type: 'sleep', icon: '😴', status: 'confirmed' },
      { position: 67, time: '08:55', label: '偏移', severity: 'high', type: 'direction', icon: '🔄', status: 'resolved' }
    ]
  },
  {
    id: 3,
    name: '王师傅',
    position: '驾驶员',
    line: 'M05',
    shift: '白班',
    riskLevel: 'medium',
    fatigueScore: 68,
    visualFatigueScore: 66,
    physiologicalFatigueScore: 67,
    driveHours: 3.2,
    trend: 'stable',
    alertCount: 1,
    vitals: { heartRate: 88, heartRateBaseline: 72, heartRateDeviation: 16, spo2: 97, temp: 36.7, stressIndex: 58 },
    eye: { perclos: 18, gazeAway: 1.8, yawnCount: 2 },
    sleep: { lastNightDuration: 5.5, deepSleepRatio: 32, sleepQuality: 'fair' },
    fatigueTrend: [35, 42, 50, 58, 62, 66, 68],
    visualFatigueTrend: [33, 40, 48, 56, 60, 64, 66],
    physioFatigueTrend: [36, 44, 52, 60, 64, 68, 67],
    heartRateTrend: [70, 75, 80, 84, 86, 87, 88],
    stressTrend: [38, 42, 48, 52, 55, 57, 58],
    perclosTrend: [8, 11, 14, 16, 17, 18, 18],
    predictTrend: [68, 69, 70],
    fatigueComposition: {
      drivingDuration: { value: 30, label: '连续驾驶3.2h' },
      perclos: { value: 24, label: '眼部疲劳（PERCLOS 18%）' },
      heartRate: { value: 18, label: '心率异常（88 bpm）' },
      environment: { value: 16, label: '白班驾驶' },
      historyFatigue: { value: 12, label: '前48h基础疲劳' }
    },
    events: [],
    eventsList: [
      { position: 40, time: '08:45', label: '压力↑', severity: 'medium', type: 'heart', icon: '❤️', status: 'confirmed' }
    ]
  },
  {
    id: 4,
    name: '赵师傅',
    position: '驾驶员',
    line: 'G08',
    shift: '早班',
    riskLevel: 'medium',
    fatigueScore: 62,
    visualFatigueScore: 60,
    physiologicalFatigueScore: 61,
    driveHours: 2.8,
    trend: 'down',
    alertCount: 0,
    vitals: { heartRate: 82, heartRateBaseline: 71, heartRateDeviation: 11, spo2: 98, temp: 36.6, stressIndex: 52 },
    eye: { perclos: 15, gazeAway: 1.5, yawnCount: 1 },
    sleep: { lastNightDuration: 5.8, deepSleepRatio: 35, sleepQuality: 'fair' },
    fatigueTrend: [30, 38, 45, 52, 56, 60, 62],
    visualFatigueTrend: [28, 36, 43, 50, 54, 58, 60],
    physioFatigueTrend: [31, 40, 47, 54, 58, 62, 61],
    heartRateTrend: [68, 72, 76, 79, 81, 82, 82],
    stressTrend: [35, 40, 44, 48, 50, 51, 52],
    perclosTrend: [6, 9, 12, 14, 15, 15, 15],
    predictTrend: [62, 61, 60],
    fatigueComposition: {
      drivingDuration: { value: 28, label: '连续驾驶2.8h' },
      perclos: { value: 20, label: '眼部疲劳（PERCLOS 15%）' },
      heartRate: { value: 16, label: '心率异常（82 bpm）' },
      environment: { value: 18, label: '早班驾驶' },
      historyFatigue: { value: 18, label: '前48h基础疲劳' }
    },
    events: [],
    eventsList: []
  },
  {
    id: 5,
    name: '刘师傅',
    position: '驾驶员',
    line: 'B12',
    shift: '夜班',
    riskLevel: 'medium',
    fatigueScore: 71,
    visualFatigueScore: 69,
    physiologicalFatigueScore: 70,
    driveHours: 4.1,
    trend: 'up',
    alertCount: 1,
    vitals: { heartRate: 92, heartRateBaseline: 71, heartRateDeviation: 21, spo2: 96, temp: 36.8, stressIndex: 64 },
    eye: { perclos: 22, gazeAway: 2.1, yawnCount: 3 },
    sleep: { lastNightDuration: 5.2, deepSleepRatio: 30, sleepQuality: 'fair' },
    fatigueTrend: [38, 46, 54, 61, 66, 69, 71],
    visualFatigueTrend: [36, 44, 52, 59, 64, 67, 69],
    physioFatigueTrend: [39, 48, 56, 63, 68, 71, 70],
    heartRateTrend: [72, 78, 84, 88, 90, 91, 92],
    stressTrend: [40, 46, 52, 58, 62, 63, 64],
    perclosTrend: [10, 14, 18, 20, 21, 22, 22],
    predictTrend: [71, 73, 75],
    fatigueComposition: {
      drivingDuration: { value: 32, label: '连续驾驶4.1h' },
      perclos: { value: 25, label: '眼部疲劳（PERCLOS 22%）' },
      heartRate: { value: 19, label: '心率异常（92 bpm）' },
      environment: { value: 14, label: '夜间驾驶' },
      historyFatigue: { value: 10, label: '前48h基础疲劳' }
    },
    events: [],
    eventsList: [
      { position: 50, time: '08:50', label: '哈欠', severity: 'medium', type: 'sleep', icon: '😴', status: 'pending' }
    ]
  },
  {
    id: 6,
    name: '陈师傅',
    position: '驾驶员',
    line: 'K23',
    shift: '白班',
    riskLevel: 'low',
    fatigueScore: 45,
    visualFatigueScore: 43,
    physiologicalFatigueScore: 44,
    driveHours: 2.1,
    trend: 'stable',
    alertCount: 0,
    vitals: { heartRate: 75, heartRateBaseline: 72, heartRateDeviation: 3, spo2: 98, temp: 36.5, stressIndex: 42 },
    eye: { perclos: 8, gazeAway: 0.8, yawnCount: 0 },
    sleep: { lastNightDuration: 7.2, deepSleepRatio: 40, sleepQuality: 'good' },
    fatigueTrend: [25, 30, 35, 38, 41, 43, 45],
    visualFatigueTrend: [23, 28, 33, 36, 39, 41, 43],
    physioFatigueTrend: [26, 32, 37, 40, 43, 45, 44],
    heartRateTrend: [68, 70, 72, 74, 75, 75, 75],
    stressTrend: [32, 35, 38, 40, 41, 42, 42],
    perclosTrend: [4, 6, 7, 8, 8, 8, 8],
    predictTrend: [45, 45, 44],
    fatigueComposition: {
      drivingDuration: { value: 22, label: '连续驾驶2.1h' },
      perclos: { value: 12, label: '眼部疲劳（PERCLOS 8%）' },
      heartRate: { value: 10, label: '心率正常（75 bpm）' },
      environment: { value: 26, label: '白班驾驶' },
      historyFatigue: { value: 30, label: '前48h基础疲劳' }
    },
    events: [],
    eventsList: []
  },
  {
    id: 7,
    name: '杨师傅',
    position: '驾驶员',
    line: 'M05',
    shift: '早班',
    riskLevel: 'low',
    fatigueScore: 38,
    visualFatigueScore: 36,
    physiologicalFatigueScore: 37,
    driveHours: 1.5,
    trend: 'stable',
    alertCount: 0,
    vitals: { heartRate: 72, heartRateBaseline: 70, heartRateDeviation: 2, spo2: 99, temp: 36.4, stressIndex: 35 },
    eye: { perclos: 5, gazeAway: 0.5, yawnCount: 0 },
    sleep: { lastNightDuration: 7.5, deepSleepRatio: 42, sleepQuality: 'good' },
    fatigueTrend: [20, 25, 28, 32, 35, 37, 38],
    visualFatigueTrend: [18, 23, 26, 30, 33, 35, 36],
    physioFatigueTrend: [21, 27, 30, 34, 37, 39, 37],
    heartRateTrend: [65, 68, 70, 71, 72, 72, 72],
    stressTrend: [28, 30, 32, 34, 35, 35, 35],
    perclosTrend: [3, 4, 5, 5, 5, 5, 5],
    predictTrend: [38, 38, 37],
    fatigueComposition: {
      drivingDuration: { value: 18, label: '连续驾驶1.5h' },
      perclos: { value: 8, label: '眼部疲劳（PERCLOS 5%）' },
      heartRate: { value: 8, label: '心率正常（72 bpm）' },
      environment: { value: 28, label: '早班驾驶' },
      historyFatigue: { value: 38, label: '前48h基础疲劳' }
    },
    events: [],
    eventsList: []
  },
  {
    id: 8,
    name: '周师傅',
    position: '驾驶员',
    line: 'G08',
    shift: '白班',
    riskLevel: 'low',
    fatigueScore: 41,
    visualFatigueScore: 39,
    physiologicalFatigueScore: 40,
    driveHours: 1.8,
    trend: 'stable',
    alertCount: 0,
    vitals: { heartRate: 78, heartRateBaseline: 72, heartRateDeviation: 6, spo2: 98, temp: 36.6, stressIndex: 38 },
    eye: { perclos: 6, gazeAway: 0.6, yawnCount: 0 },
    sleep: { lastNightDuration: 7.3, deepSleepRatio: 41, sleepQuality: 'good' },
    fatigueTrend: [22, 27, 31, 34, 37, 39, 41],
    visualFatigueTrend: [20, 25, 29, 32, 35, 37, 39],
    physioFatigueTrend: [23, 29, 33, 36, 39, 41, 40],
    heartRateTrend: [68, 72, 74, 76, 77, 78, 78],
    stressTrend: [30, 32, 34, 36, 37, 38, 38],
    perclosTrend: [3, 4, 5, 6, 6, 6, 6],
    predictTrend: [41, 41, 40],
    fatigueComposition: {
      drivingDuration: { value: 20, label: '连续驾驶1.8h' },
      perclos: { value: 9, label: '眼部疲劳（PERCLOS 6%）' },
      heartRate: { value: 9, label: '心率正常（78 bpm）' },
      environment: { value: 26, label: '白班驾驶' },
      historyFatigue: { value: 36, label: '前48h基础疲劳' }
    },
    events: [],
    eventsList: []
  },
  {
    id: 9,
    name: '孙师傅',
    position: '驾驶员',
    line: 'B12',
    shift: '早班',
    riskLevel: 'low',
    fatigueScore: 35,
    visualFatigueScore: 33,
    physiologicalFatigueScore: 34,
    driveHours: 1.2,
    trend: 'stable',
    alertCount: 0,
    vitals: { heartRate: 70, heartRateBaseline: 69, heartRateDeviation: 1, spo2: 99, temp: 36.4, stressIndex: 32 },
    eye: { perclos: 4, gazeAway: 0.4, yawnCount: 0 },
    sleep: { lastNightDuration: 7.8, deepSleepRatio: 43, sleepQuality: 'good' },
    fatigueTrend: [18, 22, 26, 30, 32, 34, 35],
    visualFatigueTrend: [16, 20, 24, 28, 30, 32, 33],
    physioFatigueTrend: [19, 24, 28, 32, 34, 36, 34],
    heartRateTrend: [65, 67, 68, 69, 70, 70, 70],
    stressTrend: [26, 28, 30, 31, 32, 32, 32],
    perclosTrend: [2, 3, 4, 4, 4, 4, 4],
    predictTrend: [35, 34, 33],
    fatigueComposition: {
      drivingDuration: { value: 16, label: '连续驾驶1.2h' },
      perclos: { value: 6, label: '眼部疲劳（PERCLOS 4%）' },
      heartRate: { value: 7, label: '心率正常（70 bpm）' },
      environment: { value: 29, label: '早班驾驶' },
      historyFatigue: { value: 42, label: '前48h基础疲劳' }
    },
    events: [],
    eventsList: []
  },
  {
    id: 10,
    name: '吴师傅',
    position: '驾驶员',
    line: 'K23',
    shift: '白班',
    riskLevel: 'medium',
    fatigueScore: 65,
    visualFatigueScore: 63,
    physiologicalFatigueScore: 64,
    driveHours: 3.5,
    trend: 'up',
    alertCount: 1,
    vitals: { heartRate: 85, heartRateBaseline: 72, heartRateDeviation: 13, spo2: 97, temp: 36.7, stressIndex: 55 },
    eye: { perclos: 16, gazeAway: 1.6, yawnCount: 2 },
    sleep: { lastNightDuration: 5.4, deepSleepRatio: 31, sleepQuality: 'fair' },
    fatigueTrend: [32, 40, 48, 55, 60, 63, 65],
    visualFatigueTrend: [30, 38, 46, 53, 58, 61, 63],
    physioFatigueTrend: [33, 42, 50, 57, 62, 65, 64],
    heartRateTrend: [70, 74, 78, 82, 84, 85, 85],
    stressTrend: [36, 42, 46, 50, 53, 54, 55],
    perclosTrend: [7, 10, 13, 15, 16, 16, 16],
    predictTrend: [65, 67, 68],
    fatigueComposition: {
      drivingDuration: { value: 29, label: '连续驾驶3.5h' },
      perclos: { value: 22, label: '眼部疲劳（PERCLOS 16%）' },
      heartRate: { value: 17, label: '心率异常（85 bpm）' },
      environment: { value: 18, label: '白班驾驶' },
      historyFatigue: { value: 14, label: '前48h基础疲劳' }
    },
    events: [],
    eventsList: [
      { position: 55, time: '08:55', label: '压力↑', severity: 'medium', type: 'heart', icon: '❤️', status: 'confirmed' }
    ]
  },
  {
    id: 11,
    name: '郑师傅',
    position: '驾驶员',
    line: 'M05',
    shift: '夜班',
    riskLevel: 'high',
    fatigueScore: 79,
    visualFatigueScore: 76,
    physiologicalFatigueScore: 78,
    driveHours: 4.5,
    trend: 'up',
    alertCount: 2,
    vitals: { heartRate: 96, heartRateBaseline: 71, heartRateDeviation: 25, spo2: 96, temp: 36.9, stressIndex: 70 },
    eye: { perclos: 26, gazeAway: 2.5, yawnCount: 4 },
    sleep: { lastNightDuration: 4.2, deepSleepRatio: 23, sleepQuality: 'poor' },
    fatigueTrend: [40, 50, 58, 66, 72, 76, 79],
    visualFatigueTrend: [38, 48, 56, 64, 70, 74, 76],
    physioFatigueTrend: [41, 52, 60, 68, 74, 78, 78],
    heartRateTrend: [72, 78, 84, 90, 94, 95, 96],
    stressTrend: [40, 48, 55, 62, 66, 68, 70],
    perclosTrend: [10, 15, 20, 23, 25, 26, 26],
    predictTrend: [79, 81, 83],
    fatigueComposition: {
      drivingDuration: { value: 34, label: '连续驾驶4.5h' },
      perclos: { value: 27, label: '眼部疲劳（PERCLOS 26%）' },
      heartRate: { value: 21, label: '心率异常（96 bpm）' },
      environment: { value: 11, label: '夜间驾驶' },
      historyFatigue: { value: 7, label: '前48h基础疲劳' }
    },
    events: [
      { x: 550, y: 45, time: '08:50', label: '哈欠频繁', type: 'sleep', icon: '😴' }
    ],
    eventsList: [
      { position: 50, time: '08:50', label: '哈欠', severity: 'medium', type: 'sleep', icon: '😴', status: 'confirmed' },
      { position: 72, time: '09:10', label: '眼闭↑', severity: 'high', type: 'eye', icon: '👁️', status: 'processing' }
    ]
  },
  {
    id: 12,
    name: '马师傅',
    position: '驾驶员',
    line: 'G08',
    shift: '早班',
    riskLevel: 'low',
    fatigueScore: 42,
    visualFatigueScore: 40,
    physiologicalFatigueScore: 41,
    driveHours: 1.9,
    trend: 'stable',
    alertCount: 0,
    vitals: { heartRate: 76, heartRateBaseline: 72, heartRateDeviation: 4, spo2: 98, temp: 36.5, stressIndex: 40 },
    eye: { perclos: 7, gazeAway: 0.7, yawnCount: 0 },
    sleep: { lastNightDuration: 7.4, deepSleepRatio: 41, sleepQuality: 'good' },
    fatigueTrend: [24, 28, 32, 36, 39, 41, 42],
    visualFatigueTrend: [22, 26, 30, 34, 37, 39, 40],
    physioFatigueTrend: [25, 30, 34, 38, 41, 43, 41],
    heartRateTrend: [68, 70, 72, 74, 75, 76, 76],
    stressTrend: [32, 34, 36, 38, 39, 40, 40],
    perclosTrend: [4, 5, 6, 7, 7, 7, 7],
    predictTrend: [42, 42, 41],
    fatigueComposition: {
      drivingDuration: { value: 21, label: '连续驾驶1.9h' },
      perclos: { value: 10, label: '眼部疲劳（PERCLOS 7%）' },
      heartRate: { value: 9, label: '心率正常（76 bpm）' },
      environment: { value: 26, label: '早班驾驶' },
      historyFatigue: { value: 34, label: '前48h基础疲劳' }
    },
    events: [],
    eventsList: []
  }
])

// 当前关注的司机
const currentDriver = ref(drivers.value[0])

// 选择司机
const selectDriver = (driver) => {
  currentDriver.value = driver
}

// 过滤后的司机列表
const filteredDrivers = computed(() => {
  return drivers.value.filter(driver => {
    if (filters.value.line && driver.line !== filters.value.line) return false
    if (filters.value.shift && driver.shift !== filters.value.shift) return false
    if (filters.value.risk && driver.riskLevel !== filters.value.risk) return false
    return true
  }).sort((a, b) => b.fatigueScore - a.fatigueScore)
})

// 统计数据
const stats = computed(() => {
  const filtered = filteredDrivers.value
  return {
    total: filtered.length,
    high: filtered.filter(d => d.riskLevel === 'high').length,
    medium: filtered.filter(d => d.riskLevel === 'medium').length,
    low: filtered.filter(d => d.riskLevel === 'low').length
  }
})

const highPercent = computed(() => Math.round(stats.value.high / stats.value.total * 100) || 0)
const mediumPercent = computed(() => Math.round(stats.value.medium / stats.value.total * 100) || 0)
const lowPercent = computed(() => Math.round(stats.value.low / stats.value.total * 100) || 0)

// 设备状态
const devices = ref({
  wearableOnline: 142,
  wearableTotal: 150,
  cameraOnline: 138,
  cameraTotal: 150
})

// 告警列表
const alerts = ref([
  {
    id: 1,
    severity: 'high',
    title: '连续3次检测到长时间闭眼 + 车辆轻微偏移',
    driver: '张师傅',
    line: 'B12',
    time: '2分钟前'
  },
  {
    id: 2,
    severity: 'high',
    title: '心率持续偏高(>100bpm) + PERCLOS超过30%',
    driver: '李师傅',
    line: 'K23',
    time: '5分钟前'
  },
  {
    id: 3,
    severity: 'high',
    title: '连续打哈欠4次 + 压力指数飙升',
    driver: '郑师傅',
    line: 'M05',
    time: '7分钟前'
  },
  {
    id: 4,
    severity: 'medium',
    title: '连续驾驶超过4小时未休息',
    driver: '刘师傅',
    line: 'B12',
    time: '8分钟前'
  },
  {
    id: 5,
    severity: 'medium',
    title: '压力指数偏高 + 打哈欠频次增加',
    driver: '王师傅',
    line: 'M05',
    time: '12分钟前'
  },
  {
    id: 6,
    severity: 'medium',
    title: '方向盘操作异常波动',
    driver: '吴师傅',
    line: 'K23',
    time: '15分钟前'
  },
  {
    id: 7,
    severity: 'low',
    title: '视线偏移持续时间略长',
    driver: '赵师傅',
    line: 'G08',
    time: '18分钟前'
  }
])

// 过滤后的告警
const filteredAlerts = computed(() => {
  if (alertFilter.value === 'all') return alerts.value
  return alerts.value.filter(a => a.severity === alertFilter.value)
})

// 班次统计
const shiftStats = ref([
  { name: '早班', total: 48, high: 2, medium: 8, low: 38 },
  { name: '白班', total: 52, high: 1, medium: 12, low: 39 },
  { name: '夜班', total: 50, high: 5, medium: 15, low: 30 }
])

// 映射
const riskLabelMap = {
  high: '高危',
  medium: '中等',
  low: '低'
}

const riskStatusMap = {
  high: '高危',
  medium: '警戒',
  low: '正常'
}

const severityMap = {
  high: '高危',
  medium: '预警',
  low: '提示'
}

// 趋势箭头
const getTrendIcon = (trend) => {
  if (trend === 'up') return '↗'
  if (trend === 'down') return '↘'
  return '→'
}

// 头像相关函数
const getDriverInitial = (name) => {
  return name ? name.charAt(0) : '?'
}

const getAvatarGradient = (id) => {
  const gradients = [
    'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
    'linear-gradient(135deg, #f093fb 0%, #f5576c 100%)',
    'linear-gradient(135deg, #4facfe 0%, #00f2fe 100%)',
    'linear-gradient(135deg, #43e97b 0%, #38f9d7 100%)',
    'linear-gradient(135deg, #fa709a 0%, #fee140 100%)',
    'linear-gradient(135deg, #30cfd0 0%, #330867 100%)',
    'linear-gradient(135deg, #a8edea 0%, #fed6e3 100%)',
    'linear-gradient(135deg, #ff9a56 0%, #ff6a88 100%)',
    'linear-gradient(135deg, #fbc2eb 0%, #a6c1ee 100%)',
    'linear-gradient(135deg, #fdcbf1 0%, #e6dee9 100%)',
    'linear-gradient(135deg, #a1c4fd 0%, #c2e9fb 100%)',
    'linear-gradient(135deg, #d299c2 0%, #fef9d7 100%)'
  ]
  return gradients[(id - 1) % gradients.length]
}

// 告警滚动相关
const alertsContainer = ref(null)
const scrollOffset = ref(0)
const isScrollPaused = ref(false)
let scrollInterval = null

// 复制告警列表以实现无缝滚动
const displayAlerts = computed(() => {
  const filtered = filteredAlerts.value
  // 复制两份用于无缝循环
  return [...filtered, ...filtered].map((alert, index) => ({
    ...alert,
    uniqueId: `${alert.id}-${index}`
  }))
})

const pauseScroll = () => {
  isScrollPaused.value = true
}

const resumeScroll = () => {
  isScrollPaused.value = false
}

const startAutoScroll = () => {
  scrollInterval = setInterval(() => {
    if (!isScrollPaused.value) {
      scrollOffset.value += 0.5

      // 计算单组告警的总高度（每个告警约110px）
      const singleGroupHeight = filteredAlerts.value.length * 110

      // 当滚动超过一组的高度时，重置回0实现无缝循环
      if (scrollOffset.value >= singleGroupHeight) {
        scrollOffset.value = 0
      }
    }
  }, 30)
}

// SVG 路径计算
const createPath = (data, width, height, yMax = 100) => {
  if (!data || data.length === 0) return ''
  const stepX = width / (data.length - 1)
  return data.map((value, index) => {
    const x = index * stepX
    const y = height - (value / yMax * height)
    return index === 0 ? `M ${x} ${y}` : `L ${x} ${y}`
  }).join(' ')
}

const createAreaPath = (data, width, height, yMax = 100) => {
  if (!data || data.length === 0) return ''
  const stepX = width / (data.length - 1)
  let path = `M 0 ${height} `
  path += data.map((value, index) => {
    const x = index * stepX
    const y = height - (value / yMax * height)
    return `L ${x} ${y}`
  }).join(' ')
  path += ` L ${width} ${height} Z`
  return path
}

// 疲劳曲线
const fatiguePoints = computed(() => {
  if (!currentDriver.value?.fatigueTrend) return []
  const data = currentDriver.value.fatigueTrend
  const width = 900
  const height = 200
  const stepX = width / (data.length - 1)
  return data.map((value, index) => ({
    x: index * stepX,
    y: height - (value / 100 * height)
  }))
})

const fatigueLinePath = computed(() => {
  if (!currentDriver.value?.fatigueTrend) return ''
  return createPath(currentDriver.value.fatigueTrend, 900, 200)
})

const fatigueAreaPath = computed(() => {
  if (!currentDriver.value?.fatigueTrend) return ''
  return createAreaPath(currentDriver.value.fatigueTrend, 900, 200)
})

// 心率曲线（映射到0-100范围：50-120 -> 0-100）
const heartRatePath = computed(() => {
  if (!currentDriver.value?.heartRateTrend) return ''
  const normalized = currentDriver.value.heartRateTrend.map(v => ((v - 50) / 70) * 100)
  return createPath(normalized, 900, 200)
})

// 压力曲线
const stressLinePath = computed(() => {
  if (!currentDriver.value?.stressTrend) return ''
  return createPath(currentDriver.value.stressTrend, 900, 200)
})

const stressAreaPath = computed(() => {
  if (!currentDriver.value?.stressTrend) return ''
  return createAreaPath(currentDriver.value.stressTrend, 900, 200)
})

// 预测曲线
const predictPath = computed(() => {
  if (!currentDriver.value?.predictTrend) return ''
  const allData = [...currentDriver.value.fatigueTrend, ...currentDriver.value.predictTrend]
  const width = 900
  const height = 200
  const totalPoints = allData.length
  const stepX = width / (totalPoints - 1)

  const startIdx = currentDriver.value.fatigueTrend.length - 1
  return currentDriver.value.predictTrend.map((value, index) => {
    const x = (startIdx + index) * stepX
    const y = height - (value / 100 * height)
    return index === 0 ? `M ${x} ${y}` : `L ${x} ${y}`
  }).join(' ')
})

const getPointColor = (value) => {
  if (value > 80) return '#ff3d3d'
  if (value > 60) return '#ffa500'
  return '#00ff88'
}

const getPerclosClass = (value) => {
  if (value > 30) return 'danger'
  if (value > 20) return 'warning'
  return 'safe'
}

// 疲劳构成条颜色分类
const getCompositionClass = (key) => {
  const classMap = {
    drivingDuration: 'comp-duration',
    perclos: 'comp-perclos',
    heartRate: 'comp-heart',
    environment: 'comp-env',
    historyFatigue: 'comp-history'
  }
  return classMap[key] || 'comp-default'
}

// 生命周期
onMounted(() => {
  updateTime()
  timeInterval = setInterval(updateTime, 1000)
  startAutoScroll()
})

onUnmounted(() => {
  if (timeInterval) {
    clearInterval(timeInterval)
  }
  if (scrollInterval) {
    clearInterval(scrollInterval)
  }
})
</script>

<style scoped lang="scss">
* {
  box-sizing: border-box;
}

.dashboard {
  min-height: 100vh;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: linear-gradient(135deg, #0a0e27 0%, #1a1442 50%, #0f1635 100%);
  color: #e0e6ed;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'PingFang SC', sans-serif;
  padding: 14px;
  overflow: hidden;
}

/* ========== 顶部导航（紧凑版）========== */
.top-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: rgba(20, 25, 45, 0.6);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(45, 225, 255, 0.2);
  border-radius: 10px;
  padding: 10px 18px;
  margin-bottom: 14px;
  flex-shrink: 0;
}

.nav-left {
  display: flex;
  align-items: center;
  gap: 10px;
}

.logo {
  width: 36px;
  height: 36px;
  background: linear-gradient(135deg, #2de1ff, #9b5bff);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 16px;
  color: #fff;
  box-shadow: 0 0 16px rgba(45, 225, 255, 0.5);
}

.title-group {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.main-title {
  font-size: 16px;
  font-weight: 600;
  margin: 0;
  background: linear-gradient(90deg, #2de1ff, #9b5bff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.sub-title {
  font-size: 10px;
  color: #8b92a7;
  margin: 0;
}

.nav-center {
  flex: 1;
  display: flex;
  justify-content: center;
}

.filter-group {
  display: flex;
  gap: 10px;
}

.filter-item {
  display: flex;
  flex-direction: column;
  gap: 2px;

  label {
    font-size: 10px;
    color: #8b92a7;
  }

  select {
    background: rgba(45, 225, 255, 0.1);
    border: 1px solid rgba(45, 225, 255, 0.3);
    border-radius: 5px;
    padding: 4px 8px;
    color: #e0e6ed;
    font-size: 11px;
    outline: none;
    cursor: pointer;
    transition: all 0.3s;

    &:hover {
      border-color: rgba(45, 225, 255, 0.5);
    }

    option {
      background: #1a1442;
    }
  }
}

.nav-right {
  display: flex;
  align-items: center;
  gap: 14px;
}

.time-display {
  text-align: right;
}

.current-time {
  font-size: 18px;
  font-weight: 600;
  color: #2de1ff;
  font-family: 'Courier New', monospace;
}

.current-date {
  font-size: 10px;
  color: #8b92a7;
}

.ai-assistant-btn {
  background: linear-gradient(135deg, #9b5bff, #2de1ff);
  border: none;
  border-radius: 7px;
  padding: 7px 14px;
  color: #fff;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;

  &:hover {
    transform: translateY(-2px);
  }
}

/* ========== 主体内容（一屏布局）========== */
.main-content {
  flex: 1;
  display: grid;
  grid-template-columns: 250px 1fr 320px;
  gap: 14px;
  overflow: hidden;
  min-height: 0;
}

/* ========== 卡片通用 ========== */
.card {
  background: rgba(20, 25, 45, 0.5);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(45, 225, 255, 0.15);
  border-radius: 8px;
  padding: 12px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.2);
}

.card-title-mini {
  font-size: 12px;
  font-weight: 600;
  color: #2de1ff;
  margin: 0 0 10px 0;
  padding-bottom: 8px;
  border-bottom: 1px solid rgba(45, 225, 255, 0.2);
}

/* ========== 左侧列（超紧凑）========== */
.left-column {
  display: flex;
  flex-direction: column;
  gap: 12px;
  overflow: hidden;
}

.overview-card-compact {
  flex-shrink: 0;
}

.stats-row {
  display: flex;
  gap: 8px;
  margin-bottom: 10px;
}

.stat-mini {
  flex: 1;
  text-align: center;
  display: flex;
  flex-direction: column;
  gap: 2px;

  .stat-num {
    font-size: 20px;
    font-weight: 700;
    color: #2de1ff;
  }

  .stat-lbl {
    font-size: 10px;
    color: #8b92a7;
  }

  &.high .stat-num {
    color: #ff6b3d;
  }

  &.medium .stat-num {
    color: #ffa500;
  }
}

.distribution-bar-mini {
  display: flex;
  height: 5px;
  border-radius: 2px;
  overflow: hidden;
}

.bar-seg {
  transition: width 0.3s;

  &.high { background: #ff6b3d; }
  &.medium { background: #ffa500; }
  &.low { background: #00ff88; }
}

/* 司机榜（47px高度）*/
.driver-rank-card {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.driver-rank-list {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 5px;

  &::-webkit-scrollbar {
    width: 3px;
  }

  &::-webkit-scrollbar-thumb {
    background: rgba(45, 225, 255, 0.2);
    border-radius: 2px;
  }
}

.driver-rank-item {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 8px;
  padding: 6px 8px;
  background: rgba(45, 225, 255, 0.05);
  border: 1px solid rgba(45, 225, 255, 0.1);
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.3s;
  flex-shrink: 0;
  min-height: 47px;

  &:hover {
    background: rgba(45, 225, 255, 0.1);
    border-color: rgba(45, 225, 255, 0.3);
  }

  &.active {
    background: rgba(45, 225, 255, 0.15);
    border-color: rgba(45, 225, 255, 0.5);
    box-shadow: 0 0 10px rgba(45, 225, 255, 0.2);
  }
}

.driver-avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  font-weight: 700;
  color: #fff;
  flex-shrink: 0;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

.driver-rank-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 3px;
  min-width: 0;
}

.rank-row-1 {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.rank-name-group {
  display: flex;
  align-items: center;
  gap: 5px;
}

.rank-name {
  font-size: 12px;
  font-weight: 600;
  color: #e0e6ed;
}

.rank-line {
  font-size: 10px;
  color: #8b92a7;
  padding: 1px 5px;
  background: rgba(45, 225, 255, 0.1);
  border-radius: 3px;
}

.rank-score-group {
  display: flex;
  align-items: center;
  gap: 4px;
}

.rank-trend {
  font-size: 14px;

  &.up { color: #ff6b3d; }
  &.down { color: #00ff88; }
  &.stable { color: #8b92a7; }
}

.rank-score {
  font-size: 18px;
  font-weight: 700;

  &.high { color: #ff6b3d; }
  &.medium { color: #ffa500; }
  &.low { color: #00ff88; }
}

.rank-badge {
  font-size: 9px;
  padding: 2px 5px;
  border-radius: 3px;
  font-weight: 600;

  &.high {
    background: rgba(255, 107, 61, 0.2);
    color: #ff6b3d;
  }

  &.medium {
    background: rgba(255, 165, 0, 0.2);
    color: #ffa500;
  }

  &.low {
    background: rgba(0, 255, 136, 0.2);
    color: #00ff88;
  }
}

.rank-row-2 {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 9px;
  color: #8b92a7;
}

.rank-drive-time {
  color: #9b5bff;
}

.rank-vital {
  padding: 1px 4px;
  background: rgba(45, 225, 255, 0.1);
  border-radius: 2px;
}

.rank-alerts {
  color: #ff6b3d;
  font-weight: 600;
}

/* ========== 中间列（4层图表）========== */
.center-column {
  display: flex;
  flex-direction: column;
  gap: 10px;
  overflow: hidden;
  min-height: 0;
}

.driver-summary-bar-pro {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: rgba(20, 25, 45, 0.5);
  border: 1px solid rgba(45, 225, 255, 0.15);
  border-radius: 8px;
  padding: 10px 14px;
  flex-shrink: 0;
}

.summary-left-pro {
  display: flex;
  align-items: center;
  gap: 8px;
}

.focus-icon {
  font-size: 16px;
}

.driver-name-pro {
  font-size: 15px;
  font-weight: 700;
  color: #e0e6ed;
}

.tag-pro {
  font-size: 10px;
  padding: 2px 7px;
  background: rgba(45, 225, 255, 0.1);
  border: 1px solid rgba(45, 225, 255, 0.3);
  border-radius: 4px;
  color: #2de1ff;
}

.risk-badge-pro {
  font-size: 13px;
  font-weight: 700;
  padding: 5px 12px;
  border-radius: 5px;

  &.high {
    background: rgba(255, 107, 61, 0.2);
    color: #ff6b3d;
    border: 2px solid rgba(255, 107, 61, 0.5);
  }

  &.medium {
    background: rgba(255, 165, 0, 0.2);
    color: #ffa500;
    border: 2px solid rgba(255, 165, 0, 0.5);
  }

  &.low {
    background: rgba(0, 255, 136, 0.2);
    color: #00ff88;
    border: 2px solid rgba(0, 255, 136, 0.5);
  }
}

/* 三区图表区 */
.professional-charts-pro {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 10px;
  overflow: hidden;
  min-height: 0;
}

.chart-layer {
  background: rgba(20, 25, 45, 0.5);
  border: 1px solid rgba(45, 225, 255, 0.15);
  border-radius: 8px;
  padding: 10px;
  overflow: hidden;
}

.chart-main-trend {
  flex: 1.5;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.chart-bottom-row {
  display: grid;
  grid-template-columns: 1.2fr 1fr;
  gap: 10px;
  flex-shrink: 0;
  height: 140px;
}

.chart-header-pro {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
  padding-bottom: 6px;
  border-bottom: 1px solid rgba(45, 225, 255, 0.15);
}

.chart-title-pro {
  font-size: 12px;
  font-weight: 600;
  color: #2de1ff;
  margin: 0;
}

.chart-legend-pro {
  display: flex;
  gap: 10px;
}

.legend-item-pro {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 10px;
  color: #8b92a7;

  .dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;

    &.fatigue { background: #ff6b3d; }
    &.heart { background: #2de1ff; }
    &.stress { background: #9b5bff; }
    &.perclos { background: #ffa500; }
    &.predict { background: #ffd700; }
  }
}

.chart-canvas-pro {
  flex: 1;
  position: relative;
  min-height: 0;
}

/* 三段色带背景（增强版）*/
.zone-bg-enhanced {
  position: absolute;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  z-index: 0;
}

.zone-segment {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  padding-right: 12px;
  position: relative;

  &.danger {
    background: linear-gradient(90deg, rgba(255, 61, 61, 0.08), rgba(255, 61, 61, 0.15));
    border-bottom: 1px solid rgba(255, 61, 61, 0.3);
  }

  &.warning {
    background: linear-gradient(90deg, rgba(255, 165, 0, 0.08), rgba(255, 165, 0, 0.15));
    border-bottom: 1px solid rgba(255, 165, 0, 0.3);
  }

  &.safe {
    background: linear-gradient(90deg, rgba(0, 255, 136, 0.08), rgba(0, 255, 136, 0.15));
  }
}

.zone-label {
  font-size: 10px;
  font-weight: 700;
  opacity: 0.6;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.5);

  .danger & { color: #ff6b3d; }
  .warning & { color: #ffa500; }
  .safe & { color: #00ff88; }
}

.main-svg-pro {
  position: absolute;
  width: 100%;
  height: 100%;
  z-index: 1;
}

/* 数据点增强版 */
.data-point-enhanced {
  transition: all 0.3s;
  cursor: pointer;
  filter: drop-shadow(0 0 2px currentColor);

  &:hover {
    r: 6.5;
    filter: drop-shadow(0 0 6px currentColor);
  }
}

/* 事件图标标记 */
.event-icon-group {
  cursor: pointer;
  transition: all 0.3s;

  &:hover {
    transform: scale(1.15);
  }
}

.event-marker-bg {
  animation: eventPulse 2.5s ease-in-out infinite;
}

.event-icon {
  pointer-events: none;
  user-select: none;
}

@keyframes eventPulse {
  0%, 100% {
    opacity: 0.8;
    r: 10;
  }
  50% {
    opacity: 1;
    r: 12;
  }
}

/* ========== ② 疲劳构成条 ========== */
.chart-composition {
  display: flex;
  flex-direction: column;
}

.composition-bars {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 5px 0;
}

.composition-item {
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.composition-label {
  font-size: 10px;
  color: #8b92a7;
  font-weight: 500;
}

.composition-bar-wrapper {
  width: 100%;
  height: 16px;
  background: rgba(45, 225, 255, 0.08);
  border-radius: 4px;
  overflow: hidden;
  border: 1px solid rgba(45, 225, 255, 0.15);
}

.composition-bar-fill {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  padding-right: 6px;
  transition: width 0.5s ease;
  position: relative;

  &.comp-duration {
    background: linear-gradient(90deg, #9b5bff, #c77dff);
  }

  &.comp-perclos {
    background: linear-gradient(90deg, #ffa500, #ffd700);
  }

  &.comp-heart {
    background: linear-gradient(90deg, #ff6b3d, #ff9966);
  }

  &.comp-env {
    background: linear-gradient(90deg, #2de1ff, #4facfe);
  }

  &.comp-history {
    background: linear-gradient(90deg, #8b92a7, #b0b7c3);
  }
}

.composition-value {
  font-size: 10px;
  font-weight: 700;
  color: #fff;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.5);
}

/* ========== ③ 事件时间轴 ========== */
.chart-timeline {
  display: flex;
  flex-direction: column;
}

.timeline-container {
  position: relative;
  flex: 1;
  padding: 15px 10px 25px 10px;
}

.timeline-axis {
  position: absolute;
  top: 30px;
  left: 10px;
  right: 10px;
  height: 3px;
  background: linear-gradient(90deg, rgba(45, 225, 255, 0.3), rgba(155, 91, 255, 0.3));
  border-radius: 2px;
}

.timeline-event-item {
  position: absolute;
  top: 8px;
  transform: translateX(-50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 3px;
  cursor: pointer;
  transition: all 0.3s;

  &:hover {
    transform: translateX(-50%) scale(1.1);
    z-index: 10;

    .timeline-event-info {
      opacity: 1;
      transform: translateY(0);
    }
  }
}

.timeline-event-marker {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  z-index: 2;

  .high & {
    background: rgba(255, 61, 61, 0.2);
    border: 2px solid #ff3d3d;
    box-shadow: 0 0 12px rgba(255, 61, 61, 0.6);
  }

  .medium & {
    background: rgba(255, 165, 0, 0.2);
    border: 2px solid #ffa500;
    box-shadow: 0 0 10px rgba(255, 165, 0, 0.5);
  }
}

.timeline-icon {
  font-size: 14px;
}

.timeline-event-info {
  position: absolute;
  top: 100%;
  margin-top: 8px;
  background: rgba(20, 25, 45, 0.95);
  border: 1px solid rgba(45, 225, 255, 0.3);
  border-radius: 5px;
  padding: 4px 7px;
  white-space: nowrap;
  opacity: 0;
  transform: translateY(-5px);
  transition: all 0.3s;
  pointer-events: none;
}

.timeline-event-time {
  font-size: 9px;
  color: #2de1ff;
  font-weight: 600;
  margin-bottom: 1px;
}

.timeline-event-label {
  font-size: 10px;
  color: #e0e6ed;
}

.timeline-scale {
  position: absolute;
  bottom: 0;
  left: 10px;
  right: 10px;
  display: flex;
  justify-content: space-between;
}

.timeline-tick {
  font-size: 9px;
  color: #6b7280;
}

/* ========== 右侧列（分级告警）========== */
.right-column {
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-height: 0;
}

.alerts-card-pro {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-height: 0;
}

.alerts-header-pro {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  padding-bottom: 8px;
  border-bottom: 1px solid rgba(45, 225, 255, 0.2);
}

.alert-filter-pro {
  display: flex;
  gap: 4px;
}

.filter-btn-pro {
  padding: 3px 8px;
  font-size: 10px;
  background: rgba(45, 225, 255, 0.1);
  border: 1px solid rgba(45, 225, 255, 0.2);
  color: #8b92a7;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.3s;

  &:hover {
    border-color: rgba(45, 225, 255, 0.4);
  }

  &.active {
    background: rgba(45, 225, 255, 0.2);
    border-color: rgba(45, 225, 255, 0.5);
    color: #2de1ff;
  }

  &.high.active {
    background: rgba(255, 107, 61, 0.2);
    border-color: rgba(255, 107, 61, 0.5);
    color: #ff6b3d;
  }

  &.medium.active {
    background: rgba(255, 165, 0, 0.2);
    border-color: rgba(255, 165, 0, 0.5);
    color: #ffa500;
  }
}

.alerts-stream-pro {
  flex: 1;
  overflow: hidden;
  position: relative;

  &::-webkit-scrollbar {
    display: none;
  }
}

.alerts-scroll-wrapper {
  display: flex;
  flex-direction: column;
  gap: 8px;
  transition: transform 0.05s linear;
}

.alert-item-pro {
  display: flex;
  gap: 8px;
  padding: 8px;
  background: rgba(45, 225, 255, 0.05);
  border: 1px solid rgba(45, 225, 255, 0.1);
  border-radius: 6px;
  transition: all 0.3s;
  flex-shrink: 0;

  &:hover {
    background: rgba(45, 225, 255, 0.08);
    border-color: rgba(45, 225, 255, 0.2);
  }

  &.high {
    border-left: 3px solid #ff6b3d;
  }

  &.medium {
    border-left: 3px solid #ffa500;
  }

  &.low {
    border-left: 3px solid #2de1ff;
  }
}

.alert-severity-stripe {
  width: 3px;
  border-radius: 2px;
  flex-shrink: 0;

  &.high { background: linear-gradient(180deg, #ff6b3d, #ff3d3d); }
  &.medium { background: linear-gradient(180deg, #ffa500, #ffd700); }
  &.low { background: linear-gradient(180deg, #2de1ff, #00ff88); }
}

.alert-content-pro {
  flex: 1;
  min-width: 0;
}

.alert-header-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 5px;
}

.alert-badge-pro {
  font-size: 9px;
  padding: 2px 5px;
  border-radius: 3px;
  font-weight: 600;

  &.high {
    background: rgba(255, 107, 61, 0.2);
    color: #ff6b3d;
  }

  &.medium {
    background: rgba(255, 165, 0, 0.2);
    color: #ffa500;
  }

  &.low {
    background: rgba(45, 225, 255, 0.2);
    color: #2de1ff;
  }
}

.alert-time-pro {
  font-size: 9px;
  color: #6b7280;
}

.alert-title-pro {
  font-size: 11px;
  color: #e0e6ed;
  margin-bottom: 4px;
  line-height: 1.4;
}

.alert-meta-pro {
  font-size: 9px;
  color: #8b92a7;
  margin-bottom: 5px;

  .sep {
    margin: 0 3px;
  }
}

.alert-action-pro {
  font-size: 9px;
  padding: 3px 8px;
  background: rgba(45, 225, 255, 0.1);
  border: 1px solid rgba(45, 225, 255, 0.3);
  color: #2de1ff;
  border-radius: 3px;
  cursor: pointer;
  transition: all 0.3s;

  &:hover {
    background: rgba(45, 225, 255, 0.2);
  }
}

/* ========== 底部三栏（增强版）========== */
.bottom-bar-pro {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 14px;
  margin-top: 14px;
  flex-shrink: 0;
  height: 120px;
}

.bottom-card-pro {
  background: rgba(20, 25, 45, 0.5);
  border: 1px solid rgba(45, 225, 255, 0.15);
  border-radius: 8px;
  padding: 10px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.2);
  overflow: hidden;
}

.bottom-title-pro {
  font-size: 11px;
  font-weight: 600;
  color: #2de1ff;
  margin: 0 0 8px 0;
  padding-bottom: 6px;
  border-bottom: 1px solid rgba(45, 225, 255, 0.2);
}

/* 班次堆叠柱状图 */
.shift-chart-pro {
  display: flex;
  justify-content: space-around;
  align-items: flex-end;
  height: 70px;
  gap: 8px;
}

.shift-column-pro {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

.shift-bars-pro {
  flex: 1;
  width: 100%;
  display: flex;
  align-items: flex-end;
}

.shift-bar-stack {
  width: 100%;
  display: flex;
  flex-direction: column-reverse;
  border-radius: 4px 4px 0 0;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
}

.bar-stack-item {
  display: flex;
  align-items: center;
  justify-content: center;
  transition: height 0.3s;

  &.high { background: #ff6b3d; }
  &.medium { background: #ffa500; }
  &.low { background: #00ff88; }

  .bar-value {
    font-size: 9px;
    font-weight: 700;
    color: #fff;
    text-shadow: 0 1px 2px rgba(0,0,0,0.3);
  }
}

.shift-name-pro {
  font-size: 10px;
  font-weight: 600;
  color: #e0e6ed;
}

.shift-total-pro {
  font-size: 9px;
  color: #8b92a7;
}

/* 设备点阵图 */
.device-grid-pro {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.device-category-pro {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.device-name-tag {
  font-size: 10px;
  color: #8b92a7;
}

.device-dots-pro {
  display: flex;
  flex-wrap: wrap;
  gap: 3px;
}

.device-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: rgba(255, 61, 61, 0.3);
  transition: all 0.3s;

  &.online {
    background: #00ff88;
    box-shadow: 0 0 4px rgba(0, 255, 136, 0.6);
  }
}

/* AI卡片式建议 */
.ai-cards-pro {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.ai-card-item {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 6px;
  background: rgba(45, 225, 255, 0.05);
  border: 1px solid rgba(45, 225, 255, 0.15);
  border-radius: 5px;
  border-left: 3px solid #9b5bff;
}

.ai-icon {
  font-size: 16px;
  flex-shrink: 0;
}

.ai-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.ai-title {
  font-size: 10px;
  font-weight: 600;
  color: #e0e6ed;
}

.ai-desc {
  font-size: 9px;
  color: #8b92a7;
  line-height: 1.3;
}
</style>
