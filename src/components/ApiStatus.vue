<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { getHealth, type HealthStatus } from '@/api';

// 状态数据
const healthData = ref<HealthStatus | null>(null);
const loading = ref(true);
const error = ref<string | null>(null);

// 复制状态
const copySuccess = ref(false);
const copyTimeout = ref<number | null>(null);

// 环境变量折叠状态
const envExpanded = ref(false);

// 获取健康状态
const fetchHealth = async () => {
  try {
    loading.value = true;
    error.value = null;
    healthData.value = await getHealth();
  } catch (err) {
    error.value = '获取状态失败';
    console.error('获取健康状态失败:', err);
  } finally {
    loading.value = false;
  }
};

// 获取当前时间
const getCurrentTime = () => {
  return new Date().toLocaleString('zh-CN', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit'
  });
};

// 生成环境变量字符串
const generateEnvString = () => {
  if (!healthData.value) return '';
  
  const channels = healthData.value.channels.join(',');
  return `export CHANNELS=${channels}`;
};

// 复制环境变量
const copyEnvVariable = async () => {
  try {
    const envString = generateEnvString();
    await navigator.clipboard.writeText(envString);
    
    copySuccess.value = true;
    
    // 清除之前的定时器
    if (copyTimeout.value) {
      clearTimeout(copyTimeout.value);
    }
    
    // 3秒后重置状态
    copyTimeout.value = window.setTimeout(() => {
      copySuccess.value = false;
    }, 3000);
  } catch (err) {
    console.error('复制失败:', err);
  }
};



// 组件挂载时获取数据
onMounted(() => {
  fetchHealth();
});
</script>

<template>
  <div class="api-status-container">
    <!-- 头部标题 -->
    <div class="status-header">
      <h1 class="status-title">
        <span class="title-icon">📊</span>
        清羽盘搜 API 状态监控
      </h1>
    </div>

    <!-- 加载状态 -->
    <div v-if="loading" class="loading-state">
      <div class="loading-spinner"></div>
      <p>获取状态信息中...</p>
    </div>

    <!-- 错误状态 -->
    <div v-else-if="error" class="error-state">
      <div class="error-icon">❌</div>
      <h3>获取状态失败</h3>
      <p>{{ error }}</p>
      <button @click="fetchHealth" class="retry-btn">重试</button>
    </div>

    <!-- 状态信息 -->
    <div v-else-if="healthData" class="status-content">
      <!-- 系统状态卡片 -->
      <div class="system-status-card">
        <div class="card-header">
          <h2 class="card-title">
            <span class="status-indicator" :class="{ 'healthy': healthData.status === 'ok' }"></span>
            系统状态
            <div class="status-badge" :class="{ 'healthy': healthData.status === 'ok' }">
              {{ healthData.status === 'ok' ? '正常' : '异常' }}
            </div>
          </h2>
        </div>
        
        <div class="card-content">
          <div class="status-info">
            <div class="status-item">
              <span class="status-label">状态:</span>
              <span class="status-value">{{ healthData.status }}</span>
            </div>
            <div class="status-item">
              <span class="status-label">插件:</span>
              <span class="status-value">{{ healthData.plugins_enabled ? '已启用' : '已禁用' }}</span>
            </div>
            <div class="status-item">
              <span class="status-label">插件数量:</span>
              <span class="status-value">{{ healthData.plugin_count }}</span>
            </div>
            <div class="status-item">
              <span class="status-label">频道数量:</span>
              <span class="status-value">{{ healthData.channels.length }}</span>
            </div>
          </div>
          
          <!-- 环境变量折叠区域 -->
          <div class="env-section">
            <button 
              @click="envExpanded = !envExpanded"
              class="env-toggle"
            >
              <span class="toggle-icon" :class="{ 'expanded': envExpanded }">▶</span>
              环境变量配置
            </button>
            
            <div v-show="envExpanded" class="env-content">
              <div class="env-variable">
                <code>{{ generateEnvString() }}</code>
              </div>
            </div>
          </div>
          
          <div class="card-actions">
            <button 
              @click="copyEnvVariable" 
              class="copy-btn"
              :class="{ 'success': copySuccess }"
            >
              <span v-if="copySuccess">✅ 已复制</span>
              <span v-else>📋 复制环境变量</span>
            </button>
          </div>
        </div>
      </div>

      <!-- TG频道卡片区 -->
      <div class="channels-section">
        <h2 class="section-title">
          <span class="channel-icon">📡</span>
          TG 频道配置
          <span class="count-badge">{{ healthData.channels.length }}</span>
        </h2>
        
        <div class="channels-grid">
          <div 
            v-for="channel in healthData.channels" 
            :key="channel"
            class="channel-card"
          >
            <div class="channel-name">{{ channel }}</div>
            <div class="channel-status">已启用</div>
          </div>
        </div>
      </div>

      <!-- 插件卡片区 -->
      <div class="plugins-section">
        <h2 class="section-title">
          <span class="plugin-icon">🧩</span>
          已加载插件
          <span class="count-badge">{{ healthData.plugin_count }}</span>
        </h2>
        
        <div class="plugins-grid">
          <div 
            v-for="plugin in healthData.plugins" 
            :key="plugin"
            class="plugin-card"
          >
            <div class="plugin-name">{{ plugin }}</div>
            <div class="plugin-status">已加载</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.api-status-container {
  max-width: 1400px;
  margin: 0 auto;
  padding: 2rem;
  background: #f8fafc;
  min-height: 100vh;
}

