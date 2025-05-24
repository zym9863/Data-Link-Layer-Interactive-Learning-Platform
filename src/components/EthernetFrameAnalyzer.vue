<template>
  <div class="ethernet-analyzer">
    <div class="analyzer-header">
      <h2>以太网帧结构交互式解析器</h2>
      <p>选择或输入数据，观察以太网帧的封装过程</p>
    </div>

    <div class="input-section">
      <div class="input-controls">
        <div class="control-group">
          <label>选择示例数据：</label>
          <select v-model="selectedExample" @change="loadExample">
            <option value="">-- 自定义 --</option>
            <option value="http">HTTP 请求</option>
            <option value="ping">Ping 包</option>
            <option value="arp">ARP 请求</option>
          </select>
        </div>
        
        <div class="control-group">
          <label>目标MAC地址：</label>
          <input v-model="frameData.destMac" placeholder="FF:FF:FF:FF:FF:FF" />
        </div>
        
        <div class="control-group">
          <label>源MAC地址：</label>
          <input v-model="frameData.srcMac" placeholder="AA:BB:CC:DD:EE:FF" />
        </div>
        
        <div class="control-group">
          <label>类型/长度：</label>
          <input v-model="frameData.type" placeholder="0x0800" />
        </div>
        
        <div class="control-group">
          <label>数据负载：</label>
          <textarea v-model="frameData.payload" placeholder="输入要传输的数据..."></textarea>
        </div>
      </div>
    </div>

    <div class="frame-visualization">
      <h3>以太网帧结构可视化</h3>
      <div class="frame-container">
        <div 
          v-for="field in frameFields" 
          :key="field.name"
          :class="['frame-field', { active: selectedField === field.name }]"
          :style="{ width: field.width }"
          @click="selectField(field.name)"
        >
          <div class="field-header">{{ field.name }}</div>
          <div class="field-value">{{ field.value }}</div>
          <div class="field-bytes">{{ field.bytes }} bytes</div>
        </div>
      </div>
    </div>

    <div class="field-details" v-if="selectedField">
      <h3>字段详细信息</h3>
      <div class="detail-card">
        <h4>{{ fieldDetails[selectedField]?.name }}</h4>
        <div class="detail-item">
          <strong>长度：</strong> {{ fieldDetails[selectedField]?.length }}
        </div>
        <div class="detail-item">
          <strong>作用：</strong> {{ fieldDetails[selectedField]?.purpose }}
        </div>
        <div class="detail-item">
          <strong>当前值：</strong> <code>{{ fieldDetails[selectedField]?.currentValue }}</code>
        </div>
        <div class="detail-item">
          <strong>说明：</strong> {{ fieldDetails[selectedField]?.description }}
        </div>
      </div>
    </div>

    <div class="hex-view">
      <h3>十六进制视图</h3>
      <div class="hex-container">
        <pre>{{ hexView }}</pre>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch } from 'vue'

interface FrameData {
  destMac: string
  srcMac: string
  type: string
  payload: string
}

interface FrameField {
  name: string
  value: string
  bytes: number
  width: string
}

interface FieldDetail {
  name: string
  length: string
  purpose: string
  currentValue: string
  description: string
}

const selectedExample = ref('')
const selectedField = ref('')

const frameData = ref<FrameData>({
  destMac: 'FF:FF:FF:FF:FF:FF',
  srcMac: 'AA:BB:CC:DD:EE:FF',
  type: '0x0800',
  payload: 'Hello, Network World!'
})

const examples = {
  http: {
    destMac: '00:1B:44:11:3A:B7',
    srcMac: '00:50:56:C0:00:08',
    type: '0x0800',
    payload: 'GET /index.html HTTP/1.1\r\nHost: www.example.com\r\n\r\n'
  },
  ping: {
    destMac: '00:1B:44:11:3A:B7',
    srcMac: '00:50:56:C0:00:08',
    type: '0x0800',
    payload: 'ICMP Echo Request: ping data payload'
  },
  arp: {
    destMac: 'FF:FF:FF:FF:FF:FF',
    srcMac: '00:50:56:C0:00:08',
    type: '0x0806',
    payload: 'ARP Request: Who has 192.168.1.1?'
  }
}

