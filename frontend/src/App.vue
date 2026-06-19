<template>
  <div class="app-container">
    <!-- 顶部导航栏 -->
    <header class="app-header">
      <div class="header-left">
        <el-icon :size="24" class="text-cyan-400"><Monitor /></el-icon>
        <h1 class="app-title">OPC-UA 工业节点浏览与数据采集</h1>
      </div>
      <div class="header-right">
        <el-badge :value="store.activeAlarmsCount" :max="99" class="alarm-badge">
          <el-icon :size="20" class="text-yellow-400"><Bell /></el-icon>
        </el-badge>
        <el-tag type="success" v-if="store.isConnected" class="status-tag">
          <el-icon><CircleCheck /></el-icon>
          在线
        </el-tag>
        <el-tag type="danger" v-else class="status-tag">
          <el-icon><CircleClose /></el-icon>
          离线
        </el-tag>
        <el-button
          :type="store.isConnected ? 'danger' : 'success'"
          size="small"
          @click="toggleConnection"
        >
          {{ store.isConnected ? '断开' : '连接' }}
        </el-button>
      </div>
    </header>

    <!-- 主内容区域 -->
    <div class="app-main">
      <!-- 左侧面板: 节点树 -->
      <aside class="left-panel">
        <NodeTree />
      </aside>

      <!-- 中央区域: 仪表盘 -->
      <main class="center-panel">
        <DataDashboard />
      </main>

      <!-- 右侧面板: 报警 & 订阅 -->
      <aside class="right-panel">
        <el-tabs v-model="activeTab" class="right-tabs">
          <!-- 报警事件 Tab -->
          <el-tab-pane label="报警事件" name="alarm">
            <div class="alarm-panel">
              <div class="alarm-header">
                <h3 class="text-lg font-bold text-yellow-400">报警事件</h3>
                <el-button type="danger" size="small" text @click="store.clearAlarms()" v-if="store.alarms.length > 0">
                  清空
                </el-button>
              </div>

              <div class="alarm-stats">
                <el-tag type="danger" size="small">严重: {{ criticalCount }}</el-tag>
                <el-tag type="warning" size="small">活跃: {{ store.activeAlarmsCount }}</el-tag>
                <el-tag type="info" size="small">总计: {{ store.alarms.length }}</el-tag>
              </div>

              <div class="alarm-list">
                <div
                  v-for="alarm in store.alarms"
                  :key="alarm.id"
                  class="alarm-item"
                  :class="{
                    'alarm-critical': alarm.severity === 'Critical',
                    'alarm-high': alarm.severity === 'High',
                    'alarm-medium': alarm.severity === 'Medium',
                    'alarm-acknowledged': alarm.acknowledged
                  }"
                >
                  <div class="alarm-item-header">
                    <el-tag
                      :type="getSeverityType(alarm.severity)"
                      size="small"
                      effect="dark"
                    >
                      {{ getSeverityLabel(alarm.severity) }}
                    </el-tag>
                    <span class="alarm-time">{{ formatTime(alarm.timestamp) }}</span>
                  </div>
                  <div class="alarm-item-body">
                    <span class="alarm-node">{{ alarm.nodeName }}</span>
                    <p class="alarm-message">{{ alarm.message }}</p>
                  </div>
                  <el-button
                    v-if="!alarm.acknowledged"
                    type="primary"
                    size="small"
                    text
                    @click="store.acknowledgeAlarm(alarm.id)"
                  >
                    确认
                  </el-button>
                </div>

                <div v-if="store.alarms.length === 0" class="no-alarms">
                  <div class="empty-guide">
                    <div class="empty-icon">
                      <el-icon :size="60" color="#64748b">
                        <Bell />
                      </el-icon>
                    </div>
                    <div class="empty-title">暂无报警</div>
                    <div class="empty-desc">订阅关键指标并启用监控后，系统将自动检测异常并产生报警</div>
                    
                    <div class="guide-steps">
                      <div class="guide-step">
                        <div class="step-number">1</div>
                        <div class="step-content">
                          <div class="step-title">切换到「订阅列表」标签</div>
                          <div class="step-desc">查看并管理已订阅的监控节点</div>
                        </div>
                      </div>
                      <div class="guide-step">
                        <div class="step-number">2</div>
                        <div class="step-content">
                          <div class="step-title">订阅关键指标节点</div>
                          <div class="step-desc">如温度、压力、电机转速等，系统将实时监控数据</div>
                        </div>
                      </div>
                      <div class="guide-step">
                        <div class="step-number">3</div>
                        <div class="step-content">
                          <div class="step-title">系统自动检测异常</div>
                          <div class="step-desc">当数据超出阈值时自动触发报警通知</div>
                        </div>
                      </div>
                    </div>

                    <div class="empty-actions">
                      <el-button
                        type="warning"
                        size="default"
                        @click="switchToSubscription"
                        class="action-btn"
                      >
                        <el-icon><Switch /></el-icon>
                        前往订阅
                      </el-button>
                      <el-button
                        type="primary"
                        size="default"
                        @click="handleQuickSubscribe"
                        class="action-btn"
                      >
                        <el-icon><Lightning /></el-icon>
                        一键启用监控
                      </el-button>
                    </div>

                    <div class="monitoring-tip" v-if="store.subscriptions.size === 0">
                      <el-icon color="#f59e0b"><Warning /></el-icon>
                      <span>当前未订阅任何节点，请先订阅后再启用监控</span>
                    </div>
                    <div class="monitoring-tip success" v-else>
                      <el-icon color="#10b981"><CircleCheck /></el-icon>
                      <span>已订阅 {{ store.subscriptions.size }} 个节点，系统正在监控中</span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </el-tab-pane>

          <!-- 订阅列表 Tab -->
          <el-tab-pane label="订阅列表" name="subscription">
            <template #label>
              <div class="tab-label-with-badge">
                <span>订阅列表</span>
                <el-badge :value="store.subscriptions.size" :max="99" class="tab-badge" />
              </div>
            </template>
            <SubscriptionList />
          </el-tab-pane>
        </el-tabs>
      </aside>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from 'vue'
