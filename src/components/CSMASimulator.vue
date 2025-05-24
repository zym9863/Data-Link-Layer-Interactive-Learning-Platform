<template>
  <div class="csma-simulator">
    <div class="simulator-header">
      <h2>CSMA/CD 协议动画模拟器</h2>
      <p>模拟共享总线环境下的载波侦听多路访问/碰撞检测协议</p>
    </div>

    <div class="control-panel">
      <div class="simulation-controls">
        <button @click="startSimulation" :disabled="isSimulating" class="btn-primary">
          开始模拟
        </button>
        <button @click="resetSimulation" class="btn-secondary">
          重置
        </button>
        <button @click="addCollision" :disabled="!isSimulating" class="btn-warning">
          模拟碰撞
        </button>
      </div>
      
      <div class="parameters">
        <div class="param-group">
          <label>节点数量:</label>
          <input v-model.number="nodeCount" type="range" min="2" max="6" :disabled="isSimulating" />
          <span>{{ nodeCount }}</span>
        </div>
        <div class="param-group">
          <label>传输速度:</label>
          <input v-model.number="transmissionSpeed" type="range" min="1" max="10" :disabled="isSimulating" />
          <span>{{ transmissionSpeed }}x</span>
        </div>
        <div class="param-group">
          <label>最大退避时间:</label>
          <input v-model.number="maxBackoffTime" type="range" min="1" max="10" :disabled="isSimulating" />
          <span>{{ maxBackoffTime }}s</span>
        </div>
      </div>
    </div>

    <div class="network-topology">
      <div class="bus-container">
        <!-- 共享总线 -->
        <div class="shared-bus" :class="{ 'has-signal': busHasSignal, 'collision': hasCollision }">
          <div class="bus-signal" v-if="busSignal" :style="busSignal.style"></div>
          <div class="collision-indicator" v-if="hasCollision">碰撞!</div>
        </div>
        
        <!-- 网络节点 -->
        <div 
          v-for="node in nodes" 
          :key="node.id"
          class="network-node"
          :class="{ 
            'listening': node.state === 'listening',
            'transmitting': node.state === 'transmitting',
            'backing-off': node.state === 'backing-off',
            'waiting': node.state === 'waiting'
          }"
          :style="{ left: node.position + '%' }"
          @click="selectNode(node.id)"
        >
          <div class="node-circle">
            <span>节点{{ node.id }}</span>
          </div>
          <div class="node-status">{{ getNodeStatusText(node.state) }}</div>
          <div class="node-controls" v-if="selectedNode === node.id">
            <button @click="sendData(node.id)" :disabled="node.state !== 'idle'" class="btn-small">
              发送数据
            </button>
          </div>
        </div>
      </div>
    </div>

    <div class="simulation-info">
      <div class="info-panel">
        <h3>模拟状态</h3>
        <div class="status-grid">
          <div class="status-item">
            <strong>总线状态:</strong> 
            <span :class="getBusStatusClass()">{{ getBusStatusText() }}</span>
          </div>
          <div class="status-item">
            <strong>碰撞次数:</strong> {{ collisionCount }}
          </div>
          <div class="status-item">
            <strong>成功传输:</strong> {{ successfulTransmissions }}
          </div>
          <div class="status-item">
            <strong>总尝试次数:</strong> {{ totalAttempts }}
          </div>
        </div>
      </div>

      <div class="algorithm-steps">
        <h3>CSMA/CD 算法步骤</h3>
        <div class="steps-container">
          <div 
            v-for="(step, index) in algorithmSteps" 
            :key="index"
            class="step-item"
            :class="{ active: currentStep === index }"
          >
            <div class="step-number">{{ index + 1 }}</div>
            <div class="step-content">
              <strong>{{ step.title }}</strong>
              <p>{{ step.description }}</p>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="event-log">
      <h3>事件日志</h3>
      <div class="log-container">
        <div 
          v-for="event in eventLog" 
          :key="event.id"
          class="log-entry"
          :class="event.type"
        >
          <span class="timestamp">[{{ event.timestamp }}]</span>
          <span class="message">{{ event.message }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

interface NetworkNode {
  id: number
  position: number // 在总线上的位置百分比
  state: 'idle' | 'listening' | 'transmitting' | 'backing-off' | 'waiting'
  backoffTime: number
  transmissionProgress: number
  data: string
}

interface BusSignal {
  style: {
    left: string
    width: string
    animationDuration: string
  }
}

interface LogEvent {
  id: number
  timestamp: string
  message: string
  type: 'info' | 'warning' | 'error' | 'success'
}

const nodeCount = ref(4)
const transmissionSpeed = ref(5)
const maxBackoffTime = ref(5)
const isSimulating = ref(false)
const selectedNode = ref<number | null>(null)
const busHasSignal = ref(false)
const hasCollision = ref(false)
const busSignal = ref<BusSignal | null>(null)
const currentStep = ref(-1)
const collisionCount = ref(0)
const successfulTransmissions = ref(0)
const totalAttempts = ref(0)
const eventLog = ref<LogEvent[]>([])

let simulationInterval: number | null = null
let eventIdCounter = 0

const nodes = ref<NetworkNode[]>([])

const algorithmSteps = [
  {
    title: '载波侦听 (Carrier Sense)',
    description: '节点在发送前先侦听总线，检查是否有其他节点正在传输'
  },
  {
    title: '多路访问 (Multiple Access)',
    description: '如果总线空闲，节点开始传输数据到共享媒介'
  },
  {
    title: '碰撞检测 (Collision Detection)',
    description: '在传输过程中持续监听总线，检测是否发生碰撞'
  },
  {
    title: '碰撞处理',
    description: '如果检测到碰撞，立即停止传输并发送强化碰撞信号'
  },
  {
    title: '随机退避',
    description: '使用二进制指数退避算法，等待随机时间后重新尝试'
  }
]

onMounted(() => {
  initializeNodes()
})

onUnmounted(() => {
  if (simulationInterval) {
    clearInterval(simulationInterval)
  }
})

function initializeNodes() {
  nodes.value = []
  for (let i = 1; i <= nodeCount.value; i++) {
    nodes.value.push({
      id: i,
      position: (i * 80) / (nodeCount.value + 1),
      state: 'idle',
      backoffTime: 0,
      transmissionProgress: 0,
      data: `数据包${i}`
    })
  }
}

function startSimulation() {
  isSimulating.value = true
  addEvent('模拟开始', 'info')
  
  simulationInterval = setInterval(() => {
    updateSimulation()
  }, 100) as unknown as number
}

function resetSimulation() {
  isSimulating.value = false
  if (simulationInterval) {
    clearInterval(simulationInterval)
    simulationInterval = null
  }
  
  busHasSignal.value = false
  hasCollision.value = false
  busSignal.value = null
  currentStep.value = -1
  collisionCount.value = 0
  successfulTransmissions.value = 0
  totalAttempts.value = 0
  eventLog.value = []
  
  initializeNodes()
  addEvent('模拟重置', 'info')
}

function updateSimulation() {
  // 更新节点状态
  for (const node of nodes.value) {
    updateNodeState(node)
  }
  
  // 检查碰撞
  checkCollision()
  
  // 更新总线信号
  updateBusSignal()
}

function updateNodeState(node: NetworkNode) {
  switch (node.state) {
    case 'listening':
      currentStep.value = 0
      if (!busHasSignal.value) {
        node.state = 'transmitting'
        node.transmissionProgress = 0
        addEvent(`节点${node.id} 开始传输数据`, 'info')
        currentStep.value = 1
      }
      break
      
    case 'transmitting':
      currentStep.value = 2
      node.transmissionProgress += transmissionSpeed.value
      busHasSignal.value = true
      
      if (node.transmissionProgress >= 100) {
        node.state = 'idle'
        node.transmissionProgress = 0
        busHasSignal.value = false
        successfulTransmissions.value++
        addEvent(`节点${node.id} 成功完成传输`, 'success')
        currentStep.value = -1
      }
      break
      
    case 'backing-off':
      node.backoffTime -= 0.1
      if (node.backoffTime <= 0) {
        node.state = 'listening'
        addEvent(`节点${node.id} 退避完成，重新侦听`, 'info')
      }
      break
  }
}

function checkCollision() {
  const transmittingNodes = nodes.value.filter(node => node.state === 'transmitting')
  
  if (transmittingNodes.length > 1) {
    hasCollision.value = true
    currentStep.value = 3
    collisionCount.value++
    
    for (const node of transmittingNodes) {
      node.state = 'backing-off'
      node.backoffTime = Math.random() * maxBackoffTime.value
      node.transmissionProgress = 0
      addEvent(`节点${node.id} 检测到碰撞，开始退避 ${node.backoffTime.toFixed(1)}s`, 'warning')
    }
    
    busHasSignal.value = false
    addEvent('检测到信号碰撞！', 'error')
    currentStep.value = 4
    
    setTimeout(() => {
      hasCollision.value = false
    }, 1000)
  }
}

function updateBusSignal() {
  const transmittingNodes = nodes.value.filter(node => node.state === 'transmitting')
  
  if (transmittingNodes.length === 1) {
    const node = transmittingNodes[0]
    busSignal.value = {
      style: {
        left: '0%',
        width: node.transmissionProgress + '%',
        animationDuration: '0.5s'
      }
    }
  } else {
    busSignal.value = null
  }
}

function selectNode(nodeId: number) {
  selectedNode.value = selectedNode.value === nodeId ? null : nodeId
}

function sendData(nodeId: number) {
  const node = nodes.value.find(n => n.id === nodeId)
  if (node && node.state === 'idle') {
    node.state = 'listening'
    totalAttempts.value++
    addEvent(`节点${nodeId} 准备发送数据，开始载波侦听`, 'info')
  }
}

function addCollision() {
  // 强制让多个节点同时传输以创建碰撞
  const idleNodes = nodes.value.filter(node => node.state === 'idle').slice(0, 2)
  
  for (const node of idleNodes) {
    node.state = 'transmitting'
    node.transmissionProgress = 10
    totalAttempts.value++
  }
  
  if (idleNodes.length >= 2) {
    addEvent('手动触发碰撞场景', 'warning')
  }
}

function getNodeStatusText(state: string): string {
  const statusMap: Record<string, string> = {
    'idle': '空闲',
    'listening': '侦听中',
    'transmitting': '传输中',
    'backing-off': '退避中',
    'waiting': '等待中'
  }
  return statusMap[state] || state
}

function getBusStatusText(): string {
  if (hasCollision.value) return '碰撞'
  if (busHasSignal.value) return '忙碌'
  return '空闲'
}

function getBusStatusClass(): string {
  if (hasCollision.value) return 'status-collision'
  if (busHasSignal.value) return 'status-busy'
  return 'status-idle'
}

function addEvent(message: string, type: LogEvent['type']) {
  const now = new Date()
  eventLog.value.unshift({
    id: eventIdCounter++,
    timestamp: now.toLocaleTimeString(),
    message,
    type
  })
  
  // 保持日志条目数量限制
  if (eventLog.value.length > 50) {
    eventLog.value = eventLog.value.slice(0, 50)
  }
}
</script>

<style scoped>
.csma-simulator {
  color: var(--text-primary);
}

.simulator-header {
  text-align: center;
  margin-bottom: 2rem;
  padding: 2rem;
  background: var(--bg-glass);
  backdrop-filter: blur(10px);
  border-radius: var(--radius-xl);
  border: 1px solid var(--border-color);
  box-shadow: var(--shadow-lg);
  animation: fadeIn 0.6s ease-out;
}

.simulator-header h2 {
  margin: 0 0 1rem 0;
  font-size: 2.2rem;
  font-weight: 600;
  background: var(--gradient-primary);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.simulator-header p {
  margin: 0;
  color: var(--text-secondary);
  font-size: 1.1rem;
  font-weight: 300;
}

.control-panel {
  margin-bottom: 3rem;
  background: var(--bg-glass);
  backdrop-filter: blur(10px);
  border-radius: var(--radius-xl);
  padding: 2rem;
  border: 1px solid var(--border-color);
  box-shadow: var(--shadow-lg);
  animation: fadeIn 0.8s ease-out 0.2s both;
}

.simulation-controls {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 2rem;
  flex-wrap: wrap;
}

.simulation-controls button {
  min-width: 120px;
}

.btn-primary {
  background: var(--gradient-primary);
  color: white;
  border: none;
  font-weight: 600;
}

.btn-secondary {
  background: var(--bg-secondary);
  border-color: var(--border-color);
}

.btn-warning {
  background: var(--gradient-accent);
  color: white;
  border: none;
}

.btn-small {
  padding: 0.5rem 1rem;
  font-size: 0.875rem;
}

.parameters {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.5rem;
}

.param-group {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 1rem;
  background: var(--bg-secondary);
  border-radius: var(--radius-lg);
  border: 1px solid var(--border-color);
  transition: all 0.3s ease;
}

.param-group:hover {
  border-color: var(--border-hover);
  box-shadow: var(--shadow-md);
}

.param-group label {
  font-weight: 600;
  min-width: 100px;
  color: var(--text-secondary);
  font-size: 0.9rem;
}

.param-group input[type="range"] {
  flex: 1;
  background: transparent;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: var(--bg-tertiary);
  outline: none;
}

.param-group input[type="range"]::-webkit-slider-thumb {
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: var(--primary-color);
  cursor: pointer;
  box-shadow: var(--shadow-md);
  transition: all 0.2s ease;
}

.param-group input[type="range"]::-webkit-slider-thumb:hover {
  background: var(--primary-light);
  transform: scale(1.1);
}

.param-group span {
  min-width: 40px;
  text-align: center;
  font-weight: 600;
  color: var(--primary-light);
}

.network-topology {
  margin-bottom: 3rem;
  padding: 2rem;
  background: var(--bg-glass);
  backdrop-filter: blur(10px);
  border-radius: var(--radius-xl);
  border: 1px solid var(--border-color);
  box-shadow: var(--shadow-lg);
  animation: fadeIn 1s ease-out 0.4s both;
}

.bus-container {
  position: relative;
  height: 240px;
  background: radial-gradient(ellipse at center, rgba(99, 102, 241, 0.1) 0%, transparent 70%);
  border-radius: var(--radius-lg);
  padding: 2rem;
}

.shared-bus {
  position: absolute;
  top: 50%;
  left: 10%;
  right: 10%;
  height: 12px;
  background: var(--bg-tertiary);
  border-radius: 6px;
  transform: translateY(-50%);
  transition: all 0.3s ease;
  border: 2px solid var(--border-color);
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.1);
}

.shared-bus.has-signal {
  background: var(--success-color);
  border-color: var(--success-color);
  box-shadow: 0 0 20px rgba(16, 185, 129, 0.5);
}

.shared-bus.collision {
  background: var(--error-color);
  border-color: var(--error-color);
  box-shadow: 0 0 30px rgba(239, 68, 68, 0.8);
  animation: collision-flash 0.4s infinite;
}

@keyframes collision-flash {
  0%, 100% { 
    opacity: 1; 
    transform: translateY(-50%) scale(1);
  }
  50% { 
    opacity: 0.7; 
    transform: translateY(-50%) scale(1.05);
  }
}

.bus-signal {
  position: absolute;
  top: 0;
  height: 100%;
  background: var(--gradient-primary);
  border-radius: 6px;
  transition: all 0.2s ease;
  box-shadow: 0 0 10px rgba(99, 102, 241, 0.6);
}

.collision-indicator {
  position: absolute;
  top: -40px;
  left: 50%;
  transform: translateX(-50%);
  background: var(--error-color);
  color: white;
  padding: 0.5rem 1rem;
  border-radius: var(--radius-lg);
  font-weight: 700;
  font-size: 0.9rem;
  box-shadow: var(--shadow-lg);
  animation: collision-bounce 0.3s infinite;
}

@keyframes collision-bounce {
  0%, 100% { transform: translateX(-50%) translateY(0); }
  50% { transform: translateX(-50%) translateY(-5px); }
}

.network-node {
  position: absolute;
  top: 20px;
  transform: translateX(-50%);
  text-align: center;
  cursor: pointer;
  transition: all 0.3s ease;
}

.network-node:hover {
  transform: translateX(-50%) scale(1.05);
}

.node-circle {
  width: 70px;
  height: 70px;
  border-radius: 50%;
  background: var(--bg-secondary);
  border: 3px solid var(--border-color);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.8rem;
  font-weight: 700;
  transition: all 0.3s ease;
  margin-bottom: 0.75rem;
  box-shadow: var(--shadow-md);
  backdrop-filter: blur(10px);
  position: relative;
  overflow: hidden;
}

.node-circle::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.3), transparent);
  transition: left 0.6s ease;
}

