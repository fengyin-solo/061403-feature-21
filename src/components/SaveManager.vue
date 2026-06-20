<template>
  <div class="save-manager">
    <button class="save-btn" @click="showModal = true">
      💾 存档管理
    </button>
    
    <div v-if="showModal" class="modal-overlay" @click.self="showModal = false">
      <div class="modal-content">
        <div class="modal-header">
          <h3>存档管理</h3>
          <button class="close-btn" @click="showModal = false">✕</button>
        </div>
        
        <div class="modal-body">
          <div class="save-actions">
            <div class="manual-save">
              <input 
                v-model="newSlotName" 
                type="text" 
                placeholder="输入存档名称..."
                class="slot-input"
                @keyup.enter="handleSave"
              />
              <button class="action-btn save" @click="handleSave">保存</button>
            </div>
          </div>
          
          <div class="save-slots">
            <h4>存档列表</h4>
            <div v-if="slots.length === 0" class="empty-slots">
              暂无存档
            </div>
            <div v-else class="slots-list">
              <div 
                v-for="slot in slots" 
                :key="slot.name" 
                class="slot-item"
              >
                <div class="slot-info">
                  <span class="slot-name">{{ slot.name === 'auto' ? '📅 自动存档' : slot.name }}</span>
                  <span class="slot-meta">第 {{ slot.dayCount }} 天 · {{ formatDate(slot.savedAt) }}</span>
                </div>
                <div class="slot-actions">
                  <button class="action-btn compare" @click="handleCompare(slot.name)">📊 对比</button>
                  <button class="action-btn load" @click="handlePreviewLoad(slot.name)">加载</button>
                  <button 
                    v-if="slot.name !== 'auto'" 
                    class="action-btn delete" 
                    @click="$emit('delete', slot.name)"
                  >
                    删除
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div v-if="showPreview" class="modal-overlay" @click.self="showPreview = false">
      <div class="modal-content preview-modal">
        <div class="modal-header">
          <h3>⚠️ 回滚预览</h3>
          <button class="close-btn" @click="showPreview = false">✕</button>
        </div>
        
        <div class="modal-body">
          <div class="preview-warning">
            <p>即将回滚到存档：<strong>{{ previewSlotName }}</strong></p>
            <p>以下是回滚后的变化预览：</p>
          </div>

          <div v-if="comparison" class="comparison-content">
            <div class="comparison-section">
              <h5>📦 资源变化</h5>
              <div class="resource-list">
                <div 
                  v-for="(change, key) in comparison.resources" 
                  :key="key"
                  class="resource-item"
                  :class="{ 
                    'positive': change.isPositive, 
                    'negative': change.isNegative,
                    'same': change.isSame 
                  }"
                >
                  <span class="resource-name">{{ getResourceLabel(key) }}</span>
                  <span class="resource-values">
                    <span class="current">{{ Math.round(change.current) }}</span>
                    <span class="arrow">→</span>
                    <span class="target">{{ Math.round(change.target) }}</span>
                    <span class="diff" :class="{ 'up': change.isPositive, 'down': change.isNegative }">
                      {{ change.isPositive ? '+' : '' }}{{ change.diff > 0 ? Math.round(change.diff) : change.diff < 0 ? Math.round(change.diff) : '—' }}
                    </span>
                  </span>
                </div>
              </div>
            </div>

            <div class="comparison-section">
              <h5>⚠️ 风险变化</h5>
              <div class="risk-list">
                <div class="risk-item" :class="{ 'same': comparison.risks.dayCount.isSame }">
                  <span class="risk-name">📅 天数</span>
                  <span class="risk-values">
                    第 {{ comparison.risks.dayCount.current }} 天 → 第 {{ comparison.risks.dayCount.target }} 天
                    <span v-if="!comparison.risks.dayCount.isSame" class="risk-tag">
                      {{ comparison.risks.dayCount.target < comparison.risks.dayCount.current ? '⏪ 回退' : '⏩ 前进' }}
                    </span>
                  </span>
                </div>

                <div class="risk-item" :class="{ 'same': comparison.risks.isDay.isSame }">
                  <span class="risk-name">🌅 时段</span>
                  <span class="risk-values">
                    {{ comparison.risks.isDay.current ? '☀️ 白天' : '🌙 夜晚' }} → {{ comparison.risks.isDay.target ? '☀️ 白天' : '🌙 夜晚' }}
                    <span v-if="!comparison.risks.isDay.isSame" class="risk-tag">
                      {{ comparison.risks.isDay.target ? '🌞 更安全' : '🌑 更危险' }}
                    </span>
                  </span>
                </div>

                <div class="risk-item" :class="{ 'same': comparison.risks.isBlizzard.isSame }">
                  <span class="risk-name">🌨️ 暴风雪</span>
                  <span class="risk-values">
                    {{ comparison.risks.isBlizzard.current ? '⚠️ 进行中' : '✅ 无' }} → {{ comparison.risks.isBlizzard.target ? '⚠️ 进行中' : '✅ 无' }}
                    <span v-if="!comparison.risks.isBlizzard.isSame" class="risk-tag" :class="{ 'danger': comparison.risks.isBlizzard.target }">
                      {{ comparison.risks.isBlizzard.target ? '🔥 危险增加' : '💧 危险减少' }}
                    </span>
                  </span>
                </div>

                <div class="risk-item danger-item" :class="{ 'same': comparison.risks.danger.isSame }">
                  <span class="risk-name">💀 危险等级</span>
                  <span class="risk-values">
                    <span class="danger-level" :class="'level-' + comparison.risks.danger.level.current">
                      {{ getDangerLabel(comparison.risks.danger.level.current) }}
                    </span>
                    →
                    <span class="danger-level" :class="'level-' + comparison.risks.danger.level.target">
                      {{ getDangerLabel(comparison.risks.danger.level.target) }}
                    </span>
                    <span v-if="!comparison.risks.danger.isSame" class="risk-tag" :class="{ 'danger': comparison.risks.danger.target }">
                      {{ comparison.risks.danger.target ? '🔥 更危险' : '✅ 更安全' }}
                    </span>
                  </span>
                </div>
              </div>
            </div>

            <div class="risk-summary" v-if="getRiskSummary().length > 0">
              <h5>📋 风险摘要</h5>
              <ul>
                <li v-for="(warning, idx) in getRiskSummary()" :key="idx" :class="warning.type">
                  {{ warning.icon }} {{ warning.message }}
                </li>
              </ul>
            </div>
          </div>

          <div class="preview-actions">
            <button class="action-btn cancel" @click="showPreview = false">取消</button>
            <button class="action-btn confirm-load" @click="confirmLoad">✅ 确认回滚</button>
          </div>
        </div>
      </div>
    </div>

    <div v-if="showCompareModal" class="modal-overlay" @click.self="showCompareModal = false">
      <div class="modal-content compare-modal">
        <div class="modal-header">
          <h3>📊 存档对比</h3>
          <button class="close-btn" @click="showCompareModal = false">✕</button>
        </div>
        
        <div class="modal-body">
          <div class="compare-header">
            <div class="compare-slot">
              <label>对比存档：</label>
              <select v-model="compareSlotA" class="slot-select">
                <option value="">选择存档...</option>
                <option v-for="slot in slots" :key="slot.name" :value="slot.name">
                  {{ slot.name === 'auto' ? '📅 自动存档' : slot.name }}
                </option>
              </select>
            </div>
            <div class="compare-vs">VS</div>
            <div class="compare-slot">
              <label>目标存档：</label>
              <select v-model="compareSlotB" class="slot-select">
                <option value="">选择存档...</option>
                <option v-for="slot in slots" :key="slot.name" :value="slot.name">
                  {{ slot.name === 'auto' ? '📅 自动存档' : slot.name }}
                </option>
              </select>
            </div>
          </div>

          <div v-if="compareComparison" class="comparison-content">
            <div class="compare-labels">
              <span class="label-a">{{ getSlotDisplayName(compareSlotA) }}</span>
              <span class="label-b">{{ getSlotDisplayName(compareSlotB) }}</span>
            </div>

            <div class="comparison-section">
              <h5>📦 资源对比</h5>
              <div class="resource-list">
                <div 
                  v-for="(change, key) in compareComparison.resources" 
                  :key="key"
                  class="resource-item compare-item"
                  :class="{ 
                    'positive': change.isPositive, 
                    'negative': change.isNegative,
                    'same': change.isSame 
                  }"
                >
                  <span class="resource-name">{{ getResourceLabel(key) }}</span>
                  <span class="resource-values compare-values">
                    <span class="value-a">{{ Math.round(change.current) }}</span>
                    <span class="diff" :class="{ 'up': change.isPositive, 'down': change.isNegative }">
                      {{ change.isPositive ? '+' : '' }}{{ change.diff > 0 ? Math.round(change.diff) : change.diff < 0 ? Math.round(change.diff) : '—' }}
                    </span>
                    <span class="value-b">{{ Math.round(change.target) }}</span>
                  </span>
                </div>
              </div>
            </div>

            <div class="comparison-section">
              <h5>⚠️ 风险对比</h5>
              <div class="risk-list">
                <div class="risk-item compare-item" :class="{ 'same': compareComparison.risks.dayCount.isSame }">
                  <span class="risk-name">📅 天数</span>
                  <span class="risk-values compare-values">
                    <span class="value-a">第 {{ compareComparison.risks.dayCount.current }} 天</span>
                    <span class="value-b">第 {{ compareComparison.risks.dayCount.target }} 天</span>
                  </span>
                </div>

                <div class="risk-item compare-item" :class="{ 'same': compareComparison.risks.isDay.isSame }">
                  <span class="risk-name">🌅 时段</span>
                  <span class="risk-values compare-values">
                    <span class="value-a">{{ compareComparison.risks.isDay.current ? '☀️ 白天' : '🌙 夜晚' }}</span>
                    <span class="value-b">{{ compareComparison.risks.isDay.target ? '☀️ 白天' : '🌙 夜晚' }}</span>
                  </span>
                </div>

                <div class="risk-item compare-item" :class="{ 'same': compareComparison.risks.isBlizzard.isSame }">
                  <span class="risk-name">🌨️ 暴风雪</span>
                  <span class="risk-values compare-values">
                    <span class="value-a">{{ compareComparison.risks.isBlizzard.current ? '⚠️ 是' : '✅ 否' }}</span>
                    <span class="value-b">{{ compareComparison.risks.isBlizzard.target ? '⚠️ 是' : '✅ 否' }}</span>
                  </span>
                </div>

                <div class="risk-item danger-item compare-item" :class="{ 'same': compareComparison.risks.danger.isSame }">
                  <span class="risk-name">💀 危险等级</span>
                  <span class="risk-values compare-values">
                    <span class="value-a danger-level" :class="'level-' + compareComparison.risks.danger.level.current">
                      {{ getDangerLabel(compareComparison.risks.danger.level.current) }}
                    </span>
                    <span class="value-b danger-level" :class="'level-' + compareComparison.risks.danger.level.target">
                      {{ getDangerLabel(compareComparison.risks.danger.level.target) }}
                    </span>
                  </span>
                </div>
              </div>
            </div>
          </div>

          <div v-else-if="compareSlotA && compareSlotB" class="empty-compare">
            请选择两个不同的存档进行对比
          </div>
          <div v-else class="empty-compare">
            选择两个存档开始对比
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'