/* 头部样式 */
.status-header {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 2rem;
  padding-bottom: 1rem;
  border-bottom: 2px solid #e2e8f0;
}

.status-title {
  font-size: 2rem;
  font-weight: 700;
  color: #1a202c;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.title-icon {
  font-size: 2.5rem;
}

/* 加载和错误状态 */
.loading-state, .error-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 4rem 2rem;
  text-align: center;
}

.loading-spinner {
  width: 3rem;
  height: 3rem;
  border: 4px solid #e2e8f0;
  border-top: 4px solid #3b82f6;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 1rem;
}

.error-icon {
  font-size: 4rem;
  margin-bottom: 1rem;
}

.retry-btn {
  margin-top: 1rem;
  padding: 0.75rem 1.5rem;
  background: #ef4444;
  color: white;
  border: none;
  border-radius: 0.5rem;
  cursor: pointer;
  font-weight: 500;
}

/* 状态内容 */
.status-content {
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

/* 系统状态卡片 */
.system-status-card {
  background: white;
  border-radius: 1rem;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.07);
  overflow: hidden;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
  max-width: 100%;
}

.system-status-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.1);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.25rem 1.5rem;
  border-bottom: 1px solid #e2e8f0;
  background: #f8fafc;
}

.card-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #374151;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.card-content {
  padding: 1.25rem;
}

/* 状态指示器 */
.status-indicator {
  width: 0.75rem;
  height: 0.75rem;
  border-radius: 50%;
  margin-right: 0.5rem;
  background: #ef4444;
  box-shadow: 0 0 0 2px rgba(239, 68, 68, 0.3);
}

.status-indicator.healthy {
  background: #10b981;
  box-shadow: 0 0 0 2px rgba(16, 185, 129, 0.3);
}

.status-badge {
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.875rem;
  font-weight: 500;
  background: #fee2e2;
  color: #b91c1c;
}

.status-badge.healthy {
  background: #d1fae5;
  color: #065f46;
}

/* 状态信息布局 */
.status-info {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.status-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem;
  background: #f9fafb;
  border-radius: 0.5rem;
}

.status-label {
  font-size: 0.875rem;
  color: #6b7280;
  font-weight: 500;
}

.status-value {
  font-weight: 600;
  color: #374151;
}

/* 计数徽章 */
.count-badge {
  background: #3b82f6;
  color: white;
  font-size: 0.75rem;
  padding: 0.25rem 0.5rem;
  border-radius: 9999px;
  font-weight: 600;
  margin-left: 0.5rem;
}

