<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted } from 'vue';
import { search, type SearchParams } from '@/api';
import type { SearchResponse, MergedResults } from '@/types';
import SearchForm from '@/components/SearchForm.vue';
import ResultTabs from '@/components/ResultTabs.vue';
import SearchStats from '@/components/SearchStats.vue';
import ApiStatus from '@/components/ApiStatus.vue';
import ApiDocs from '@/components/ApiDocs.vue';

// 搜索状态
const loading = ref(false);
const searchResults = reactive<{
  total: number;
  mergedResults: MergedResults;
}>({
  total: 0,
  mergedResults: {}
});

// 搜索时间
const searchTime = ref<number | undefined>(undefined);

// 后台更新状态
const isUpdating = ref(false);
const updateCount = ref(0);
const updateTimer = ref<number | null>(null);
const secondSearchTimeout = ref<number | null>(null);
const thirdSearchTimeout = ref<number | null>(null);
const lastSearchParams = ref<SearchParams | null>(null);

// 是否已经执行过搜索
const hasSearched = ref(false);
// 是否正在进行后台搜索（包括初始搜索和后续更新）
const isActivelySearching = ref(false);

// 当前页面状态
const currentPage = ref<'search' | 'status' | 'docs'>('search');

// 页面切换
const switchToStatus = () => {
  currentPage.value = 'status';
};

const switchToDocs = () => {
  currentPage.value = 'docs';
};

const switchToHome = () => {
  window.open('https://www.liushen.fun/', '_blank');
};



// 处理搜索
const handleSearch = async (params: SearchParams) => {
  // 停止之前的更新
  stopUpdate();
  
  // 标记已执行搜索和正在搜索
  hasSearched.value = true;
  isActivelySearching.value = true;
  
  // 重置状态
  loading.value = true;
  
  // 清空之前的搜索结果
  searchResults.total = 0;
  searchResults.mergedResults = {};
  searchTime.value = undefined;
  
  // 保存搜索参数
  lastSearchParams.value = { ...params };
  
  const startTime = Date.now();
  
  try {
    // 创建TG源搜索参数
    const tgParams: SearchParams = {
      ...params,
      src: 'tg'
    };
    
    // 创建ALL源搜索参数
    const allParams: SearchParams = {
      ...params,
      src: 'all'
    };
    
    // 先发起TG源搜索请求
    search(tgParams)
      .then(tgResponse => {
        
        if (tgResponse && tgResponse.total !== undefined) {
          // 使用TG的搜索结果进行显示
          updateSearchResults(tgResponse);
          searchTime.value = Date.now() - startTime;
          // TG搜索完成后，关闭加载状态
          loading.value = false;
          
          // TG搜索完成后，再发起第一次ALL源搜索
          search(allParams)
            .then(allResponse => {
              
              // 记录第一次ALL搜索完成时间
              const firstAllSearchCompleteTime = Date.now();
              
              // 如果ALL源结果比当前结果更多，则更新显示
              if (allResponse && allResponse.total >= searchResults.total) {
                updateSearchResults(allResponse);
              }
              
              // 开始第二次ALL源搜索
              startSecondAllSearch(firstAllSearchCompleteTime);
            })
            .catch(error => {
              console.error('第一次ALL搜索出错:', error);
              
              // 即使第一次ALL搜索失败，也继续进行第二次搜索
              startSecondAllSearch(Date.now());
            });
        } else {
          console.error('TG搜索结果格式不正确:', tgResponse);
          loading.value = false;
          
          // 即使TG搜索失败，也尝试ALL源搜索
          search(allParams)
            .then(allResponse => {
              
              if (allResponse && allResponse.total !== undefined) {
                updateSearchResults(allResponse);
                const firstAllSearchCompleteTime = Date.now();
                startSecondAllSearch(firstAllSearchCompleteTime);
              }
            })
            .catch(error => {
              console.error('第一次ALL搜索出错:', error);
              isActivelySearching.value = false;
            });
        }
      })
      .catch(error => {
        console.error('TG搜索出错:', error);
        loading.value = false;
        
        // TG搜索出错时，尝试ALL源搜索
        search(allParams)
          .then(allResponse => {
            
            if (allResponse && allResponse.total !== undefined) {
              updateSearchResults(allResponse);
              const firstAllSearchCompleteTime = Date.now();
              startSecondAllSearch(firstAllSearchCompleteTime);
            }
          })
          .catch(error => {
            console.error('第一次ALL搜索出错:', error);
            isActivelySearching.value = false;
          });
      });
    
    // 设置一个超时，确保即使搜索很慢，UI也不会一直处于加载状态
    setTimeout(() => {
      if (loading.value) {
        loading.value = false;
      }
    }, 5000); // 5秒后如果还在加载，则关闭加载状态
    
  } catch (error) {
    console.error('搜索初始化出错:', error);
    loading.value = false;
    isActivelySearching.value = false;
  }
};