const props = defineProps({
  slots: {
    type: Array,
    default: () => []
  },
  show: {
    type: Boolean,
    default: false
  },
  getSlotData: {
    type: Function,
    required: true
  },
  getCurrentState: {
    type: Function,
    required: true
  },
  compareSaves: {
    type: Function,
    required: true
  }
})

const emit = defineEmits(['save', 'load', 'delete', 'update:show'])

const showModal = ref(props.show)
const newSlotName = ref('')
const showPreview = ref(false)
const showCompareModal = ref(false)
const previewSlotName = ref('')
const comparison = ref(null)
const compareComparison = ref(null)
const compareSlotA = ref('')
const compareSlotB = ref('')

watch(() => props.show, (val) => {
  showModal.value = val
})

watch(showModal, (val) => {
  emit('update:show', val)
})

watch([compareSlotA, compareSlotB], () => {
  if (compareSlotA.value && compareSlotB.value && compareSlotA.value !== compareSlotB.value) {
    const dataA = props.getSlotData(compareSlotA.value)
    const dataB = props.getSlotData(compareSlotB.value)
    if (dataA && dataB) {
      compareComparison.value = props.compareSaves(dataA, dataB)
    }
  } else {
    compareComparison.value = null
  }
})

function handleSave() {
  if (newSlotName.value.trim()) {
    emit('save', newSlotName.value.trim())
    newSlotName.value = ''
  }
}

