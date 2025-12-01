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
      <!-- 左侧列 -->
      <aside class="left-column">
        <!-- 今日整体风险概况 -->
        <section class="card overview-card">
          <h2 class="card-title">今日整体风险概况</h2>
          <div class="stats-grid">
            <div class="stat-item">
              <div class="stat-value">{{ stats.total }}</div>
              <div class="stat-label">监控司机</div>
            </div>
            <div class="stat-item">
              <div class="stat-value high">{{ stats.high }}</div>
              <div class="stat-label">高危</div>
            </div>
            <div class="stat-item">
              <div class="stat-value medium">{{ stats.medium }}</div>
              <div class="stat-label">中等</div>
            </div>
          </div>

          <div class="risk-distribution">
            <div class="distribution-bar">
              <div class="bar-segment high" :style="{ width: highPercent + '%' }"></div>
              <div class="bar-segment medium" :style="{ width: mediumPercent + '%' }"></div>
              <div class="bar-segment low" :style="{ width: lowPercent + '%' }"></div>
            </div>
          </div>
        </section>

        <!-- 司机疲劳风险排名（压缩版） -->
        <section class="card driver-list-card">
          <h2 class="card-title">司机风险排名 Top {{ filteredDrivers.length }}</h2>
          <div class="driver-list-compact">
            <div
              v-for="driver in filteredDrivers"
              :key="driver.id"
              class="driver-item-compact"
              :class="{ active: currentDriver?.id === driver.id }"
              @click="selectDriver(driver)"
            >
              <div class="driver-main">
                <span class="driver-name">{{ driver.name }}</span>
                <span class="driver-line">{{ driver.line }}</span>
              </div>
              <div class="driver-score" :class="driver.riskLevel">{{ driver.fatigueScore }}</div>
            </div>
          </div>
        </section>
      </aside>

      <!-- 中间列（核心专业图表区） -->
      <section class="center-column">
        <!-- 当前司机概要 -->
        <div class="driver-summary-bar">
          <div class="summary-left">
            <span class="focus-label">关注司机：</span>
            <span class="driver-name-large">{{ currentDriver?.name }}</span>
            <span class="tag">{{ currentDriver?.line }}</span>
            <span class="tag">{{ currentDriver?.shift }}</span>
          </div>
          <div class="summary-right">
            <span class="risk-badge-large" :class="currentDriver?.riskLevel">
              {{ riskStatusMap[currentDriver?.riskLevel] }}
            </span>
            <span class="drive-hours">连续驾驶 {{ currentDriver?.driveHours }}h</span>
          </div>
        </div>

        <!-- 三合一专业图表区 -->
        <div class="professional-chart-area">
          <!-- 图表1: 疲劳评分趋势（带分区背景） -->
          <div class="chart-section chart-trend">
            <h3 class="chart-title">疲劳评分趋势 · 最近30分钟</h3>
            <div class="trend-chart-professional">
              <!-- 背景分区 -->
              <div class="zone-background">
                <div class="zone safe">安全区</div>
                <div class="zone warning">警戒区</div>
                <div class="zone danger">高危区</div>
              </div>
              <!-- 折线图 -->
              <svg class="trend-svg" viewBox="0 0 700 120" preserveAspectRatio="none">
                <defs>
                  <linearGradient id="trendGradient" x1="0%" y1="0%" x2="0%" y2="100%">
                    <stop offset="0%" style="stop-color:#ff6b3d;stop-opacity:0.3" />
                    <stop offset="100%" style="stop-color:#ff6b3d;stop-opacity:0" />
                  </linearGradient>
                </defs>
                <!-- 面积 -->
                <path :d="trendAreaPath" fill="url(#trendGradient)" />
                <!-- 折线 -->
                <path :d="trendLinePath" fill="none" stroke="#ff6b3d" stroke-width="2" />
                <!-- 数据点 -->
                <circle
                  v-for="(point, idx) in trendPoints"
                  :key="idx"
                  :cx="point.x"
                  :cy="point.y"
                  r="3"
                  :fill="getPointColor(currentDriver?.fatigueTrend[idx])"
                  class="trend-point"
                />
              </svg>
            </div>
          </div>

          <!-- 图表2: 多维体征叠加图 -->
          <div class="chart-section chart-multi">
            <h3 class="chart-title">多维体征监测 · 心率/压力/眨眼率</h3>
            <div class="multi-dimension-chart">
              <div class="dimension-row">
                <div class="dimension-label">
                  <span class="label-text">心率</span>
                  <span class="label-value" :class="{ warning: currentDriver?.vitals.heartRate > 100 }">
                    {{ currentDriver?.vitals.heartRate }} bpm
                  </span>
                </div>
                <div class="dimension-graph">
                  <div class="graph-line">
                    <div
                      class="graph-fill heart-rate"
                      :style="{ width: (currentDriver?.vitals.heartRate / 120 * 100) + '%' }"
                    ></div>
                  </div>
                </div>
              </div>

              <div class="dimension-row">
                <div class="dimension-label">
                  <span class="label-text">压力指数</span>
                  <span class="label-value" :class="{ warning: currentDriver?.vitals.stressIndex > 75 }">
                    {{ currentDriver?.vitals.stressIndex }}
                  </span>
                </div>
                <div class="dimension-graph">
                  <div class="graph-line">
                    <div
                      class="graph-fill stress"
                      :style="{ width: currentDriver?.vitals.stressIndex + '%' }"
                    ></div>
                  </div>
                </div>
              </div>

              <div class="dimension-row">
                <div class="dimension-label">
                  <span class="label-text">PERCLOS</span>
                  <span class="label-value" :class="{ warning: currentDriver?.eye.perclos > 30 }">
                    {{ currentDriver?.eye.perclos }}%
                  </span>
                </div>
                <div class="dimension-graph">
                  <div class="graph-line">
                    <div
                      class="graph-fill perclos"
                      :style="{ width: currentDriver?.eye.perclos + '%' }"
                    ></div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- 图表3: 疲劳区段剖面图 -->
          <div class="chart-section chart-zone">
            <h3 class="chart-title">疲劳区段剖面 · 安全监控</h3>
            <div class="zone-ribbon">
              <div class="ribbon-track">
                <div class="ribbon-zone safe-zone"></div>
                <div class="ribbon-zone warning-zone"></div>
                <div class="ribbon-zone danger-zone"></div>
                <div
                  class="current-position"
                  :style="{ left: (currentDriver?.fatigueScore) + '%' }"
                >
                  <div class="position-marker"></div>
                  <div class="position-label">{{ currentDriver?.fatigueScore }}</div>
                </div>
              </div>
              <div class="ribbon-labels">
                <span class="zone-label">0</span>
                <span class="zone-label">60</span>
                <span class="zone-label">80</span>
                <span class="zone-label">100</span>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- 右侧列（实时告警流） -->
      <aside class="right-column">
        <section class="card alerts-card">
          <h2 class="card-title">实时告警流 · 疲劳岗位</h2>
          <div class="alerts-stream">
            <div v-for="alert in alerts" :key="alert.id" class="alert-item-enhanced">
              <div class="alert-severity-bar" :class="alert.severity"></div>
              <div class="alert-content-enhanced">
                <div class="alert-header-enhanced">
                  <span class="severity-badge-enhanced" :class="alert.severity">
                    {{ severityMap[alert.severity] }}
                  </span>
                  <span class="alert-time">{{ alert.time }}</span>
                </div>
                <div class="alert-title-enhanced">{{ alert.title }}</div>
                <div class="alert-meta-enhanced">
                  <span>{{ alert.driver }}</span>
                  <span class="separator">·</span>
                  <span>{{ alert.line }}</span>
                </div>
              </div>
            </div>
          </div>
        </section>
      </aside>
    </main>

    <!-- 底部横向三栏 -->
    <footer class="bottom-bar">
      <!-- 班次风险分布 -->
      <section class="bottom-card">
        <h3 class="bottom-card-title">班次风险分布</h3>
        <div class="shift-stats-horizontal">
          <div v-for="shift in shiftStats" :key="shift.name" class="shift-item-h">
            <div class="shift-header-h">
              <span class="shift-name-h">{{ shift.name }}</span>
              <span class="shift-total-h">{{ shift.total }}人</span>
            </div>
            <div class="shift-bar-h">
              <div class="bar-segment high" :style="{ width: (shift.high / shift.total * 100) + '%' }"></div>
              <div class="bar-segment medium" :style="{ width: (shift.medium / shift.total * 100) + '%' }"></div>
              <div class="bar-segment low" :style="{ width: (shift.low / shift.total * 100) + '%' }"></div>
            </div>
            <div class="shift-counts-h">
              <span class="count high">{{ shift.high }}</span>
              <span class="count medium">{{ shift.medium }}</span>
              <span class="count low">{{ shift.low }}</span>
            </div>
          </div>
        </div>
      </section>

      <!-- 设备在线情况 -->
      <section class="bottom-card">
        <h3 class="bottom-card-title">设备在线情况</h3>
        <div class="device-status-horizontal">
          <div class="device-item-h">
            <div class="device-info-h">
              <span class="device-name-h">穿戴设备</span>
              <span class="device-count-h">{{ devices.wearableOnline }}/{{ devices.wearableTotal }}</span>
            </div>
            <div class="device-progress-h">
              <div class="progress-bar-h" :style="{ width: (devices.wearableOnline / devices.wearableTotal * 100) + '%' }"></div>
            </div>
          </div>
          <div class="device-item-h">
            <div class="device-info-h">
              <span class="device-name-h">驾驶舱摄像头</span>
              <span class="device-count-h">{{ devices.cameraOnline }}/{{ devices.cameraTotal }}</span>
            </div>
            <div class="device-progress-h">
              <div class="progress-bar-h" :style="{ width: (devices.cameraOnline / devices.cameraTotal * 100) + '%' }"></div>
            </div>
          </div>
        </div>
      </section>

      <!-- AI 运营建议 -->
      <section class="bottom-card">
        <h3 class="bottom-card-title">AI 运营建议</h3>
        <div class="ai-suggestions-compact">
          <div class="suggestion-compact">
            <i class="suggestion-icon">💡</i>
            <span>夜班02:00-04:00高危集中，建议强制轮换</span>
          </div>
          <div class="suggestion-compact">
            <i class="suggestion-icon">⚠️</i>
            <span>B12/K23线路风险偏高，缩短驾驶时长</span>
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