// 搜索完成处理
const handleSearchComplete = () => {
  // 只处理UI相关的状态，不影响搜索流程
};

// 更新搜索结果
const updateSearchResults = (response: SearchResponse) => {
  if (!response) return;
  
  searchResults.total = response.total || 0;
  
  if (response.merged_by_type) {
    searchResults.mergedResults = { ...response.merged_by_type };
  } else {
    console.warn('搜索结果中没有merged_by_type字段');
    searchResults.mergedResults = {};
  }
};

// 开始第二次ALL源搜索
const startSecondAllSearch = (firstAllSearchCompleteTime: number) => {
  if (!lastSearchParams.value) return;
  
  isUpdating.value = true;
  isActivelySearching.value = true;
  updateCount.value = 1;
  
  // 创建ALL源搜索参数
  const allParams: SearchParams = {
    ...lastSearchParams.value,
    src: 'all'
  };
  
  // 计算需要等待的时间，确保与第一次ALL搜索至少间隔2秒
  const currentTime = Date.now();
  const timeElapsedSinceFirstAllSearch = currentTime - firstAllSearchCompleteTime;
  const delayForSecondSearch = Math.max(0, 2000 - timeElapsedSinceFirstAllSearch);
  
  // 执行第二次ALL搜索
  const executeSecondAllSearch = async () => {
    if (!lastSearchParams.value) {
      stopUpdate();
      return;
    }
    
    try {
      const secondAllSearchStartTime = Date.now();
      const response = await search(allParams);
      
      // 更新结果
      if (response && response.total >= searchResults.total) {
        updateSearchResults(response);
      }
      
      // 记录第二次ALL搜索完成时间
      const secondAllSearchCompleteTime = Date.now();
      
      // 开始第三次ALL源搜索
      startThirdAllSearch(secondAllSearchCompleteTime);
    } catch (error) {
      console.error('第二次ALL搜索出错:', error);
      stopUpdate();
    }
  };
  
  // 设置定时器，在适当的时间执行第二次ALL搜索
  secondSearchTimeout.value = window.setTimeout(executeSecondAllSearch, delayForSecondSearch);
};

// 开始第三次ALL源搜索
const startThirdAllSearch = (secondAllSearchCompleteTime: number) => {
  if (!lastSearchParams.value) return;
  
  updateCount.value = 2;
  
  // 创建ALL源搜索参数
  const allParams: SearchParams = {
    ...lastSearchParams.value,
    src: 'all'
  };
  
  // 计算需要等待的时间，确保与第二次ALL搜索至少间隔3秒
  const currentTime = Date.now();
  const timeElapsedSinceSecondAllSearch = currentTime - secondAllSearchCompleteTime;
  const delayForThirdSearch = Math.max(0, 3000 - timeElapsedSinceSecondAllSearch);
  
  // 执行第三次ALL搜索
  const executeThirdAllSearch = async () => {
    if (!lastSearchParams.value) {
      stopUpdate();
      return;
    }
    
    try {
      const response = await search(allParams);
      
      // 更新结果
      if (response && response.total >= searchResults.total) {
        updateSearchResults(response);
      }
    } catch (error) {
      console.error('第三次ALL搜索出错:', error);
    } finally {
      // 完成所有搜索，停止更新
      stopUpdate();
    }
  };
  
  // 设置定时器，在适当的时间执行第三次ALL搜索
  thirdSearchTimeout.value = window.setTimeout(executeThirdAllSearch, delayForThirdSearch);
};

// 停止后台更新
const stopUpdate = () => {
  // 清除所有定时器
  if (updateTimer.value) {
    clearInterval(updateTimer.value);
    updateTimer.value = null;
  }
  
  if (secondSearchTimeout.value) {
    clearTimeout(secondSearchTimeout.value);
    secondSearchTimeout.value = null;
  }
  
  if (thirdSearchTimeout.value) {
    clearTimeout(thirdSearchTimeout.value);
    thirdSearchTimeout.value = null;
  }
  
  // 标记搜索已结束
  isUpdating.value = false;
  isActivelySearching.value = false;
};