function handleCompare(slotName) {
  showModal.value = false
  showCompareModal.value = true
  compareSlotA.value = slotName
  compareSlotB.value = ''
}

function handlePreviewLoad(slotName) {
  previewSlotName.value = slotName === 'auto' ? '📅 自动存档' : slotName
  const currentState = props.getCurrentState()
  const targetState = props.getSlotData(slotName)
  
  if (targetState) {
    comparison.value = props.compareSaves(currentState, targetState)
    showModal.value = false
    showPreview.value = true
  }
}

function confirmLoad() {
  emit('load', previewSlotName.value === '📅 自动存档' ? 'auto' : previewSlotName.value)
  showPreview.value = false
}

function formatDate(timestamp) {
  return new Date(timestamp).toLocaleString('zh-CN', {
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit'
  })
}

function getResourceLabel(key) {
  const labels = {
    wood: '🪵 木头',
    food: '🍖 食物',
    hide: '🦊 兽皮',
    tools: '🔧 工具',
    heat: '🔥 热量',
    temperature: '🌡️ 体温'
  }
  return labels[key] || key
}

function getDangerLabel(level) {
  const labels = {
    safe: '✅ 安全',
    danger: '⚠️ 危险',
    critical: '💀 致命'
  }
  return labels[level] || level
}

