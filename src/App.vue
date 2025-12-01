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

    <!-- 主体内容区 -->
    <main class="main-content">
      <!-- 左侧列 -->
      <aside class="left-column">
        <!-- 今日整体风险概况 -->
        <section class="card overview-card">
          <h2 class="card-title">今日整体风险概况</h2>
          <div class="stats-grid">
            <div class="stat-item">
              <div class="stat-value">{{ stats.total }}</div>
              <div class="stat-label">监控司机总数</div>
            </div>
            <div class="stat-item">
              <div class="stat-value high">{{ stats.high }}</div>
              <div class="stat-label">高危疲劳人数</div>
            </div>
            <div class="stat-item">
              <div class="stat-value medium">{{ stats.medium }}</div>
              <div class="stat-label">中等风险人数</div>
            </div>
          </div>

          <div class="risk-distribution">
            <div class="distribution-bar">
              <div class="bar-segment high" :style="{ width: highPercent + '%' }"></div>
              <div class="bar-segment medium" :style="{ width: mediumPercent + '%' }"></div>
              <div class="bar-segment low" :style="{ width: lowPercent + '%' }"></div>
            </div>
            <div class="distribution-legend">
              <span class="legend-item">
                <i class="legend-dot high"></i>高危 {{ highPercent }}%
              </span>
              <span class="legend-item">
                <i class="legend-dot medium"></i>中等 {{ mediumPercent }}%
              </span>
              <span class="legend-item">
                <i class="legend-dot low"></i>低风险 {{ lowPercent }}%
              </span>
            </div>
          </div>
        </section>

        <!-- 司机疲劳风险排名 -->
        <section class="card driver-list-card">
          <h2 class="card-title">司机疲劳风险排名</h2>
          <div class="driver-list">
            <div
              v-for="driver in filteredDrivers"
              :key="driver.id"
              class="driver-item"
              :class="{ active: currentDriver?.id === driver.id }"
              @click="selectDriver(driver)"
            >
              <div class="driver-info">
                <div class="driver-header">
                  <span class="driver-name">{{ driver.name }}</span>
                  <span class="risk-badge" :class="driver.riskLevel">
                    {{ riskLabelMap[driver.riskLevel] }}
                  </span>
                </div>
                <div class="driver-meta">
                  <span class="meta-item">{{ driver.line }}</span>
                  <span class="meta-separator">·</span>
                  <span class="meta-item">连续 {{ driver.driveHours }}h</span>
                </div>
              </div>
              <div class="fatigue-score" :class="driver.riskLevel">
                {{ driver.fatigueScore }}
              </div>
            </div>
          </div>
        </section>

        <!-- 设备在线情况 -->
        <section class="card device-status-card">
          <h2 class="card-title">设备在线情况</h2>
          <div class="device-item">
            <div class="device-header">
              <span class="device-name">穿戴设备</span>
              <span class="device-count">{{ devices.wearableOnline }} / {{ devices.wearableTotal }}</span>
            </div>
            <div class="device-progress">
              <div class="progress-bar" :style="{ width: (devices.wearableOnline / devices.wearableTotal * 100) + '%' }"></div>
            </div>
          </div>
          <div class="device-item">
            <div class="device-header">
              <span class="device-name">驾驶舱摄像头</span>
              <span class="device-count">{{ devices.cameraOnline }} / {{ devices.cameraTotal }}</span>
            </div>
            <div class="device-progress">
              <div class="progress-bar" :style="{ width: (devices.cameraOnline / devices.cameraTotal * 100) + '%' }"></div>
            </div>
          </div>
        </section>
      </aside>

      <!-- 中间列 -->
      <section class="center-column">
        <!-- 当前司机概要 -->
        <div class="card driver-summary">
          <div class="summary-header">
            <h2 class="driver-focus-title">当前关注司机：{{ currentDriver?.name }}</h2>
            <div class="driver-tags">
              <span class="tag">线路 {{ currentDriver?.line }}</span>
              <span class="tag">{{ currentDriver?.shift }}</span>
              <span class="tag risk" :class="currentDriver?.riskLevel">
                疲劳风险：{{ riskStatusMap[currentDriver?.riskLevel] }}
              </span>
              <span class="tag">连续驾驶 {{ currentDriver?.driveHours }} h</span>
            </div>
          </div>
        </div>

        <!-- 双列卡片 -->
        <div class="dual-cards">
          <!-- 智能穿戴体征 -->
          <div class="card vitals-card">
            <h3 class="card-title">智能穿戴体征 · 最近5分钟平均</h3>
            <div class="vitals-grid">
              <div class="vital-item">
                <div class="vital-label">心率</div>
                <div class="vital-value" :class="{ warning: currentDriver?.vitals.heartRate > 100 }">
                  {{ currentDriver?.vitals.heartRate }} <span class="unit">bpm</span>
                </div>
                <div class="vital-threshold">阈值区间：60-100</div>
              </div>
              <div class="vital-item">
                <div class="vital-label">血氧</div>
                <div class="vital-value" :class="{ warning: currentDriver?.vitals.spo2 < 94 }">
                  {{ currentDriver?.vitals.spo2 }} <span class="unit">%</span>
                </div>
                <div class="vital-threshold">低于94%视为异常</div>
              </div>
              <div class="vital-item">
                <div class="vital-label">压力指数</div>
                <div class="vital-value" :class="{ warning: currentDriver?.vitals.stressIndex > 75 }">
                  {{ currentDriver?.vitals.stressIndex }}
                </div>
                <div class="vital-threshold">结合 HRV 模型</div>
              </div>
              <div class="vital-item">
                <div class="vital-label">体温</div>
                <div class="vital-value">
                  {{ currentDriver?.vitals.temp }} <span class="unit">℃</span>
                </div>
                <div class="vital-threshold">正常范围</div>
              </div>
            </div>
          </div>

          <!-- 眼球与注意力检测 -->
          <div class="card eye-detection-card">
            <h3 class="card-title">眼球与注意力检测</h3>
            <div class="eye-content">
              <div class="gauge-container">
                <div class="gauge">
                  <div class="gauge-fill" :style="{
                    background: getGaugeGradient(currentDriver?.fatigueScore),
                    transform: `rotate(${currentDriver?.fatigueScore * 1.8}deg)`
                  }"></div>
                  <div class="gauge-cover">
                    <div class="gauge-value">{{ currentDriver?.fatigueScore }}</div>
                    <div class="gauge-label">疲劳指数</div>
                  </div>
                </div>
              </div>

              <div class="eye-metrics">
                <div class="metric-row">
                  <div class="metric-label">PERCLOS（闭眼比例）</div>
                  <div class="metric-value">{{ currentDriver?.eye.perclos }}%</div>
                  <div class="metric-bar">
                    <div class="bar-fill" :class="getMetricClass(currentDriver?.eye.perclos, 30)"
                         :style="{ width: currentDriver?.eye.perclos + '%' }"></div>
                  </div>
                </div>
                <div class="metric-row">
                  <div class="metric-label">连续凝视偏移</div>
                  <div class="metric-value">{{ currentDriver?.eye.gazeAway }} s/min</div>
                  <div class="metric-bar">
                    <div class="bar-fill" :class="getMetricClass(currentDriver?.eye.gazeAway, 3)"
                         :style="{ width: (currentDriver?.eye.gazeAway / 5 * 100) + '%' }"></div>
                  </div>
                </div>
                <div class="metric-row">
                  <div class="metric-label">打哈欠频次</div>
                  <div class="metric-value">{{ currentDriver?.eye.yawnCount }} 次/10min</div>
                  <div class="metric-bar">
                    <div class="bar-fill" :class="getMetricClass(currentDriver?.eye.yawnCount, 4)"
                         :style="{ width: (currentDriver?.eye.yawnCount / 10 * 100) + '%' }"></div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 疲劳趋势 -->
        <div class="card trend-card">
          <h3 class="card-title">过去 30 分钟疲劳趋势</h3>
          <div class="trend-chart">
            <div class="chart-bars">
              <div
                v-for="(value, index) in currentDriver?.fatigueTrend"
                :key="index"
                class="chart-bar"
              >
                <div class="bar-column"
                     :class="getTrendClass(value)"
                     :style="{ height: value + '%' }">
                </div>
                <div class="bar-label">{{ (index + 1) * 5 }}min</div>
              </div>
            </div>
            <div class="chart-legend">
              <span class="legend-item"><i class="legend-dot safe"></i>安全区 (&lt;60)</span>
              <span class="legend-item"><i class="legend-dot warning"></i>警戒区 (60-80)</span>
              <span class="legend-item"><i class="legend-dot danger"></i>高风险 (&gt;80)</span>
            </div>
          </div>
        </div>
      </section>

      <!-- 右侧列 -->
      <aside class="right-column">
        <!-- 实时告警 -->
        <section class="card alerts-card">
          <h2 class="card-title">实时告警 · 疲劳岗位</h2>
          <div class="alerts-list">
            <div v-for="alert in alerts" :key="alert.id" class="alert-item">
              <div class="alert-severity" :class="alert.severity"></div>
              <div class="alert-content">
                <div class="alert-header">
                  <span class="severity-badge" :class="alert.severity">
                    {{ severityMap[alert.severity] }}
                  </span>
                  <span class="alert-time">{{ alert.time }}</span>
                </div>
                <div class="alert-title">{{ alert.title }}</div>
                <div class="alert-meta">
                  <span>{{ alert.driver }}</span>
                  <span class="separator">·</span>
                  <span>{{ alert.line }}</span>
                  <span class="separator">·</span>
                  <span class="alert-source">{{ alert.source }}</span>
                </div>
                <button class="alert-action-btn">查看录像</button>
              </div>
            </div>
          </div>
        </section>

        <!-- 班次 & 区域风险分布 -->
        <section class="card shift-stats-card">
          <h2 class="card-title">班次 & 区域风险分布</h2>
          <div class="shift-stats">
            <div v-for="shift in shiftStats" :key="shift.name" class="shift-item">
              <div class="shift-header">
                <span class="shift-name">{{ shift.name }}</span>
                <span class="shift-total">共 {{ shift.total }} 人</span>
              </div>
              <div class="shift-bar">
                <div class="bar-segment high"
                     :style="{ width: (shift.high / shift.total * 100) + '%' }"></div>
                <div class="bar-segment medium"
                     :style="{ width: (shift.medium / shift.total * 100) + '%' }"></div>
                <div class="bar-segment low"
                     :style="{ width: (shift.low / shift.total * 100) + '%' }"></div>
              </div>
              <div class="shift-counts">
                <span class="count-item high">{{ shift.high }}</span>
                <span class="count-item medium">{{ shift.medium }}</span>
                <span class="count-item low">{{ shift.low }}</span>
              </div>
            </div>
          </div>
        </section>

        <!-- AI 运营建议 -->
        <section class="card ai-suggestions-card">
          <h2 class="card-title">AI 运营建议</h2>
          <ul class="suggestions-list">
            <li class="suggestion-item">
              夜班高危司机集中在 02:00-04:00 时段，建议强制轮换或增加休息站观察点。
            </li>
            <li class="suggestion-item">
              B12、K23 线路高风险明显偏高，建议缩短单次连续驾驶时长至 4 小时以内。
            </li>
            <li class="suggestion-item">
              对高危司机启用分级告警策略：先语音提醒，其次减速提示，严重时通知调度中心。
            </li>
            <li class="suggestion-item">
              建议在高风险时段增加智能穿戴设备检测频率，从 5 分钟缩短至 2 分钟一次。
            </li>
          </ul>
        </section>
      </aside>
    </main>
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