.node-circle:hover::before {
  left: 100%;
}

.network-node.listening .node-circle {
  background: var(--warning-color);
  border-color: var(--warning-color);
  color: white;
  animation: pulse 2s infinite;
  box-shadow: 0 0 20px rgba(245, 158, 11, 0.5);
}

.network-node.transmitting .node-circle {
  background: var(--success-color);
  border-color: var(--success-color);
  color: white;
  animation: transmit 1s infinite;
  box-shadow: 0 0 25px rgba(16, 185, 129, 0.6);
}

.network-node.backing-off .node-circle {
  background: var(--error-color);
  border-color: var(--error-color);
  color: white;
  animation: backoff 0.8s infinite;
  box-shadow: 0 0 20px rgba(239, 68, 68, 0.5);
}

.network-node.waiting .node-circle {
  background: var(--secondary-color);
  border-color: var(--secondary-color);
  color: white;
  animation: waiting 1.5s infinite;
}

@keyframes pulse {
  0%, 100% { 
    transform: scale(1);
    box-shadow: 0 0 20px rgba(245, 158, 11, 0.5);
  }
  50% { 
    transform: scale(1.1);
    box-shadow: 0 0 30px rgba(245, 158, 11, 0.8);
  }
}

@keyframes transmit {
  0%, 100% { 
    box-shadow: 0 0 25px rgba(16, 185, 129, 0.6);
  }
  50% { 
    box-shadow: 0 0 40px rgba(16, 185, 129, 1);
  }
}