const frameFields = computed<FrameField[]>(() => {
  const payload = frameData.value.payload
  const paddedPayload = payload.length < 46 ? payload + ' '.repeat(46 - payload.length) : payload
  
  return [
    {
      name: '前导码',
      value: '10101010...',
      bytes: 7,
      width: '10%'
    },
    {
      name: 'SFD',
      value: '10101011',
      bytes: 1,
      width: '8%'
    },
    {
      name: '目标MAC',
      value: frameData.value.destMac,
      bytes: 6,
      width: '15%'
    },
    {
      name: '源MAC',
      value: frameData.value.srcMac,
      bytes: 6,
      width: '15%'
    },
    {
      name: '类型/长度',
      value: frameData.value.type,
      bytes: 2,
      width: '10%'
    },
    {
      name: '数据',
      value: paddedPayload.substring(0, 20) + (paddedPayload.length > 20 ? '...' : ''),
      bytes: Math.max(46, payload.length),
      width: '30%'
    },
    {
      name: 'FCS',
      value: calculateFCS(),
      bytes: 4,
      width: '12%'
    }
  ]
})

const fieldDetails = computed<Record<string, FieldDetail>>(() => ({
  '前导码': {
    name: '前导码 (Preamble)',
    length: '7 字节',
    purpose: '同步时钟，通知接收方数据帧即将到来',
    currentValue: '10101010 10101010 10101010 10101010 10101010 10101010 10101010',
    description: '由7个字节的交替0和1组成，用于接收方的时钟同步'
  },
  'SFD': {
    name: '帧开始定界符 (Start Frame Delimiter)',
    length: '1 字节',
    purpose: '标记帧的开始',
    currentValue: '10101011',
    description: '固定模式，表示真正的帧数据即将开始'
  },
  '目标MAC': {
    name: '目标MAC地址',
    length: '6 字节',
    purpose: '指定数据帧的接收方',
    currentValue: frameData.value.destMac,
    description: '48位硬件地址，唯一标识网络接口'
  },
  '源MAC': {
    name: '源MAC地址',
    length: '6 字节',
    purpose: '标识数据帧的发送方',
    currentValue: frameData.value.srcMac,
    description: '发送方网络接口的48位硬件地址'
  },
  '类型/长度': {
    name: '类型/长度字段',
    length: '2 字节',
    purpose: '指示上层协议类型或数据长度',
    currentValue: frameData.value.type,
    description: '大于1536为类型字段(如0x0800表示IPv4)，小于等于1500为长度字段'
  },
  '数据': {
    name: '数据字段',
    length: `${Math.max(46, frameData.value.payload.length)} 字节`,
    purpose: '承载上层协议的数据',
    currentValue: frameData.value.payload,
    description: '最小46字节，最大1500字节。不足46字节时需要填充'
  },
  'FCS': {
    name: '帧校验序列 (Frame Check Sequence)',
    length: '4 字节',
    purpose: '检测传输错误',
    currentValue: calculateFCS(),
    description: '32位CRC校验码，用于检测帧在传输过程中是否出现错误'
  }
}))

const hexView = computed(() => {
  const hex = []
  let offset = 0
  
  // 前导码 + SFD
  hex.push(`${offset.toString(16).padStart(4, '0')}: AA AA AA AA AA AA AA AB  ......... (前导码+SFD)`)
  offset += 8
  
  // 目标MAC
  const destMacHex = frameData.value.destMac.replace(/:/g, ' ')
  hex.push(`${offset.toString(16).padStart(4, '0')}: ${destMacHex}        (目标MAC)`)
  offset += 6
  
  // 源MAC
  const srcMacHex = frameData.value.srcMac.replace(/:/g, ' ')
  hex.push(`${offset.toString(16).padStart(4, '0')}: ${srcMacHex}        (源MAC)`)
  offset += 6
  
  // 类型/长度
  const typeHex = frameData.value.type.replace('0x', '').padStart(4, '0')
  hex.push(`${offset.toString(16).padStart(4, '0')}: ${typeHex.substring(0, 2)} ${typeHex.substring(2, 4)}              (类型/长度)`)
  offset += 2
  
  // 数据
  const data = frameData.value.payload
  const dataBytes = []
  for (let i = 0; i < data.length; i++) {
    dataBytes.push(data.charCodeAt(i).toString(16).padStart(2, '0'))
  }
  
  // 填充到最小46字节
  while (dataBytes.length < 46) {
    dataBytes.push('00')
  }
  
  for (let i = 0; i < dataBytes.length; i += 8) {
    const chunk = dataBytes.slice(i, i + 8).join(' ')
    hex.push(`${offset.toString(16).padStart(4, '0')}: ${chunk.padEnd(23, ' ')} (数据${i === 0 ? '+填充' : ''})`)
    offset += Math.min(8, dataBytes.length - i)
  }
  
  // FCS
  hex.push(`${offset.toString(16).padStart(4, '0')}: ${calculateFCS().replace('0x', '')}        (FCS校验)`)
  
  return hex.join('\n')
})