// 重置到初始页面
const resetToInitial = () => {
  // 停止之前的更新
  stopUpdate();
  
  // 切换到搜索页面
  currentPage.value = 'search';
  
  // 重置所有状态
  hasSearched.value = false;
  isActivelySearching.value = false;
  loading.value = false;
  searchResults.total = 0;
  searchResults.mergedResults = {};
  searchTime.value = undefined;
  isUpdating.value = false;
  updateCount.value = 0;
};

// 组件卸载时清除定时器
onMounted(() => {
  // App组件已挂载
});

onUnmounted(() => {
  // 确保在组件卸载时清理所有定时器
  stopUpdate();
});
</script>

<template>
  <div class="min-h-screen bg-background text-foreground transition-colors duration-300 flex flex-col">
    <!-- 背景装饰 -->
    <div class="bg-decorative"></div>
    
    <!-- 导航栏 -->
    <nav class="nav-header backdrop-blur-md bg-background/80 border-b border-border">
      <div class="container mx-auto px-4 h-16 flex items-center justify-between">
        <div class="flex items-center gap-3 cursor-pointer" @click="resetToInitial">
          <div class="w-8 h-8 rounded-lg flex items-center justify-center">
            <svg t="1755163867101" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="6270"><path d="M351.1808 59.2896A435.2 435.2 0 0 1 805.376 715.264 460.8 460.8 0 0 1 351.1808 59.3408z" fill="#20C997" p-id="6271"></path><path d="M754.3808 722.2272a358.4 358.4 0 1 0-267.8272 120.2176 51.2 51.2 0 0 1 0 102.4 460.8 460.8 0 1 1 365.1584-179.712l118.8864 121.2416c23.7568 24.2176 23.552 63.0272-0.4096 87.04l-0.4096 0.4096a61.184 61.184 0 0 1-86.9888-0.4608l-148.0192-150.9376a61.7984 61.7984 0 0 1 0.4096-86.9888l0.4096-0.4096c5.632-5.5808 11.9808-9.8304 18.7904-12.8z m-467.968-364.5952h409.6a51.2 51.2 0 1 1 0 102.4h-409.6a51.2 51.2 0 1 1 0-102.4z m0 204.8h256a51.2 51.2 0 0 1 0 102.4h-256a51.2 51.2 0 1 1 0-102.4z" fill="#2C6DD2" p-id="6272"></path></svg>
          </div>
          <div>
            <h1 class="text-xl font-bold">清羽盘搜</h1>
          </div>
        </div>
        
        <!-- 导航菜单 -->
        <nav class="hidden md:flex items-center gap-2">
          <button 
            @click="switchToStatus"
            class="nav-button"
          >
            <span class="nav-icon">📊</span>
            状态面板
          </button>
          <button 
            @click="switchToHome"
            class="nav-button"
          >
            <span class="nav-icon">👨‍💻</span>
            站长主页
          </button>
        </nav>
      </div>
    </nav>
    
    <!-- 主要内容区域 -->
    <main class="container mx-auto px-4 py-8 flex-1">
      <!-- 搜索页面 -->
      <div v-if="currentPage === 'search'" class="search-page">
        <!-- 搜索表单 -->
        <div class="mb-6">
          <SearchForm 
            @search="handleSearch" 
            @search-complete="handleSearchComplete"
          />
        </div>
        
        <!-- 搜索统计 -->
        <div v-if="hasSearched || loading" class="mb-6">
          <SearchStats 
            :total="searchResults.total || 0" 
            :mergedResults="searchResults.mergedResults || {}" 
            :loading="loading"
            :searchTime="searchTime"
            :isUpdating="isUpdating"
            :updateCount="updateCount"
          />
        </div>
        
        <!-- 加载状态 -->
        <div v-if="loading" class="card p-6">
          <div class="space-y-3">
            <div class="h-4 bg-muted rounded animate-pulse"></div>
            <div class="h-4 bg-muted rounded animate-pulse w-3/4"></div>
            <div class="h-4 bg-muted rounded animate-pulse w-1/2"></div>
            <div class="h-4 bg-muted rounded animate-pulse w-2/3"></div>
            <div class="h-4 bg-muted rounded animate-pulse"></div>
          </div>
        </div>
        
        <!-- 搜索结果 -->
        <div v-else>
          <ResultTabs 
            :mergedResults="searchResults.mergedResults || {}" 
            :loading="loading"
            :hasSearched="hasSearched"
            :isActivelySearching="isActivelySearching"
          />
        </div>
      </div>
      
      <!-- 状态页面 -->
      <div v-else-if="currentPage === 'status'" class="status-page">
        <ApiStatus />
      </div>
      
      <!-- API文档页面 -->
      <div v-else-if="currentPage === 'docs'" class="docs-page">
        <ApiDocs />
      </div>
    </main>
    
    <!-- 页脚 -->
    <footer class="border-t border-border bg-background/50 backdrop-blur-sm mt-auto">
      <div class="container mx-auto px-4 py-3 md:py-6">
        <!-- 第一行链接 - 在移动端隐藏 -->
        <div class="hidden md:flex flex-row items-center justify-center text-sm text-muted-foreground">
          <span class="flex items-center">
            © 2023-{{ new Date().getFullYear() }}
          </span>
          <div class="mx-3 text-border">|</div>
          <a @click="switchToDocs" class="hover:text-foreground transition-colors cursor-pointer">
            API文档
          </a>
          <div class="mx-3 text-border">|</div>
          <a href="https://github.com/willow-god/pansou-web" 
             target="_blank" 
             rel="noopener noreferrer" 
             class="hover:text-foreground transition-colors">
            项目仓库
          </a>
        </div>

        <!-- 第二行链接 - 在移动端隐藏 -->
        <div class="hidden md:flex flex-row items-center justify-center mt-3 text-sm text-muted-foreground">
          <span>
            Build by 
            <a href="https://www.liushen.fun/" 
               target="_blank" 
               rel="noopener noreferrer" 
               class="hover:text-foreground transition-colors">
              LiuShen
            </a>
          </span>
          <div class="mx-3 text-border">|</div>
          <span>
            Powered by 
            <a href="https://github.com/fish2018/pansou" 
               target="_blank" 
               rel="noopener noreferrer" 
               class="hover:text-foreground transition-colors">
              PanSou
            </a>
          </span>
          <div class="mx-3 text-border">|</div>
          <span>
            License 
            <a href="https://github.com/fish2018/pansou-web/blob/main/LICENSE" 
               target="_blank" 
               rel="noopener noreferrer" 
               class="hover:text-foreground transition-colors">
              MIT
            </a>
          </span>
        </div>

        <!-- 备案信息 - 始终显示 -->
        <div class="flex flex-col md:flex-row items-center justify-center space-y-1 md:space-y-0 mt-2 md:mt-3 text-sm text-muted-foreground">
          <a href="https://beian.miit.gov.cn/" 
             target="_blank" 
             rel="noopener noreferrer" 
             class="hover:text-foreground transition-colors">
            陕ICP备2024028531号
          </a>
          <div class="hidden md:block mx-3 text-border">|</div>
          <a href="http://www.beian.gov.cn/portal/registerSystemInfo?recordcode=61011602000637" 
             target="_blank" 
             rel="noopener noreferrer" 
             class="hover:text-foreground transition-colors">
            陕公网安备61011602000637号
          </a>
        </div>
      </div>
    </footer>
  </div>