import { Monitor, Bell, CircleCheck, CircleClose, Switch, Lightning, Warning } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'
import { useOpcuaStore } from './store/opcua'
import NodeTree from './components/NodeTree.vue'
import DataDashboard from './components/DataDashboard.vue'
import SubscriptionList from './components/SubscriptionList.vue'
import type { AlarmEvent } from './types'

const store = useOpcuaStore()
const updateTimer = ref<number | null>(null)
const activeTab = ref('alarm')

const criticalCount = computed(() =>
  store.alarms.filter(a => a.severity === 'Critical' && !a.acknowledged).length
)

function switchToSubscription() {
  activeTab.value = 'subscription'
}

function handleQuickSubscribe() {
  const keyNodes = ['temp_sensor', 'pressure_transmitter', 'motor_speed', 'flow_meter']
  let count = 0
  keyNodes.forEach(nodeId => {
    if (!store.subscriptions.has(nodeId)) {
      store.addSubscription(nodeId)
      count++
    }
  })
  if (count > 0) {
    ElMessage.success(`已快速订阅 ${count} 个关键指标，监控已启用`)
  } else {
    ElMessage.info('关键指标已全部订阅')
  }
}

function toggleConnection() {
  if (store.isConnected) {
    store.disconnect()
    if (updateTimer.value) {
      clearInterval(updateTimer.value)
      updateTimer.value = null
    }
    ElMessage.warning('已断开 OPC-UA 连接')
  } else {
    store.connect()
    startSimulation()
    ElMessage.success('已连接 OPC-UA 服务器')
  }
}

function startSimulation() {
  updateTimer.value = window.setInterval(() => {
    store.simulateDataUpdate()
  }, 1000)
}

function getSeverityType(severity: AlarmEvent['severity']) {
  switch (severity) {
    case 'Critical': return 'danger'
    case 'High': return 'danger'
    case 'Medium': return 'warning'
    case 'Low': return 'info'
    case 'Info': return 'info'
  }
}

function getSeverityLabel(severity: AlarmEvent['severity']) {
  switch (severity) {
    case 'Critical': return '严重'
    case 'High': return '高'
    case 'Medium': return '中'
    case 'Low': return '低'
    case 'Info': return '信息'
  }
}

function formatTime(timestamp: number): string {
  return new Date(timestamp).toLocaleTimeString('zh-CN', { hour12: false })
}

onMounted(() => {
  store.connect()
  startSimulation()
})

onUnmounted(() => {
  if (updateTimer.value) {
    clearInterval(updateTimer.value)
  }
})
</script>

<style scoped>
.app-container {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #0f172a;
  overflow: hidden;
}