function getSlotDisplayName(name) {
  if (!name) return ''
  return name === 'auto' ? '📅 自动存档' : name
}

function getRiskSummary() {
  if (!comparison.value) return []
  
  const warnings = []
  const risks = comparison.value.risks
  const resources = comparison.value.resources

  if (risks.danger.target && !risks.danger.current) {
    warnings.push({ type: 'danger', icon: '⚠️', message: '回滚后将进入危险状态！' })
  }
  if (risks.danger.level.target === 'critical') {
    warnings.push({ type: 'danger', icon: '💀', message: '回滚后体温处于致命水平！' })
  }
  if (risks.isBlizzard.target && !risks.isBlizzard.current) {
    warnings.push({ type: 'warning', icon: '🌨️', message: '回滚后将面临暴风雪，消耗加倍！' })
  }
  if (!risks.isDay.target && risks.isDay.current) {
    warnings.push({ type: 'warning', icon: '🌙', message: '回滚后进入夜晚，无法进行采集活动！' })
  }
  if (resources.food.isNegative && resources.food.target <= 0) {
    warnings.push({ type: 'warning', icon: '🍖', message: '回滚后食物耗尽！' })
  }
  if (resources.wood.isNegative && resources.wood.target <= 0) {
    warnings.push({ type: 'warning', icon: '🪵', message: '回滚后木头耗尽！' })
  }
  if (resources.heat.isNegative && resources.heat.target <= 10) {
    warnings.push({ type: 'warning', icon: '🔥', message: '回滚后热量极低！' })
  }
  if (risks.dayCount.target < risks.dayCount.current) {
    warnings.push({ type: 'info', icon: '⏪', message: `将回退 ${risks.dayCount.current - risks.dayCount.target} 天的进度` })
  }

  return warnings
}
</script>

<style scoped>
.save-btn {
  padding: 10px 20px;
  background: linear-gradient(135deg, #3498db, #2980b9);
  border: none;
  border-radius: 10px;
  color: white;
  font-size: 14px;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.2s;
}

.save-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(52, 152, 219, 0.4);
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  backdrop-filter: blur(5px);
}

.modal-content {
  background: linear-gradient(135deg, #2c3e50, #34495e);
  border-radius: 15px;
  width: 90%;
  max-width: 500px;
  max-height: 80vh;
  overflow: hidden;
  border: 2px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.5);
}