function loadExample() {
  if (selectedExample.value && examples[selectedExample.value as keyof typeof examples]) {
    Object.assign(frameData.value, examples[selectedExample.value as keyof typeof examples])
  }
}

function selectField(fieldName: string) {
  selectedField.value = fieldName
}

function calculateFCS(): string {
  // 简化的CRC计算（实际应该使用CRC-32算法）
  const data = frameData.value.destMac + frameData.value.srcMac + frameData.value.type + frameData.value.payload
  let crc = 0
  for (let i = 0; i < data.length; i++) {
    crc = (crc + data.charCodeAt(i)) % 0xFFFFFFFF
  }
  return '0x' + crc.toString(16).toUpperCase().padStart(8, '0')
}

// 默认选择第一个字段
watch(() => frameFields.value, (newFields) => {
  if (!selectedField.value && newFields.length > 0) {
    selectedField.value = newFields[2].name // 选择目标MAC作为默认
  }
}, { immediate: true })
</script>

<style scoped>
.ethernet-analyzer {
  color: var(--text-primary);
}

.analyzer-header {
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

.analyzer-header h2 {
  margin: 0 0 1rem 0;
  font-size: 2.2rem;
  font-weight: 600;
  background: var(--gradient-primary);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.analyzer-header p {
  margin: 0;
  color: var(--text-secondary);
  font-size: 1.1rem;
  font-weight: 300;
}

.input-section {
  margin-bottom: 3rem;
  animation: fadeIn 0.8s ease-out 0.2s both;
}

.input-controls {
  background: var(--bg-glass);
  backdrop-filter: blur(10px);
  border-radius: var(--radius-xl);
  padding: 2rem;
  border: 1px solid var(--border-color);
  box-shadow: var(--shadow-lg);
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 1.5rem;
}

.control-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.control-group label {
  font-weight: 600;
  font-size: 0.95rem;
  color: var(--text-secondary);
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.control-group label::before {
  content: '';
  width: 4px;
  height: 4px;
  background: var(--primary-color);
  border-radius: 50%;
}

.control-group input,
.control-group select,
.control-group textarea {
  background: var(--bg-secondary);
  border: 1px solid var(--border-color);
  border-radius: var(--radius-md);
  padding: 0.875rem 1rem;
  color: var(--text-primary);
  font-family: 'JetBrains Mono', 'Fira Code', 'Courier New', monospace;
  font-size: 0.9rem;
  transition: all 0.3s ease;
}

.control-group input:focus,
.control-group select:focus,
.control-group textarea:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
  background: var(--bg-tertiary);
}

.control-group textarea {
  min-height: 100px;
  resize: vertical;
  line-height: 1.6;
}

.frame-visualization {
  margin-bottom: 3rem;
  animation: fadeIn 1s ease-out 0.4s both;
}

.frame-visualization h3 {
  color: var(--text-primary);
  margin-bottom: 1.5rem;
  font-size: 1.5rem;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.frame-visualization h3::before {
  content: '📊';
  font-size: 1.2rem;
}

.frame-container {
  display: flex;
  border: 2px solid var(--border-color);
  border-radius: var(--radius-xl);
  overflow: hidden;
  min-height: 120px;
  background: var(--bg-secondary);
  box-shadow: var(--shadow-lg);
  transition: all 0.3s ease;
}

.frame-container:hover {
  box-shadow: var(--shadow-xl);
  transform: translateY(-2px);
}

.frame-field {
  border-right: 2px solid var(--border-color);
  padding: 1rem;
  cursor: pointer;
  transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  display: flex;
  flex-direction: column;
  justify-content: center;
  text-align: center;
  min-height: 120px;
  background: var(--bg-glass);
  backdrop-filter: blur(10px);
  position: relative;
  overflow: hidden;
}

.frame-field::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(99, 102, 241, 0.1), transparent);
  transition: left 0.6s ease;
}

.frame-field:hover::before {
  left: 100%;
}

.frame-field:last-child {
  border-right: none;
}

.frame-field:hover {
  background: var(--gradient-primary);
  color: white;
  transform: scale(1.02) translateY(-4px);
  z-index: 2;
  box-shadow: var(--shadow-xl);
}

.frame-field.active {
  background: var(--gradient-primary);
  color: white;
  transform: scale(1.02) translateY(-2px);
  z-index: 2;
  box-shadow: 0 8px 32px rgba(99, 102, 241, 0.3);
}

.field-header {
  font-weight: 700;
  font-size: 0.85rem;
  margin-bottom: 0.5rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.field-value {
  font-family: 'JetBrains Mono', 'Fira Code', 'Courier New', monospace;
  font-size: 0.8rem;
  margin-bottom: 0.5rem;
  word-break: break-all;
  line-height: 1.4;
  background: rgba(0, 0, 0, 0.1);
  padding: 0.25rem 0.5rem;
  border-radius: var(--radius-sm);
}

.field-bytes {
  font-size: 0.7rem;
  opacity: 0.9;
  font-weight: 500;
  background: rgba(0, 0, 0, 0.2);
  padding: 0.2rem 0.4rem;
  border-radius: var(--radius-sm);
  display: inline-block;
}

.field-details {
  margin-bottom: 3rem;
  animation: fadeIn 0.6s ease-out;
}

.field-details h3 {
  color: var(--text-primary);
  margin-bottom: 1.5rem;
  font-size: 1.5rem;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.field-details h3::before {
  content: '🔍';
  font-size: 1.2rem;
}

.detail-card {
  background: var(--bg-glass);
  backdrop-filter: blur(10px);
  border-radius: var(--radius-xl);
  padding: 2rem;
  border: 1px solid var(--border-color);
  border-left: 4px solid var(--primary-color);
  box-shadow: var(--shadow-lg);
  transition: all 0.3s ease;
}

.detail-card:hover {
  border-color: var(--primary-light);
  box-shadow: var(--shadow-xl);
  transform: translateY(-2px);
}

.detail-card h4 {
  color: var(--text-primary);
  margin: 0 0 1.5rem 0;
  font-size: 1.3rem;
  font-weight: 600;
}

.detail-item {
  margin-bottom: 1rem;
  line-height: 1.7;
  padding: 0.75rem 0;
  border-bottom: 1px solid var(--border-color);
}

.detail-item:last-child {
  border-bottom: none;
  margin-bottom: 0;
}

.detail-item strong {
  color: var(--primary-light);
  font-weight: 600;
  display: inline-block;
  min-width: 80px;
}

.detail-item code {
  background: var(--bg-secondary);
  color: var(--primary-light);
  padding: 0.4rem 0.6rem;
  border-radius: var(--radius-sm);
  font-family: 'JetBrains Mono', 'Fira Code', 'Courier New', monospace;
  font-size: 0.9rem;
  border: 1px solid var(--border-color);
}

.hex-view {
  animation: fadeIn 1s ease-out 0.6s both;
}

.hex-view h3 {
  color: var(--text-primary);
  margin-bottom: 1.5rem;
  font-size: 1.5rem;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.hex-view h3::before {
  content: '🔢';
  font-size: 1.2rem;
}

.hex-container {
  background: var(--bg-secondary);
  border: 1px solid var(--border-color);
  border-radius: var(--radius-xl);
  padding: 1.5rem;
  overflow-x: auto;
  box-shadow: var(--shadow-lg);
  transition: all 0.3s ease;
}

.hex-container:hover {
  box-shadow: var(--shadow-xl);
  border-color: var(--border-hover);
}

.hex-container pre {
  margin: 0;
  font-family: 'JetBrains Mono', 'Fira Code', 'Courier New', monospace;
  font-size: 0.9rem;
  line-height: 1.6;
  color: var(--primary-light);
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
}

/* 响应式设计 */
@media (max-width: 768px) {
  .input-controls {
    grid-template-columns: 1fr;
    padding: 1.5rem;
  }
  
  .frame-container {
    flex-direction: column;
    min-height: auto;
  }
  
  .frame-field {
    border-right: none;
    border-bottom: 2px solid var(--border-color);
    min-height: 80px;
  }
  
  .frame-field:last-child {
    border-bottom: none;
  }
  
  .detail-card {
    padding: 1.5rem;
  }
  
  .hex-container {
    padding: 1rem;
  }
}

@media (max-width: 480px) {
  .analyzer-header {
    padding: 1.5rem;
  }
  
  .analyzer-header h2 {
    font-size: 1.8rem;
  }
  
  .input-controls {
    padding: 1rem;
  }
  
  .frame-field {
    padding: 0.75rem;
    min-height: 70px;
  }
  
  .field-header {
    font-size: 0.75rem;
  }
  
  .field-value {
    font-size: 0.7rem;
  }
}
</style>
