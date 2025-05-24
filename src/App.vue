<script setup lang="ts">
import { ref } from 'vue'
import EthernetFrameAnalyzer from './components/EthernetFrameAnalyzer.vue'
import CSMASimulator from './components/CSMASimulator.vue'

const activeTab = ref('ethernet')
</script>

<template>
  <div class="app">
    <header class="app-header">
      <h1>数据链路层交互式学习平台</h1>
      <p>Data Link Layer Interactive Learning Platform</p>
    </header>

    <nav class="tab-nav">
      <button 
        :class="{ active: activeTab === 'ethernet' }"
        @click="activeTab = 'ethernet'"
      >
        以太网帧结构解析器
      </button>
      <button 
        :class="{ active: activeTab === 'csma' }"
        @click="activeTab = 'csma'"
      >
        CSMA/CD 协议模拟器
      </button>
    </nav>

    <main class="main-content">
      <EthernetFrameAnalyzer v-if="activeTab === 'ethernet'" />
      <CSMASimulator v-if="activeTab === 'csma'" />
    </main>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  background: var(--gradient-bg);
  font-family: inherit;
}

.app-header {
  text-align: center;
  padding: 3rem 2rem;
  background: linear-gradient(135deg, 
    rgba(99, 102, 241, 0.1) 0%, 
    rgba(6, 182, 212, 0.1) 50%, 
    rgba(245, 158, 11, 0.1) 100%);
  backdrop-filter: blur(20px);
  border-bottom: 1px solid var(--border-color);
  position: relative;
  overflow: hidden;
}

.app-header::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: url('data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100"><defs><pattern id="grid" width="10" height="10" patternUnits="userSpaceOnUse"><path d="M 10 0 L 0 0 0 10" fill="none" stroke="%23ffffff" stroke-width="0.5" opacity="0.1"/></pattern></defs><rect width="100" height="100" fill="url(%23grid)"/></svg>');
  pointer-events: none;
}

.app-header > * {
  position: relative;
  z-index: 1;
}

.app-header h1 {
  margin: 0 0 1rem 0;
  font-size: clamp(2rem, 5vw, 3.5rem);
  font-weight: 700;
  background: linear-gradient(135deg, #ffffff 0%, #cbd5e1 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  text-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
  animation: fadeIn 1s ease-out;
}

.app-header p {
  margin: 0;
  color: var(--text-secondary);
  font-size: clamp(1rem, 2vw, 1.3rem);
  font-weight: 300;
  letter-spacing: 0.5px;
  animation: fadeIn 1s ease-out 0.2s both;
}

.tab-nav {
  display: flex;
  justify-content: center;
  gap: 1.5rem;
  padding: 2rem;
  background: rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid var(--border-color);
}

.tab-nav button {
  position: relative;
  padding: 1rem 2rem;
  border: 2px solid transparent;
  background: var(--bg-glass);
  color: var(--text-primary);
  border-radius: var(--radius-xl);
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  font-size: 1.1rem;
  font-weight: 500;
  backdrop-filter: blur(10px);
  overflow: hidden;
  min-width: 200px;
  box-shadow: var(--shadow-md);
}

.tab-nav button::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.1), transparent);
  transition: left 0.6s ease;
}

.tab-nav button:hover::before {
  left: 100%;
}

.tab-nav button:hover {
  background: var(--bg-glass-hover);
  border-color: var(--primary-light);
  transform: translateY(-3px) scale(1.02);
  box-shadow: var(--shadow-xl);
}

.tab-nav button.active {
  background: var(--gradient-primary);
  color: white;
  border-color: transparent;
  box-shadow: 0 8px 32px rgba(99, 102, 241, 0.3);
  transform: translateY(-2px);
}

.tab-nav button.active::after {
  content: '';
  position: absolute;
  bottom: -2px;
  left: 50%;
  transform: translateX(-50%);
  width: 80%;
  height: 3px;
  background: white;
  border-radius: 2px;
}

.main-content {
  padding: 2rem;
  max-width: 1400px;
  margin: 0 auto;
  animation: fadeIn 0.8s ease-out 0.4s both;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .app-header {
    padding: 2rem 1rem;
  }
  
  .tab-nav {
    flex-direction: column;
    align-items: center;
    gap: 1rem;
    padding: 1.5rem 1rem;
  }
  
  .tab-nav button {
    width: 100%;
    max-width: 300px;
    min-width: auto;
  }
  
  .main-content {
    padding: 1rem;
  }
}

@media (max-width: 480px) {
  .app-header h1 {
    font-size: 1.8rem;
  }
  
  .app-header p {
    font-size: 0.9rem;
  }
  
  .tab-nav button {
    padding: 0.8rem 1.5rem;
    font-size: 1rem;
  }
}
</style>