.app-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 20px;
  height: 56px;
  background: rgba(15, 23, 42, 0.95);
  border-bottom: 1px solid rgba(71, 85, 105, 0.5);
  flex-shrink: 0;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.app-title {
  font-size: 18px;
  font-weight: bold;
  background: linear-gradient(90deg, #06b6d4, #22d3ee);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 16px;
}

.status-tag {
  display: flex;
  align-items: center;
  gap: 4px;
}

.app-main {
  flex: 1;
  display: flex;
  overflow: hidden;
}

.left-panel {
  width: 320px;
  background: rgba(30, 41, 59, 0.6);
  border-right: 1px solid rgba(71, 85, 105, 0.5);
  overflow-y: auto;
  flex-shrink: 0;
}

.center-panel {
  flex: 1;
  overflow-y: auto;
  background: #0f172a;
}

.right-panel {
  width: 360px;
  background: rgba(30, 41, 59, 0.6);
  border-left: 1px solid rgba(71, 85, 105, 0.5);
  overflow-y: auto;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
}

.right-tabs {
  height: 100%;
  display: flex;
  flex-direction: column;
}

:deep(.el-tabs__header) {
  margin: 0;
  padding: 0 12px;
  background: rgba(15, 23, 42, 0.5);
  border-bottom: 1px solid rgba(71, 85, 105, 0.5);
}

:deep(.el-tabs__nav-wrap::after) {
  display: none;
}

:deep(.el-tabs__item) {
  color: #94a3b8;
  height: 44px;
  line-height: 44px;
}

:deep(.el-tabs__item.is-active) {
  color: #22d3ee;
}

:deep(.el-tabs__active-bar) {
  background-color: #06b6d4;
}

:deep(.el-tabs__content) {
  flex: 1;
  overflow-y: auto;
}

.tab-label-with-badge {
  display: flex;
  align-items: center;
  gap: 6px;
}

.tab-badge {
  --el-badge-padding: 0 4px;
  --el-badge-size: 14px;
}

.alarm-panel {
  padding: 12px;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.alarm-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}

.alarm-stats {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
}

.alarm-list {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.alarm-item {
  background: rgba(30, 41, 59, 0.8);
  border: 1px solid rgba(71, 85, 105, 0.5);
  border-radius: 6px;
  padding: 10px;
  border-left: 3px solid #64748b;
}

.alarm-critical {
  border-left-color: #ef4444;
  background: rgba(239, 68, 68, 0.08);
}

.alarm-high {
  border-left-color: #f97316;
  background: rgba(249, 115, 22, 0.05);
}

.alarm-medium {
  border-left-color: #eab308;
  background: rgba(234, 179, 8, 0.05);
}

.alarm-acknowledged {
  opacity: 0.5;
}

.alarm-item-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 6px;
}

.alarm-time {
  font-size: 11px;
  color: #64748b;
  font-family: monospace;
}

.alarm-node {
  font-size: 12px;
  color: #94a3b8;
  font-weight: 500;
}

.alarm-message {
  font-size: 13px;
  color: #cbd5e1;
  margin-top: 4px;
}

.no-alarms {
  display: flex;
  justify-content: center;
  padding: 20px 12px;
}

.empty-guide {
  text-align: center;
  max-width: 300px;
}

.empty-icon {
  margin-bottom: 12px;
}

.empty-title {
  font-size: 16px;
  font-weight: 600;
  color: #94a3b8;
  margin-bottom: 8px;
}

.empty-desc {
  font-size: 13px;
  color: #64748b;
  margin-bottom: 20px;
}

.guide-steps {
  background: rgba(15, 23, 42, 0.6);
  border-radius: 8px;
  padding: 16px;
  margin-bottom: 16px;
  text-align: left;
}

.guide-step {
  display: flex;
  gap: 12px;
  margin-bottom: 12px;
}

.guide-step:last-child {
  margin-bottom: 0;
}

.step-number {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: linear-gradient(135deg, #eab308, #ca8a04);
  color: #fff;
  font-size: 12px;
  font-weight: bold;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.step-content {
  flex: 1;
}

.step-title {
  font-size: 13px;
  font-weight: 500;
  color: #e2e8f0;
  margin-bottom: 2px;
}

.step-desc {
  font-size: 12px;
  color: #64748b;
}

.empty-actions {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
}

.action-btn {
  flex: 1;
}

.empty-actions .action-btn:first-child {
  background: linear-gradient(135deg, #eab308, #ca8a04);
  border: none;
}

.empty-actions .action-btn:first-child:hover {
  background: linear-gradient(135deg, #ca8a04, #a16207);
}

.empty-actions .action-btn:last-child {
  background: linear-gradient(135deg, #06b6d4, #0891b2);
  border: none;
}

.empty-actions .action-btn:last-child:hover {
  background: linear-gradient(135deg, #0891b2, #0e7490);
}

.monitoring-tip {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 10px 12px;
  background: rgba(245, 158, 11, 0.1);
  border: 1px solid rgba(245, 158, 11, 0.3);
  border-radius: 6px;
  font-size: 12px;
  color: #fbbf24;
  text-align: left;
}

.monitoring-tip.success {
  background: rgba(16, 185, 129, 0.1);
  border-color: rgba(16, 185, 129, 0.3);
  color: #34d399;
}

.alarm-badge {
  cursor: pointer;
}
</style>