.preview-modal,
.compare-modal {
  max-width: 600px;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.modal-header h3 {
  color: white;
  margin: 0;
}

.close-btn {
  background: none;
  border: none;
  color: rgba(255, 255, 255, 0.7);
  font-size: 20px;
  cursor: pointer;
  padding: 5px 10px;
  border-radius: 5px;
  transition: all 0.2s;
}

.close-btn:hover {
  background: rgba(255, 255, 255, 0.1);
  color: white;
}

.modal-body {
  padding: 20px;
  max-height: 60vh;
  overflow-y: auto;
}

.save-actions {
  margin-bottom: 20px;
}

.manual-save {
  display: flex;
  gap: 10px;
}

.slot-input {
  flex: 1;
  padding: 10px 15px;
  border: 2px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  background: rgba(0, 0, 0, 0.3);
  color: white;
  font-size: 14px;
  outline: none;
  transition: border-color 0.2s;
}

.slot-input:focus {
  border-color: #3498db;
}

.slot-input::placeholder {
  color: rgba(255, 255, 255, 0.4);
}

.action-btn {
  padding: 10px 20px;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.2s;
}

.action-btn.save {
  background: linear-gradient(135deg, #2ecc71, #27ae60);
  color: white;
}

.action-btn.compare {
  background: linear-gradient(135deg, #9b59b6, #8e44ad);
  color: white;
}

.action-btn.load {
  background: linear-gradient(135deg, #3498db, #2980b9);
  color: white;
}

.action-btn.delete {
  background: linear-gradient(135deg, #e74c3c, #c0392b);
  color: white;
}

.action-btn.cancel {
  background: linear-gradient(135deg, #95a5a6, #7f8c8d);
  color: white;
}

.action-btn.confirm-load {
  background: linear-gradient(135deg, #f39c12, #e67e22);
  color: white;
}

.action-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 3px 10px rgba(0, 0, 0, 0.3);
}

.save-slots h4 {
  color: rgba(255, 255, 255, 0.8);
  margin-bottom: 15px;
}

.empty-slots,
.empty-compare {
  text-align: center;
  color: rgba(255, 255, 255, 0.4);
  padding: 30px;
  font-style: italic;
}

.slots-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.slot-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px;
  background: rgba(0, 0, 0, 0.3);
  border-radius: 10px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.slot-info {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.slot-name {
  color: white;
  font-weight: bold;
}

.slot-meta {
  color: rgba(255, 255, 255, 0.5);
  font-size: 12px;
}

.slot-actions {
  display: flex;
  gap: 8px;
}

.preview-warning {
  background: rgba(243, 156, 18, 0.2);
  border: 1px solid rgba(243, 156, 18, 0.4);
  border-radius: 8px;
  padding: 15px;
  margin-bottom: 20px;
  color: white;
}

.preview-warning p {
  margin: 5px 0;
}

.preview-warning strong {
  color: #f39c12;
}

.comparison-content {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.comparison-section h5 {
  color: rgba(255, 255, 255, 0.9);
  margin: 0 0 15px 0;
  padding-bottom: 8px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.resource-list,
.risk-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.resource-item,
.risk-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 15px;
  background: rgba(0, 0, 0, 0.2);
  border-radius: 8px;
  border-left: 3px solid transparent;
  transition: all 0.2s;
}

.resource-item.positive {
  border-left-color: #2ecc71;
  background: rgba(46, 204, 113, 0.1);
}

.resource-item.negative {
  border-left-color: #e74c3c;
  background: rgba(231, 76, 60, 0.1);
}

.resource-item.same,
.risk-item.same {
  opacity: 0.6;
}

.resource-name,
.risk-name {
  color: white;
  font-weight: 500;
}

.resource-values,
.risk-values {
  display: flex;
  align-items: center;
  gap: 8px;
  color: rgba(255, 255, 255, 0.9);
  font-size: 14px;
}

.resource-values .current,
.resource-values .target {
  min-width: 30px;
  text-align: center;
}

.resource-values .arrow {
  color: rgba(255, 255, 255, 0.4);
}

.resource-values .diff {
  font-weight: bold;
  min-width: 45px;
  text-align: right;
}

.resource-values .diff.up {
  color: #2ecc71;
}

.resource-values .diff.down {
  color: #e74c3c;
}

.risk-tag {
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 12px;
  background: rgba(52, 152, 219, 0.3);
  color: #3498db;
  margin-left: 8px;
}

.risk-tag.danger {
  background: rgba(231, 76, 60, 0.3);
  color: #e74c3c;
}

.danger-level {
  padding: 2px 8px;
  border-radius: 4px;
  font-weight: bold;
}

.danger-level.level-safe {
  background: rgba(46, 204, 113, 0.3);
  color: #2ecc71;
}

.danger-level.level-danger {
  background: rgba(243, 156, 18, 0.3);
  color: #f39c12;
}

.danger-level.level-critical {
  background: rgba(231, 76, 60, 0.3);
  color: #e74c3c;
}

.risk-summary {
  background: rgba(0, 0, 0, 0.2);
  border-radius: 8px;
  padding: 15px;
}

.risk-summary h5 {
  color: rgba(255, 255, 255, 0.9);
  margin: 0 0 10px 0;
}

.risk-summary ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.risk-summary li {
  padding: 8px 12px;
  margin-bottom: 5px;
  border-radius: 6px;
  color: white;
  font-size: 14px;
}

.risk-summary li.danger {
  background: rgba(231, 76, 60, 0.2);
  border-left: 3px solid #e74c3c;
}

.risk-summary li.warning {
  background: rgba(243, 156, 18, 0.2);
  border-left: 3px solid #f39c12;
}

.risk-summary li.info {
  background: rgba(52, 152, 219, 0.2);
  border-left: 3px solid #3498db;
}

.preview-actions {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
  margin-top: 25px;
  padding-top: 20px;
  border-top: 1px solid rgba(255, 255, 255, 0.1);
}

.compare-header {
  display: flex;
  align-items: center;
  gap: 15px;
  margin-bottom: 20px;
  padding: 15px;
  background: rgba(0, 0, 0, 0.2);
  border-radius: 8px;
}

.compare-slot {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.compare-slot label {
  color: rgba(255, 255, 255, 0.7);
  font-size: 12px;
}

.slot-select {
  padding: 10px;
  border: 2px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  background: rgba(0, 0, 0, 0.3);
  color: white;
  font-size: 14px;
  outline: none;
  cursor: pointer;
}

.slot-select:focus {
  border-color: #3498db;
}

.slot-select option {
  background: #2c3e50;
  color: white;
}

.compare-vs {
  color: #f39c12;
  font-weight: bold;
  font-size: 18px;
  margin-top: 15px;
}

.compare-labels {
  display: flex;
  justify-content: space-between;
  margin-bottom: 10px;
  padding: 0 10px;
}

.compare-labels .label-a,
.compare-labels .label-b {
  font-weight: bold;
  font-size: 12px;
  padding: 4px 10px;
  border-radius: 4px;
}

.compare-labels .label-a {
  background: rgba(52, 152, 219, 0.3);
  color: #3498db;
}

.compare-labels .label-b {
  background: rgba(155, 89, 182, 0.3);
  color: #9b59b6;
}

.compare-item .compare-values {
  display: flex;
  align-items: center;
  gap: 10px;
}

.compare-values .value-a {
  background: rgba(52, 152, 219, 0.2);
  padding: 4px 10px;
  border-radius: 4px;
  min-width: 60px;
  text-align: center;
}

.compare-values .value-b {
  background: rgba(155, 89, 182, 0.2);
  padding: 4px 10px;
  border-radius: 4px;
  min-width: 60px;
  text-align: center;
}

.compare-values .diff {
  background: rgba(243, 156, 18, 0.2);
  padding: 2px 8px;
  border-radius: 4px;
  min-width: 40px;
  text-align: center;
}

@media (max-width: 600px) {
  .slot-actions {
    flex-wrap: wrap;
    justify-content: flex-end;
  }
  
  .action-btn {
    padding: 8px 12px;
    font-size: 12px;
  }
  
  .compare-header {
    flex-direction: column;
    gap: 10px;
  }
  
  .compare-vs {
    margin-top: 0;
  }
  
  .preview-actions {
    flex-direction: column;
  }
  
  .preview-actions .action-btn {
    width: 100%;
  }
}
</style>