@keyframes backoff {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

@keyframes waiting {
  0%, 100% { 
    transform: scale(1);
  }
  50% { 
    transform: scale(0.95);
  }
}

.node-status {
  font-size: 0.8rem;
  color: var(--text-secondary);
  margin-bottom: 0.75rem;
  font-weight: 600;
  padding: 0.25rem 0.75rem;
  background: var(--bg-secondary);
  border-radius: var(--radius-md);
  border: 1px solid var(--border-color);
}

.node-controls {
  margin-top: 0.75rem;
}

.simulation-info {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2rem;
  margin-bottom: 3rem;
  animation: fadeIn 1.2s ease-out 0.6s both;
}

.info-panel, .algorithm-steps {
  background: var(--bg-glass);
  backdrop-filter: blur(10px);
  border-radius: var(--radius-xl);
  padding: 2rem;
  border: 1px solid var(--border-color);
  box-shadow: var(--shadow-lg);
  transition: all 0.3s ease;
}

.info-panel:hover, .algorithm-steps:hover {
  box-shadow: var(--shadow-xl);
  transform: translateY(-2px);
}

.info-panel h3, .algorithm-steps h3 {
  color: var(--text-primary);
  margin: 0 0 1.5rem 0;
  font-size: 1.3rem;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.info-panel h3::before {
  content: '📊';
}

.algorithm-steps h3::before {
  content: '⚡';
}

.status-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.5rem;
}

.status-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  padding: 1rem;
  background: var(--bg-secondary);
  border-radius: var(--radius-lg);
  border: 1px solid var(--border-color);
  transition: all 0.3s ease;
}