/* 环境变量折叠区域 */
.env-section {
  margin-bottom: 1.5rem;
  border: 1px solid #e2e8f0;
  border-radius: 0.5rem;
  overflow: hidden;
}

.env-toggle {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 1rem;
  background: #f8fafc;
  border: none;
  font-size: 0.875rem;
  font-weight: 500;
  color: #374151;
  cursor: pointer;
  transition: all 0.2s ease;
}

.env-toggle:hover {
  background: #f1f5f9;
}

.toggle-icon {
  font-size: 0.75rem;
  transition: transform 0.2s ease;
}

.toggle-icon.expanded {
  transform: rotate(90deg);
}

.env-content {
  padding: 1rem;
  background: white;
  border-top: 1px solid #e2e8f0;
}

/* 卡片底部操作区域 */
.card-actions {
  display: flex;
  justify-content: center;
  padding-top: 1rem;
}

/* 复制按钮 */
.copy-btn {
  padding: 0.5rem 1.5rem;
  background: #10b981;
  color: white;
  border: none;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}

.copy-btn:hover {
  background: #059669;
}

.copy-btn.success {
  background: #16a34a;
}

/* 环境变量 */
.env-variable {
  padding: 0.75rem;
  background: #f1f5f9;
  border-radius: 0.375rem;
  border: 1px solid #cbd5e1;
}

.env-variable code {
  font-family: 'Courier New', monospace;
  font-size: 0.8rem;
  color: #475569;
  word-break: break-all;
  line-height: 1.4;
}

/* 频道和插件区域 */
.channels-section, .plugins-section {
  background: white;
  border-radius: 1rem;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.07);
  padding: 1.5rem;
}

.section-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #374151;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #e2e8f0;
}

/* 频道网格 */
.channels-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
}

.channel-card {
  background: #eff6ff;
  border-radius: 0.75rem;
  padding: 1rem;
  border: 1px solid #bfdbfe;
  transition: all 0.2s ease;
}

.channel-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 6px rgba(59, 130, 246, 0.2);
}

.channel-name {
  font-weight: 600;
  color: #1e40af;
  margin-bottom: 0.5rem;
  word-break: break-all;
}

.channel-status {
  font-size: 0.75rem;
  color: #3b82f6;
  background: rgba(59, 130, 246, 0.1);
  padding: 0.25rem 0.5rem;
  border-radius: 9999px;
  display: inline-block;
}

/* 插件网格 */
.plugins-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
}

.plugin-card {
  background: #f0fdf4;
  border-radius: 0.75rem;
  padding: 1rem;
  border: 1px solid #bbf7d0;
  transition: all 0.2s ease;
}

.plugin-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 6px rgba(16, 185, 129, 0.2);
}

.plugin-name {
  font-weight: 600;
  color: #15803d;
  margin-bottom: 0.5rem;
  word-break: break-all;
}

.plugin-status {
  font-size: 0.75rem;
  color: #10b981;
  background: rgba(16, 185, 129, 0.1);
  padding: 0.25rem 0.5rem;
  border-radius: 9999px;
  display: inline-block;
}

/* 响应式设计 */
@media (max-width: 1024px) {
  .channels-grid, .plugins-grid {
    grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
  }
}

@media (max-width: 768px) {
  .api-status-container {
    padding: 1rem;
  }
  
  .status-header {
    flex-direction: column;
    gap: 1rem;
    align-items: stretch;
  }
  
  .status-title {
    font-size: 1.5rem;
    justify-content: center;
  }
  
  .status-info {
    grid-template-columns: 1fr;
  }
  
  .channels-grid, .plugins-grid {
    grid-template-columns: repeat(auto-fill, minmax(140px, 1fr));
  }
}

@media (max-width: 480px) {
  .channels-grid, .plugins-grid {
    grid-template-columns: 1fr;
  }
  
  .section-title {
    font-size: 1.1rem;
  }
  
  .status-title {
    font-size: 1.3rem;
  }
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
</style>