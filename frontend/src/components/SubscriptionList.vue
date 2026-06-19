<template>
  <div class="subscription-panel">
    <div class="subscription-header">
      <h3 class="text-lg font-bold text-cyan-400">订阅列表</h3>
      <el-button
        type="primary"
        size="small"
        text
        @click="handleUnsubscribeAll"
        v-if="subscriptionNodes.length > 0"
      >
        全部取消
      </el-button>
    </div>

    <div class="subscription-stats">
      <el-tag type="primary" size="small">已订阅: {{ subscriptionNodes.length }}</el-tag>
    </div>

    <div class="subscription-list">
      <div
        v-for="sub in subscriptionNodes"
        :key="sub.id"
        class="subscription-item"
      >
        <div class="subscription-item-header">
          <span class="subscription-node-name">{{ sub.name }}</span>
          <el-tag
            :type="getQualityType(sub.quality)"
            size="small"
            effect="dark"
          >
            {{ sub.quality }}
          </el-tag>
        </div>
        <div class="subscription-item-body">
          <span class="subscription-node-value">
            {{ formatValue(sub) }}
          </span>
          <span class="subscription-node-id text-xs text-gray-500">
            {{ sub.nodeId }}
          </span>
        </div>
        <div class="subscription-item-actions">
          <el-button
            type="info"
            size="small"
            text
            @click="handleSelectNode(sub)"
          >
            查看
          </el-button>
          <el-button
            type="danger"
            size="small"
            text
            @click="handleUnsubscribe(sub.id)"
          >
            取消订阅
          </el-button>
        </div>
      </div>

      <div v-if="subscriptionNodes.length === 0" class="no-subscriptions">
        <div class="empty-guide">
          <div class="empty-icon">
            <el-icon :size="60" color="#64748b">
              <Connection />
            </el-icon>
          </div>
          <div class="empty-title">暂无订阅</div>
          <div class="empty-desc">订阅节点后可实时监控数据变化并触发报警</div>
          
          <div class="guide-steps">
            <div class="guide-step">
              <div class="step-number">1</div>
              <div class="step-content">
                <div class="step-title">在左侧节点树中选择变量节点</div>
                <div class="step-desc">选择 PLC 下的变量节点，如温度、压力等</div>
              </div>
            </div>
            <div class="guide-step">
              <div class="step-number">2</div>
              <div class="step-content">
                <div class="step-title">点击「订阅」按钮启用监控</div>
                <div class="step-desc">订阅后系统将实时采集数据并检查报警条件</div>
              </div>
            </div>
          </div>

          <el-button
            type="primary"
            size="default"
            @click="handleQuickSubscribe"
            class="quick-subscribe-btn"
          >
            <el-icon><Lightning /></el-icon>
            一键订阅关键指标
          </el-button>
          
          <div class="quick-tip">
            <el-icon color="#fbbf24"><InfoFilled /></el-icon>
            <span>建议订阅温度、压力、电机转速等关键指标</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import { Connection, Lightning, InfoFilled } from '@element-plus/icons-vue'
import { useOpcuaStore } from '../store/opcua'
import type { OPCUANode } from '../types'

const store = useOpcuaStore()

const subscriptionNodes = computed(() => {
  const nodes: OPCUANode[] = []
  store.subscriptions.forEach((_, nodeId) => {
    const node = findNodeById(nodeId)
    if (node) {
      nodes.push(node)
    }
  })
  return nodes
})

function findNodeById(id: string): OPCUANode | null {
  function search(nodes: OPCUANode[]): OPCUANode | null {
    for (const node of nodes) {
      if (node.id === id) return node
      if (node.children) {
        const found = search(node.children)
        if (found) return found
      }
    }
    return null
  }
  return search(store.nodeTree)
}

function getQualityType(quality?: string) {
  switch (quality) {
    case 'Good': return 'success'
    case 'Bad': return 'danger'
    default: return 'warning'
  }
}

function formatValue(node: OPCUANode): string {
  const realTime = store.realTimeData.get(node.id)
  const value = realTime?.value ?? node.value
  if (node.dataType === 'Boolean') {
    return value ? '运行中' : '已停止'
  }
  return `${value}${node.unit ? ' ' + node.unit : ''}`
}

function handleSelectNode(node: OPCUANode) {
  store.selectNode(node)
}

function handleUnsubscribe(nodeId: string) {
  const node = findNodeById(nodeId)
  ElMessageBox.confirm(
    `确定要取消订阅「${node?.name || nodeId}」吗？`,
    '取消订阅',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning'
    }
  ).then(() => {
    store.removeSubscription(nodeId)
    ElMessage.success('已取消订阅')
  }).catch(() => {})
}

function handleUnsubscribeAll() {
  ElMessageBox.confirm(
    '确定要取消所有订阅吗？',
    '批量取消订阅',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning'
    }
  ).then(() => {
    store.subscriptions.forEach((_, nodeId) => {
      store.removeSubscription(nodeId)
    })
    ElMessage.success('已取消所有订阅')
  }).catch(() => {})
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
    ElMessage.success(`已快速订阅 ${count} 个关键指标`)
  } else {
    ElMessage.info('关键指标已全部订阅')
  }
}
</script>

<style scoped>
.subscription-panel {
  padding: 12px;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.subscription-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}

.subscription-stats {
  margin-bottom: 12px;
}

.subscription-list {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.subscription-item {
  background: rgba(30, 41, 59, 0.8);
  border: 1px solid rgba(71, 85, 105, 0.5);
  border-radius: 6px;
  padding: 10px;
  border-left: 3px solid #06b6d4;
}

.subscription-item-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 6px;
}

.subscription-node-name {
  font-size: 13px;
  font-weight: 500;
  color: #e2e8f0;
}

.subscription-item-body {
  margin-bottom: 8px;
}

.subscription-node-value {
  display: block;
  font-family: 'Courier New', monospace;
  font-size: 16px;
  font-weight: bold;
  color: #22d3ee;
}

.subscription-node-id {
  display: block;
  margin-top: 2px;
  font-size: 11px;
  color: #64748b;
}

.subscription-item-actions {
  display: flex;
  gap: 8px;
  justify-content: flex-end;
}

.no-subscriptions {
  display: flex;
  justify-content: center;
  padding: 20px 12px;
}

.empty-guide {
  text-align: center;
  max-width: 280px;
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
  background: linear-gradient(135deg, #06b6d4, #0891b2);
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

.quick-subscribe-btn {
  width: 100%;
  margin-bottom: 12px;
  background: linear-gradient(135deg, #06b6d4, #0891b2);
  border: none;
}

.quick-subscribe-btn:hover {
  background: linear-gradient(135deg, #0891b2, #0e7490);
}

.quick-tip {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 10px 12px;
  background: rgba(251, 191, 36, 0.1);
  border: 1px solid rgba(251, 191, 36, 0.3);
  border-radius: 6px;
  font-size: 12px;
  color: #fbbf24;
  text-align: left;
}
</style>