.status-item:hover {
  border-color: var(--border-hover);
  box-shadow: var(--shadow-md);
}

.status-item strong {
  color: var(--text-secondary);
  font-size: 0.9rem;
  font-weight: 600;
}

.status-idle { 
  color: var(--success-color);
  font-weight: 700;
  font-size: 1.1rem;
}

.status-busy { 
  color: var(--warning-color);
  font-weight: 700;
  font-size: 1.1rem;
}

.status-collision { 
  color: var(--error-color);
  font-weight: 700;
  font-size: 1.1rem;
}

.steps-container {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.step-item {
  display: flex;
  align-items: flex-start;
  gap: 1rem;
  padding: 1rem;
  border-radius: var(--radius-lg);
  transition: all 0.3s ease;
  background: var(--bg-secondary);
  border: 1px solid var(--border-color);
}

.step-item:hover {
  border-color: var(--border-hover);
  box-shadow: var(--shadow-md);
}

.step-item.active {
  background: var(--gradient-primary);
  color: white;
  border-color: transparent;
  box-shadow: var(--shadow-lg);
  transform: translateX(5px);
}

.step-number {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: var(--bg-tertiary);
  color: var(--text-primary);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 0.9rem;
  flex-shrink: 0;
  border: 2px solid var(--border-color);
  transition: all 0.3s ease;
}

.step-item.active .step-number {
  background: white;
  color: var(--primary-color);
  border-color: white;
}

.step-content strong {
  font-size: 1rem;
  font-weight: 600;
}

.step-content p {
  margin: 0.5rem 0 0 0;
  font-size: 0.9rem;
  opacity: 0.9;
  line-height: 1.5;
}

.event-log {
  background: var(--bg-glass);
  backdrop-filter: blur(10px);
  border-radius: var(--radius-xl);
  padding: 2rem;
  border: 1px solid var(--border-color);
  box-shadow: var(--shadow-lg);
  animation: fadeIn 1.4s ease-out 0.8s both;
}

.event-log h3 {
  color: var(--text-primary);
  margin: 0 0 1.5rem 0;
  font-size: 1.3rem;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.event-log h3::before {
  content: '📋';
}

.log-container {
  max-height: 300px;
  overflow-y: auto;
  border: 1px solid var(--border-color);
  border-radius: var(--radius-lg);
  background: var(--bg-secondary);
}

.log-entry {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.75rem 1rem;
  border-bottom: 1px solid var(--border-color);
  font-size: 0.9rem;
  transition: all 0.2s ease;
}

.log-entry:hover {
  background: var(--bg-glass);
}

.log-entry:last-child {
  border-bottom: none;
}

.log-entry.info { 
  border-left: 4px solid var(--primary-color);
}

.log-entry.success { 
  border-left: 4px solid var(--success-color);
}

.log-entry.warning { 
  border-left: 4px solid var(--warning-color);
}

.log-entry.error { 
  border-left: 4px solid var(--error-color);
}

.timestamp {
  color: var(--text-muted);
  font-family: 'JetBrains Mono', 'Fira Code', 'Courier New', monospace;
  min-width: 100px;
  font-size: 0.8rem;
}

.message {
  color: var(--text-primary);
  font-weight: 500;
}

/* 响应式设计 */
@media (max-width: 1024px) {
  .simulation-info {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 768px) {
  .control-panel {
    padding: 1.5rem;
  }
  
  .simulation-controls {
    flex-direction: column;
    align-items: center;
  }
  
  .simulation-controls button {
    width: 100%;
    max-width: 200px;
  }
  
  .parameters {
    grid-template-columns: 1fr;
  }
  
  .network-topology {
    padding: 1rem;
  }
  
  .bus-container {
    height: 200px;
    padding: 1rem;
  }
  
  .node-circle {
    width: 60px;
    height: 60px;
    font-size: 0.7rem;
  }
  
  .info-panel, .algorithm-steps {
    padding: 1.5rem;
  }
  
  .status-grid {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 480px) {
  .simulator-header {
    padding: 1.5rem;
  }
  
  .simulator-header h2 {
    font-size: 1.8rem;
  }
  
  .control-panel {
    padding: 1rem;
  }
  
  .node-circle {
    width: 50px;
    height: 50px;
    font-size: 0.6rem;
  }
  
  .bus-container {
    height: 160px;
  }
  
  .shared-bus {
    height: 8px;
  }
}
</style>