// Mock 数据 - 司机列表
const drivers = ref([
  {
    id: 1,
    name: '张师傅',
    line: 'B12',
    shift: '夜班',
    riskLevel: 'high',
    fatigueScore: 87,
    driveHours: 5.5,
    vitals: { heartRate: 104, spo2: 95, temp: 36.9, stressIndex: 78 },
    eye: { perclos: 34, gazeAway: 3.2, yawnCount: 5 },
    fatigueTrend: [45, 58, 65, 72, 81, 85, 87]
  },
  {
    id: 2,
    name: '李师傅',
    line: 'K23',
    shift: '夜班',
    riskLevel: 'high',
    fatigueScore: 82,
    driveHours: 4.8,
    vitals: { heartRate: 98, spo2: 96, temp: 37.1, stressIndex: 72 },
    eye: { perclos: 28, gazeAway: 2.8, yawnCount: 4 },
    fatigueTrend: [42, 55, 62, 70, 76, 80, 82]
  },
  {
    id: 3,
    name: '王师傅',
    line: 'M05',
    shift: '白班',
    riskLevel: 'medium',
    fatigueScore: 68,
    driveHours: 3.2,
    vitals: { heartRate: 88, spo2: 97, temp: 36.7, stressIndex: 58 },
    eye: { perclos: 18, gazeAway: 1.8, yawnCount: 2 },
    fatigueTrend: [35, 42, 50, 58, 62, 66, 68]
  },
  {
    id: 4,
    name: '赵师傅',
    line: 'G08',
    shift: '早班',
    riskLevel: 'medium',
    fatigueScore: 62,
    driveHours: 2.8,
    vitals: { heartRate: 82, spo2: 98, temp: 36.6, stressIndex: 52 },
    eye: { perclos: 15, gazeAway: 1.5, yawnCount: 1 },
    fatigueTrend: [30, 38, 45, 52, 56, 60, 62]
  },
  {
    id: 5,
    name: '刘师傅',
    line: 'B12',
    shift: '夜班',
    riskLevel: 'medium',
    fatigueScore: 71,
    driveHours: 4.1,
    vitals: { heartRate: 92, spo2: 96, temp: 36.8, stressIndex: 64 },
    eye: { perclos: 22, gazeAway: 2.1, yawnCount: 3 },
    fatigueTrend: [38, 46, 54, 61, 66, 69, 71]
  },
  {
    id: 6,
    name: '陈师傅',
    line: 'K23',
    shift: '白班',
    riskLevel: 'low',
    fatigueScore: 45,
    driveHours: 2.1,
    vitals: { heartRate: 75, spo2: 98, temp: 36.5, stressIndex: 42 },
    eye: { perclos: 8, gazeAway: 0.8, yawnCount: 0 },
    fatigueTrend: [25, 30, 35, 38, 41, 43, 45]
  },
  {
    id: 7,
    name: '杨师傅',
    line: 'M05',
    shift: '早班',
    riskLevel: 'low',
    fatigueScore: 38,
    driveHours: 1.5,
    vitals: { heartRate: 72, spo2: 99, temp: 36.4, stressIndex: 35 },
    eye: { perclos: 5, gazeAway: 0.5, yawnCount: 0 },
    fatigueTrend: [20, 25, 28, 32, 35, 37, 38]
  },
  {
    id: 8,
    name: '周师傅',
    line: 'G08',
    shift: '白班',
    riskLevel: 'low',
    fatigueScore: 41,
    driveHours: 1.8,
    vitals: { heartRate: 78, spo2: 98, temp: 36.6, stressIndex: 38 },
    eye: { perclos: 6, gazeAway: 0.6, yawnCount: 0 },
    fatigueTrend: [22, 27, 31, 34, 37, 39, 41]
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
    source: '摄像头 + 轨迹偏移',
    time: '2分钟前'
  },
  {
    id: 2,
    severity: 'high',
    title: '心率持续偏高(>100bpm) + PERCLOS超过30%',
    driver: '李师傅',
    line: 'K23',
    source: '智能穿戴 + 摄像头',
    time: '5分钟前'
  },
  {
    id: 3,
    severity: 'medium',
    title: '连续驾驶超过4小时未休息',
    driver: '刘师傅',
    line: 'B12',
    source: '系统监测',
    time: '8分钟前'
  },
  {
    id: 4,
    severity: 'medium',
    title: '压力指数偏高 + 打哈欠频次增加',
    driver: '王师傅',
    line: 'M05',
    source: '智能穿戴 + 摄像头',
    time: '12分钟前'
  },
  {
    id: 5,
    severity: 'low',
    title: '视线偏移持续时间略长',
    driver: '赵师傅',
    line: 'G08',
    source: '摄像头',
    time: '15分钟前'
  }
])

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