</template>

<style scoped>
.bg-decorative {
  position: fixed;
  inset: 0;
  z-index: -10;
  background-image: radial-gradient(circle at 1px 1px, hsl(var(--muted-foreground)) 1px, transparent 0);
  background-size: 20px 20px;
  opacity: 0.1;
}

.nav-header {
  position: sticky;
  top: 0;
  z-index: 50;
}

/* 导航按钮样式 */
.nav-button {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  background: transparent;
  color: hsl(var(--muted-foreground));
  border: 1px solid hsl(var(--border));
  border-radius: 0.375rem;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}

.nav-button:hover {
  background: hsl(var(--accent));
  color: hsl(var(--accent-foreground));
  border-color: hsl(var(--accent));
}



.nav-icon {
  font-size: 1rem;
}

/* 页面切换动画 */
.search-page, .status-page {
  animation: fadeIn 0.3s ease-in-out;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@media (max-width: 768px) {
  .container {
    padding-left: 1rem;
    padding-right: 1rem;
  }
  
  .nav-button {
    padding: 0.375rem 0.75rem;
    font-size: 0.8rem;
  }
  
  .nav-icon {
    font-size: 0.875rem;
  }
}

/* 页脚按钮样式 */
footer button {
  background: transparent;
  border: none;
  padding: 0;
  font-size: inherit;
  color: inherit;
  cursor: pointer;
}
</style>