// 辅助函数
const getGaugeGradient = (score) => {
  if (score < 60) {
    return 'linear-gradient(90deg, #2de1ff, #00ff88)'
  } else if (score < 80) {
    return 'linear-gradient(90deg, #ffd700, #ff8c00)'
  } else {
    return 'linear-gradient(90deg, #ff8c00, #ff3d3d)'
  }
}

const getMetricClass = (value, threshold) => {
  if (value > threshold) return 'danger'
  if (value > threshold * 0.7) return 'warning'
  return 'safe'
}

const getTrendClass = (value) => {
  if (value > 80) return 'danger'
  if (value > 60) return 'warning'
  return 'safe'
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
  background: linear-gradient(135deg, #0a0e27 0%, #1a1442 50%, #0f1635 100%);
  color: #e0e6ed;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'PingFang SC', 'Hiragino Sans GB',
               'Microsoft YaHei', sans-serif;
  padding: 20px;
}

/* 顶部导航 */
.top-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: rgba(20, 25, 45, 0.6);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(45, 225, 255, 0.2);
  border-radius: 12px;
  padding: 16px 24px;
  margin-bottom: 20px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
}

.nav-left {
  display: flex;
  align-items: center;
  gap: 16px;
}

.logo {
  width: 48px;
  height: 48px;
  background: linear-gradient(135deg, #2de1ff, #9b5bff);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 20px;
  color: #fff;
  box-shadow: 0 0 20px rgba(45, 225, 255, 0.5);
}

.title-group {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.main-title {
  font-size: 20px;
  font-weight: 600;
  margin: 0;
  background: linear-gradient(90deg, #2de1ff, #9b5bff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.sub-title {
  font-size: 12px;
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
  gap: 16px;
}

.filter-item {
  display: flex;
  flex-direction: column;
  gap: 4px;

  label {
    font-size: 12px;
    color: #8b92a7;
  }

  select {
    background: rgba(45, 225, 255, 0.1);
    border: 1px solid rgba(45, 225, 255, 0.3);
    border-radius: 6px;
    padding: 6px 12px;
    color: #e0e6ed;
    font-size: 13px;
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
  gap: 20px;
}

.time-display {
  text-align: right;
}

.current-time {
  font-size: 24px;
  font-weight: 600;
  color: #2de1ff;
  font-family: 'Courier New', monospace;
}

.current-date {
  font-size: 12px;
  color: #8b92a7;
  margin-top: 2px;
}

.ai-assistant-btn {
  background: linear-gradient(135deg, #9b5bff, #2de1ff);
  border: none;
  border-radius: 8px;
  padding: 10px 20px;
  color: #fff;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;
  box-shadow: 0 4px 15px rgba(155, 91, 255, 0.4);

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(155, 91, 255, 0.6);
  }
}

/* 主体内容 */
.main-content {
  display: grid;
  grid-template-columns: 320px 1fr 360px;
  gap: 20px;
}

/* 卡片通用样式 */
.card {
  background: rgba(20, 25, 45, 0.5);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(45, 225, 255, 0.15);
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
  margin-bottom: 20px;
}

.card-title {
  font-size: 16px;
  font-weight: 600;
  color: #2de1ff;
  margin: 0 0 16px 0;
  padding-bottom: 12px;
  border-bottom: 1px solid rgba(45, 225, 255, 0.2);
}

/* 左侧列 */
.left-column {
  display: flex;
  flex-direction: column;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
  margin-bottom: 20px;
}

.stat-item {
  text-align: center;
}

.stat-value {
  font-size: 28px;
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
  font-size: 12px;
  color: #8b92a7;
  margin-top: 4px;
}

.risk-distribution {
  margin-top: 16px;
}

.distribution-bar {
  display: flex;
  height: 8px;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 12px;
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

.distribution-legend {
  display: flex;
  justify-content: space-around;
  font-size: 11px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 4px;
  color: #8b92a7;
}

.legend-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;

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

/* 司机列表 */
.driver-list-card {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.driver-list {
  flex: 1;
  overflow-y: auto;

  &::-webkit-scrollbar {
    width: 6px;
  }

  &::-webkit-scrollbar-track {
    background: rgba(45, 225, 255, 0.05);
    border-radius: 3px;
  }

  &::-webkit-scrollbar-thumb {
    background: rgba(45, 225, 255, 0.2);
    border-radius: 3px;

    &:hover {
      background: rgba(45, 225, 255, 0.3);
    }
  }
}

.driver-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px;
  margin-bottom: 8px;
  background: rgba(45, 225, 255, 0.05);
  border: 1px solid rgba(45, 225, 255, 0.1);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s;

  &:hover {
    background: rgba(45, 225, 255, 0.1);
    border-color: rgba(45, 225, 255, 0.3);
    transform: translateX(4px);
  }

  &.active {
    background: rgba(45, 225, 255, 0.15);
    border-color: rgba(45, 225, 255, 0.5);
    box-shadow: 0 0 15px rgba(45, 225, 255, 0.2);
  }
}

.driver-info {
  flex: 1;
}

.driver-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 6px;
}

.driver-name {
  font-size: 14px;
  font-weight: 600;
  color: #e0e6ed;
}

.risk-badge {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 4px;
  font-weight: 500;

  &.high {
    background: rgba(255, 107, 61, 0.2);
    color: #ff6b3d;
    border: 1px solid rgba(255, 107, 61, 0.4);
  }

  &.medium {
    background: rgba(255, 165, 0, 0.2);
    color: #ffa500;
    border: 1px solid rgba(255, 165, 0, 0.4);
  }

  &.low {
    background: rgba(0, 255, 136, 0.2);
    color: #00ff88;
    border: 1px solid rgba(0, 255, 136, 0.4);
  }
}

.driver-meta {
  font-size: 11px;
  color: #8b92a7;
}

.meta-separator {
  margin: 0 6px;
}

.fatigue-score {
  font-size: 24px;
  font-weight: 700;
  min-width: 50px;
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

/* 设备状态 */
.device-item {
  margin-bottom: 16px;

  &:last-child {
    margin-bottom: 0;
  }
}

.device-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
  font-size: 13px;
}

.device-name {
  color: #e0e6ed;
}

.device-count {
  color: #2de1ff;
  font-weight: 600;
}

.device-progress {
  height: 6px;
  background: rgba(45, 225, 255, 0.1);
  border-radius: 3px;
  overflow: hidden;
}

.progress-bar {
  height: 100%;
  background: linear-gradient(90deg, #2de1ff, #9b5bff);
  border-radius: 3px;
  transition: width 0.3s;
}

/* 中间列 */
.center-column {
  display: flex;
  flex-direction: column;
}

.driver-summary {
  margin-bottom: 16px;
}

.summary-header {
  border: none;
}

.driver-focus-title {
  font-size: 18px;
  font-weight: 600;
  color: #e0e6ed;
  margin: 0 0 12px 0;
}

.driver-tags {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.tag {
  font-size: 12px;
  padding: 4px 12px;
  background: rgba(45, 225, 255, 0.1);
  border: 1px solid rgba(45, 225, 255, 0.3);
  border-radius: 6px;
  color: #2de1ff;

  &.risk {
    font-weight: 600;

    &.high {
      background: rgba(255, 107, 61, 0.15);
      border-color: rgba(255, 107, 61, 0.4);
      color: #ff6b3d;
    }

    &.medium {
      background: rgba(255, 165, 0, 0.15);
      border-color: rgba(255, 165, 0, 0.4);
      color: #ffa500;
    }

    &.low {
      background: rgba(0, 255, 136, 0.15);
      border-color: rgba(0, 255, 136, 0.4);
      color: #00ff88;
    }
  }
}

.dual-cards {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
  margin-bottom: 16px;
}

/* 体征卡片 */
.vitals-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
}

.vital-item {
  background: rgba(45, 225, 255, 0.05);
  border: 1px solid rgba(45, 225, 255, 0.1);
  border-radius: 8px;
  padding: 12px;
  text-align: center;
}

.vital-label {
  font-size: 12px;
  color: #8b92a7;
  margin-bottom: 8px;
}

.vital-value {
  font-size: 24px;
  font-weight: 700;
  color: #2de1ff;
  margin-bottom: 4px;

  &.warning {
    color: #ff6b3d;
  }

  .unit {
    font-size: 14px;
    font-weight: 400;
    margin-left: 2px;
  }
}

.vital-threshold {
  font-size: 10px;
  color: #6b7280;
}

/* 眼球检测卡片 */
.eye-content {
  display: flex;
  gap: 20px;
}

.gauge-container {
  flex-shrink: 0;
}

.gauge {
  width: 140px;
  height: 140px;
  position: relative;
  background: rgba(45, 225, 255, 0.05);
  border-radius: 50%;
  border: 3px solid rgba(45, 225, 255, 0.2);
  overflow: hidden;
}

.gauge-fill {
  position: absolute;
  width: 100%;
  height: 100%;
  border-radius: 50%;
  transform-origin: center;
  clip-path: polygon(50% 50%, 50% 0%, 100% 0%, 100% 100%, 0% 100%, 0% 0%, 50% 0%);
  transition: all 0.5s;
}

.gauge-cover {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 100px;
  height: 100px;
  background: rgba(20, 25, 45, 0.95);
  border-radius: 50%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.gauge-value {
  font-size: 32px;
  font-weight: 700;
  color: #2de1ff;
}

.gauge-label {
  font-size: 11px;
  color: #8b92a7;
  margin-top: 4px;
}

.eye-metrics {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.metric-row {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.metric-label {
  font-size: 12px;
  color: #8b92a7;
}

.metric-value {
  font-size: 16px;
  font-weight: 600;
  color: #e0e6ed;
}

.metric-bar {
  height: 6px;
  background: rgba(45, 225, 255, 0.1);
  border-radius: 3px;
  overflow: hidden;
}

.bar-fill {
  height: 100%;
  border-radius: 3px;
  transition: width 0.3s;

  &.safe {
    background: #00ff88;
  }

  &.warning {
    background: #ffa500;
  }

  &.danger {
    background: #ff6b3d;
  }
}

/* 趋势图 */
.trend-chart {
  padding: 10px 0;
}

.chart-bars {
  display: flex;
  align-items: flex-end;
  justify-content: space-around;
  height: 150px;
  margin-bottom: 12px;
}

.chart-bar {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.bar-column {
  width: 24px;
  border-radius: 4px 4px 0 0;
  transition: all 0.3s;
  min-height: 10%;

  &.safe {
    background: linear-gradient(180deg, #00ff88, #2de1ff);
  }

  &.warning {
    background: linear-gradient(180deg, #ffa500, #ffd700);
  }

  &.danger {
    background: linear-gradient(180deg, #ff3d3d, #ff6b3d);
  }
}

.bar-label {
  font-size: 11px;
  color: #8b92a7;
}

.chart-legend {
  display: flex;
  justify-content: center;
  gap: 20px;
  font-size: 12px;
  margin-top: 16px;
  padding-top: 16px;
  border-top: 1px solid rgba(45, 225, 255, 0.1);

  .legend-dot {
    width: 10px;
    height: 10px;
    border-radius: 2px;

    &.safe {
      background: #00ff88;
    }

    &.warning {
      background: #ffa500;
    }

    &.danger {
      background: #ff6b3d;
    }
  }
}

/* 右侧列 */
.right-column {
  display: flex;
  flex-direction: column;
}

/* 告警列表 */
.alerts-card {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.alerts-list {
  flex: 1;
  overflow-y: auto;

  &::-webkit-scrollbar {
    width: 6px;
  }

  &::-webkit-scrollbar-track {
    background: rgba(45, 225, 255, 0.05);
    border-radius: 3px;
  }

  &::-webkit-scrollbar-thumb {
    background: rgba(45, 225, 255, 0.2);
    border-radius: 3px;

    &:hover {
      background: rgba(45, 225, 255, 0.3);
    }
  }
}

.alert-item {
  display: flex;
  gap: 12px;
  padding: 12px;
  margin-bottom: 12px;
  background: rgba(45, 225, 255, 0.05);
  border: 1px solid rgba(45, 225, 255, 0.1);
  border-radius: 8px;
  position: relative;
  overflow: hidden;
}

.alert-severity {
  width: 4px;
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;

  &.high {
    background: #ff6b3d;
  }

  &.medium {
    background: #ffa500;
  }

  &.low {
    background: #2de1ff;
  }
}

.alert-content {
  flex: 1;
  padding-left: 8px;
}

.alert-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.severity-badge {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 4px;
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
  font-size: 11px;
  color: #6b7280;
}

.alert-title {
  font-size: 13px;
  color: #e0e6ed;
  margin-bottom: 8px;
  line-height: 1.4;
}

.alert-meta {
  font-size: 11px;
  color: #8b92a7;
  margin-bottom: 10px;

  .separator {
    margin: 0 4px;
  }
}

.alert-source {
  font-style: italic;
}

.alert-action-btn {
  background: rgba(45, 225, 255, 0.1);
  border: 1px solid rgba(45, 225, 255, 0.3);
  border-radius: 4px;
  padding: 4px 12px;
  font-size: 11px;
  color: #2de1ff;
  cursor: pointer;
  transition: all 0.3s;

  &:hover {
    background: rgba(45, 225, 255, 0.2);
    border-color: rgba(45, 225, 255, 0.5);
  }
}

/* 班次统计 */
.shift-item {
  margin-bottom: 20px;

  &:last-child {
    margin-bottom: 0;
  }
}

.shift-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
  font-size: 13px;
}

.shift-name {
  color: #e0e6ed;
  font-weight: 600;
}

.shift-total {
  color: #8b92a7;
}

.shift-bar {
  display: flex;
  height: 8px;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 6px;
}

.shift-counts {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  padding: 0 4px;
}

.count-item {
  font-weight: 600;

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

/* AI 建议 */
.suggestions-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.suggestion-item {
  font-size: 13px;
  line-height: 1.6;
  color: #b4bcc8;
  margin-bottom: 16px;
  padding-left: 16px;
  position: relative;

  &:last-child {
    margin-bottom: 0;
  }

  &::before {
    content: '';
    position: absolute;
    left: 0;
    top: 8px;
    width: 6px;
    height: 6px;
    background: linear-gradient(135deg, #2de1ff, #9b5bff);
    border-radius: 50%;
  }
}
</style>