// SVG 趋势图路径计算
const trendPoints = computed(() => {
  if (!currentDriver.value?.fatigueTrend) return []
  const trend = currentDriver.value.fatigueTrend
  const width = 700
  const height = 120
  const stepX = width / (trend.length - 1)

  return trend.map((value, index) => ({
    x: index * stepX,
    y: height - (value / 100 * height)
  }))
})

const trendLinePath = computed(() => {
  if (trendPoints.value.length === 0) return ''
  return trendPoints.value.map((p, i) =>
    i === 0 ? `M ${p.x} ${p.y}` : `L ${p.x} ${p.y}`
  ).join(' ')
})

const trendAreaPath = computed(() => {
  if (trendPoints.value.length === 0) return ''
  const points = trendPoints.value
  let path = `M ${points[0].x} 120 `
  path += points.map(p => `L ${p.x} ${p.y}`).join(' ')
  path += ` L ${points[points.length - 1].x} 120 Z`
  return path
})

const getPointColor = (value) => {
  if (value > 80) return '#ff3d3d'
  if (value > 60) return '#ffa500'
  return '#00ff88'
}

// 生命周期
onMounted(() => {
  updateTime()
  timeInterval = setInterval(updateTime, 1000)
})

onUnmounted(() => {
  if (timeInterval) {
    clearInterval(timeInterval)
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
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'PingFang SC', 'Hiragino Sans GB',
               'Microsoft YaHei', sans-serif;
  padding: 16px;
  overflow: hidden;
}

/* ========== 顶部导航 ========== */
.top-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: rgba(20, 25, 45, 0.6);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(45, 225, 255, 0.2);
  border-radius: 12px;
  padding: 12px 20px;
  margin-bottom: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
  flex-shrink: 0;
}

.nav-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.logo {
  width: 40px;
  height: 40px;
  background: linear-gradient(135deg, #2de1ff, #9b5bff);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 18px;
  color: #fff;
  box-shadow: 0 0 20px rgba(45, 225, 255, 0.5);
}

.title-group {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.main-title {
  font-size: 18px;
  font-weight: 600;
  margin: 0;
  background: linear-gradient(90deg, #2de1ff, #9b5bff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.sub-title {
  font-size: 11px;
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
  gap: 12px;
}

.filter-item {
  display: flex;
  flex-direction: column;
  gap: 3px;

  label {
    font-size: 11px;
    color: #8b92a7;
  }

  select {
    background: rgba(45, 225, 255, 0.1);
    border: 1px solid rgba(45, 225, 255, 0.3);
    border-radius: 6px;
    padding: 5px 10px;
    color: #e0e6ed;
    font-size: 12px;
    outline: none;
    cursor: pointer;
    transition: all 0.3s;

    &:hover {
      border-color: rgba(45, 225, 255, 0.5);
      background: rgba(45, 225, 255, 0.15);
    }

    option {
      background: #1a1442;
      color: #e0e6ed;
    }
  }
}

.nav-right {
  display: flex;
  align-items: center;
  gap: 16px;
}

.time-display {
  text-align: right;
}

.current-time {
  font-size: 20px;
  font-weight: 600;
  color: #2de1ff;
  font-family: 'Courier New', monospace;
}

.current-date {
  font-size: 11px;
  color: #8b92a7;
  margin-top: 2px;
}

.ai-assistant-btn {
  background: linear-gradient(135deg, #9b5bff, #2de1ff);
  border: none;
  border-radius: 8px;
  padding: 8px 16px;
  color: #fff;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;
  box-shadow: 0 4px 15px rgba(155, 91, 255, 0.4);

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(155, 91, 255, 0.6);
  }
}

/* ========== 主体内容区（一屏布局） ========== */
.main-content {
  flex: 1;
  display: grid;
  grid-template-columns: 280px 1fr 340px;
  gap: 16px;
  overflow: hidden;
  min-height: 0;
}

/* ========== 卡片通用样式 ========== */
.card {
  background: rgba(20, 25, 45, 0.5);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(45, 225, 255, 0.15);
  border-radius: 10px;
  padding: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
}

.card-title {
  font-size: 14px;
  font-weight: 600;
  color: #2de1ff;
  margin: 0 0 12px 0;
  padding-bottom: 10px;
  border-bottom: 1px solid rgba(45, 225, 255, 0.2);
}

/* ========== 左侧列 ========== */
.left-column {
  display: flex;
  flex-direction: column;
  gap: 16px;
  overflow: hidden;
}

.overview-card {
  flex-shrink: 0;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
  margin-bottom: 12px;
}

.stat-item {
  text-align: center;
}

.stat-value {
  font-size: 24px;
  font-weight: 700;
  color: #2de1ff;

  &.high {
    color: #ff6b3d;
  }

  &.medium {
    color: #ffa500;
  }
}

.stat-label {
  font-size: 11px;
  color: #8b92a7;
  margin-top: 2px;
}

.risk-distribution {
  margin-top: 12px;
}

.distribution-bar {
  display: flex;
  height: 6px;
  border-radius: 3px;
  overflow: hidden;
}

.bar-segment {
  transition: width 0.3s;

  &.high {
    background: #ff6b3d;
  }

  &.medium {
    background: #ffa500;
  }

  &.low {
    background: #00ff88;
  }
}

/* 司机列表（压缩版） */
.driver-list-card {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.driver-list-compact {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 6px;

  &::-webkit-scrollbar {
    width: 4px;
  }

  &::-webkit-scrollbar-track {
    background: rgba(45, 225, 255, 0.05);
    border-radius: 2px;
  }

  &::-webkit-scrollbar-thumb {
    background: rgba(45, 225, 255, 0.2);
    border-radius: 2px;

    &:hover {
      background: rgba(45, 225, 255, 0.3);
    }
  }
}

.driver-item-compact {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 10px;
  background: rgba(45, 225, 255, 0.05);
  border: 1px solid rgba(45, 225, 255, 0.1);
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.3s;
  flex-shrink: 0;

  &:hover {
    background: rgba(45, 225, 255, 0.1);
    border-color: rgba(45, 225, 255, 0.3);
    transform: translateX(3px);
  }

  &.active {
    background: rgba(45, 225, 255, 0.15);
    border-color: rgba(45, 225, 255, 0.5);
    box-shadow: 0 0 12px rgba(45, 225, 255, 0.2);
  }
}

.driver-main {
  display: flex;
  align-items: center;
  gap: 8px;
}

.driver-name {
  font-size: 13px;
  font-weight: 600;
  color: #e0e6ed;
}

.driver-line {
  font-size: 11px;
  color: #8b92a7;
  padding: 2px 6px;
  background: rgba(45, 225, 255, 0.1);
  border-radius: 3px;
}

.driver-score {
  font-size: 20px;
  font-weight: 700;
  min-width: 40px;
  text-align: right;

  &.high {
    color: #ff6b3d;
  }

  &.medium {
    color: #ffa500;
  }

  &.low {
    color: #00ff88;
  }
}

/* ========== 中间列（核心专业图表区） ========== */
.center-column {
  display: flex;
  flex-direction: column;
  gap: 12px;
  overflow: hidden;
  min-height: 0;
}

.driver-summary-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: rgba(20, 25, 45, 0.5);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(45, 225, 255, 0.15);
  border-radius: 10px;
  padding: 12px 16px;
  flex-shrink: 0;
}

.summary-left {
  display: flex;
  align-items: center;
  gap: 10px;
}

.focus-label {
  font-size: 12px;
  color: #8b92a7;
}

.driver-name-large {
  font-size: 16px;
  font-weight: 700;
  color: #e0e6ed;
}

.tag {
  font-size: 11px;
  padding: 3px 8px;
  background: rgba(45, 225, 255, 0.1);
  border: 1px solid rgba(45, 225, 255, 0.3);
  border-radius: 4px;
  color: #2de1ff;
}

.summary-right {
  display: flex;
  align-items: center;
  gap: 12px;
}

.risk-badge-large {
  font-size: 14px;
  font-weight: 700;
  padding: 6px 14px;
  border-radius: 6px;

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

.drive-hours {
  font-size: 12px;
  color: #8b92a7;
}

/* 三合一专业图表区 */
.professional-chart-area {
  flex: 1;
  display: grid;
  grid-template-rows: 1fr 140px 100px;
  gap: 12px;
  overflow: hidden;
  min-height: 0;
}

.chart-section {
  background: rgba(20, 25, 45, 0.5);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(45, 225, 255, 0.15);
  border-radius: 10px;
  padding: 12px;
  overflow: hidden;
}

.chart-title {
  font-size: 13px;
  font-weight: 600;
  color: #2de1ff;
  margin: 0 0 10px 0;
  padding-bottom: 8px;
  border-bottom: 1px solid rgba(45, 225, 255, 0.15);
}

/* 图表1: 疲劳评分趋势 */
.chart-trend {
  display: flex;
  flex-direction: column;
}

.trend-chart-professional {
  flex: 1;
  position: relative;
  min-height: 0;
}

.zone-background {
  position: absolute;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  z-index: 0;
}

.zone {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  padding-right: 10px;
  font-size: 10px;
  font-weight: 600;
  opacity: 0.6;

  &.safe {
    background: linear-gradient(90deg, rgba(0, 255, 136, 0.05), rgba(0, 255, 136, 0.15));
    color: #00ff88;
  }

  &.warning {
    background: linear-gradient(90deg, rgba(255, 165, 0, 0.05), rgba(255, 165, 0, 0.15));
    color: #ffa500;
  }

  &.danger {
    background: linear-gradient(90deg, rgba(255, 107, 61, 0.05), rgba(255, 107, 61, 0.15));
    color: #ff6b3d;
  }
}

.trend-svg {
  position: absolute;
  width: 100%;
  height: 100%;
  z-index: 1;
}

.trend-point {
  transition: all 0.3s;
  cursor: pointer;

  &:hover {
    r: 5;
    filter: drop-shadow(0 0 4px currentColor);
  }
}

/* 图表2: 多维体征叠加 */
.chart-multi {
  display: flex;
  flex-direction: column;
}

.multi-dimension-chart {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 8px 0;
}

.dimension-row {
  display: grid;
  grid-template-columns: 140px 1fr;
  align-items: center;
  gap: 12px;
}

.dimension-label {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.label-text {
  font-size: 12px;
  color: #8b92a7;
}

.label-value {
  font-size: 14px;
  font-weight: 700;
  color: #e0e6ed;

  &.warning {
    color: #ff6b3d;
  }
}

.dimension-graph {
  flex: 1;
}

.graph-line {
  height: 20px;
  background: rgba(45, 225, 255, 0.1);
  border-radius: 4px;
  overflow: hidden;
  position: relative;
}

.graph-fill {
  height: 100%;
  transition: width 0.5s ease;
  position: relative;

  &::after {
    content: '';
    position: absolute;
    right: 0;
    top: 0;
    width: 3px;
    height: 100%;
    background: rgba(255, 255, 255, 0.5);
  }

  &.heart-rate {
    background: linear-gradient(90deg, #2de1ff, #00ff88);
  }

  &.stress {
    background: linear-gradient(90deg, #9b5bff, #ff6b3d);
  }

  &.perclos {
    background: linear-gradient(90deg, #ffd700, #ff8c00);
  }
}

/* 图表3: 疲劳区段剖面 */
.chart-zone {
  display: flex;
  flex-direction: column;
}

.zone-ribbon {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.ribbon-track {
  position: relative;
  height: 40px;
  display: flex;
  border-radius: 6px;
  overflow: hidden;
  box-shadow: inset 0 2px 8px rgba(0, 0, 0, 0.3);
}

.ribbon-zone {
  flex: 1;

  &.safe-zone {
    flex: 0.6;
    background: linear-gradient(90deg, #00ff88, #2de1ff);
  }

  &.warning-zone {
    flex: 0.2;
    background: linear-gradient(90deg, #ffd700, #ff8c00);
  }

  &.danger-zone {
    flex: 0.2;
    background: linear-gradient(90deg, #ff8c00, #ff3d3d);
  }
}

.current-position {
  position: absolute;
  top: 50%;
  transform: translate(-50%, -50%);
  z-index: 10;
}

.position-marker {
  width: 16px;
  height: 16px;
  background: #fff;
  border: 3px solid #2de1ff;
  border-radius: 50%;
  box-shadow: 0 0 20px rgba(45, 225, 255, 0.8);
  animation: pulse 2s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.2);
    opacity: 0.8;
  }
}

.position-label {
  position: absolute;
  top: -28px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 12px;
  font-weight: 700;
  color: #2de1ff;
  background: rgba(20, 25, 45, 0.9);
  padding: 2px 8px;
  border-radius: 4px;
  white-space: nowrap;
  border: 1px solid rgba(45, 225, 255, 0.3);
}

.ribbon-labels {
  display: flex;
  justify-content: space-between;
  padding: 0 4px;
}

.zone-label {
  font-size: 10px;
  color: #8b92a7;
}

/* ========== 右侧列（实时告警流） ========== */
.right-column {
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-height: 0;
}

.alerts-card {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-height: 0;
}

.alerts-stream {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 10px;

  &::-webkit-scrollbar {
    width: 4px;
  }

  &::-webkit-scrollbar-track {
    background: rgba(45, 225, 255, 0.05);
    border-radius: 2px;
  }

  &::-webkit-scrollbar-thumb {
    background: rgba(45, 225, 255, 0.2);
    border-radius: 2px;

    &:hover {
      background: rgba(45, 225, 255, 0.3);
    }
  }
}

.alert-item-enhanced {
  display: flex;
  gap: 10px;
  padding: 10px;
  background: rgba(45, 225, 255, 0.05);
  border: 1px solid rgba(45, 225, 255, 0.1);
  border-radius: 8px;
  position: relative;
  overflow: hidden;
  transition: all 0.3s;
  flex-shrink: 0;

  &:hover {
    background: rgba(45, 225, 255, 0.08);
    border-color: rgba(45, 225, 255, 0.2);
  }
}

.alert-severity-bar {
  width: 4px;
  border-radius: 2px;
  flex-shrink: 0;

  &.high {
    background: linear-gradient(180deg, #ff6b3d, #ff3d3d);
  }

  &.medium {
    background: linear-gradient(180deg, #ffa500, #ffd700);
  }

  &.low {
    background: linear-gradient(180deg, #2de1ff, #00ff88);
  }
}

.alert-content-enhanced {
  flex: 1;
  min-width: 0;
}

.alert-header-enhanced {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 6px;
}

.severity-badge-enhanced {
  font-size: 10px;
  padding: 2px 6px;
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

.alert-time {
  font-size: 10px;
  color: #6b7280;
}

.alert-title-enhanced {
  font-size: 12px;
  color: #e0e6ed;
  margin-bottom: 6px;
  line-height: 1.4;
}

.alert-meta-enhanced {
  font-size: 10px;
  color: #8b92a7;

  .separator {
    margin: 0 4px;
  }
}

/* ========== 底部横向三栏 ========== */
.bottom-bar {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
  margin-top: 16px;
  flex-shrink: 0;
}

.bottom-card {
  background: rgba(20, 25, 45, 0.5);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(45, 225, 255, 0.15);
  border-radius: 10px;
  padding: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
}

.bottom-card-title {
  font-size: 13px;
  font-weight: 600;
  color: #2de1ff;
  margin: 0 0 10px 0;
  padding-bottom: 8px;
  border-bottom: 1px solid rgba(45, 225, 255, 0.2);
}

/* 班次风险分布 */
.shift-stats-horizontal {
  display: flex;
  gap: 12px;
}

.shift-item-h {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.shift-header-h {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 11px;
}

.shift-name-h {
  color: #e0e6ed;
  font-weight: 600;
}

.shift-total-h {
  color: #8b92a7;
}

.shift-bar-h {
  display: flex;
  height: 6px;
  border-radius: 3px;
  overflow: hidden;
}

.shift-counts-h {
  display: flex;
  justify-content: space-around;
  font-size: 11px;
  font-weight: 600;

  .count {
    &.high {
      color: #ff6b3d;
    }

    &.medium {
      color: #ffa500;
    }

    &.low {
      color: #00ff88;
    }
  }
}

/* 设备在线情况 */
.device-status-horizontal {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.device-item-h {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.device-info-h {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
}

.device-name-h {
  color: #e0e6ed;
}

.device-count-h {
  color: #2de1ff;
  font-weight: 600;
}

.device-progress-h {
  height: 6px;
  background: rgba(45, 225, 255, 0.1);
  border-radius: 3px;
  overflow: hidden;
}

.progress-bar-h {
  height: 100%;
  background: linear-gradient(90deg, #2de1ff, #9b5bff);
  border-radius: 3px;
  transition: width 0.3s;
}

/* AI 建议 */
.ai-suggestions-compact {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.suggestion-compact {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  font-size: 11px;
  line-height: 1.5;
  color: #b4bcc8;
  padding: 8px;
  background: rgba(45, 225, 255, 0.05);
  border-radius: 6px;
  border-left: 2px solid rgba(45, 225, 255, 0.3);
}

.suggestion-icon {
  font-size: 14px;
  flex-shrink: 0;
}
</style>
