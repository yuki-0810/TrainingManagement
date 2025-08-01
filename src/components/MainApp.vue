<script setup>
import { ref } from 'vue'
import SupabaseTest from './SupabaseTest.vue'

const appTitle = ref('家族向けトレーニング管理')
const subtitle = ref('家族でトレーニングを継続しよう')
const activeTab = ref('home')

const switchTab = (tab) => {
  activeTab.value = tab
}
</script>

<template>
  <div class="main-app">
    <header class="header">
      <div class="container">
        <h1 class="title">{{ appTitle }}</h1>
        <p class="subtitle">{{ subtitle }}</p>
        
        <!-- タブナビゲーション -->
        <nav class="tab-nav">
          <button 
            @click="switchTab('home')" 
            :class="['tab-btn', { active: activeTab === 'home' }]"
          >
            ホーム
          </button>
          <button 
            @click="switchTab('test')" 
            :class="['tab-btn', { active: activeTab === 'test' }]"
          >
            Supabaseテスト
          </button>
        </nav>
      </div>
    </header>
    
    <main class="main-content">
      <div class="container">
        <!-- ホームタブ -->
        <div v-if="activeTab === 'home'" class="tab-content">
          <div class="welcome-section">
            <h2>トレーニング管理を始めましょう</h2>
            <p>家族でトレーニングを記録・共有して、みんなで健康を維持しましょう。</p>
            
            <div class="feature-grid">
              <div class="feature-card">
                <h3>📅 今日のメニュー</h3>
                <p>個人に最適化されたトレーニングメニューを確認</p>
              </div>
              <div class="feature-card">
                <h3>📊 記録管理</h3>
                <p>トレーニング実施記録を簡単に入力・管理</p>
              </div>
              <div class="feature-card">
                <h3>👥 家族共有</h3>
                <p>家族間でトレーニング記録を共有してモチベーション向上</p>
              </div>
              <div class="feature-card">
                <h3>🔔 通知機能</h3>
                <p>サボり防止のためのLINE通知連携</p>
              </div>
            </div>
            
            <div class="action-buttons">
              <button class="btn btn-primary">トレーニング開始</button>
              <button class="btn btn-secondary">記録を見る</button>
            </div>
            
            <div class="test-note">
              <p>💡 <strong>開発中:</strong> 上部の「Supabaseテスト」タブから接続・認証・DB操作をテストできます</p>
            </div>
          </div>
        </div>
        
        <!-- Supabaseテストタブ -->
        <div v-if="activeTab === 'test'" class="tab-content">
          <SupabaseTest />
        </div>
      </div>
    </main>
  </div>
</template>

<style scoped>
.main-app {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.header {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  box-shadow: 0 2px 20px rgba(0, 0, 0, 0.1);
  padding: 1.5rem 0;
}

.title {
  font-size: 2.5rem;
  font-weight: bold;
  color: #1a202c;
  text-align: center;
  margin-bottom: 0.5rem;
}

.subtitle {
  font-size: 1.25rem;
  color: #4a5568;
  text-align: center;
  margin-bottom: 2rem;
}

.tab-nav {
  display: flex;
  justify-content: center;
  gap: 1rem;
}

.tab-btn {
  padding: 0.75rem 1.5rem;
  border: 2px solid #e2e8f0;
  background: white;
  color: #4a5568;
  font-size: 1rem;
  font-weight: 500;
  border-radius: 25px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.tab-btn:hover {
  border-color: #3b82f6;
  color: #3b82f6;
}

.tab-btn.active {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

.main-content {
  padding: 3rem 0;
}

.tab-content {
  background: white;
  border-radius: 1rem;
  padding: 2rem;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.1);
}

.welcome-section {
  text-align: center;
}

.welcome-section h2 {
  font-size: 2rem;
  color: #1a202c;
  margin-bottom: 1rem;
}

.welcome-section > p {
  font-size: 1.125rem;
  color: #4a5568;
  margin-bottom: 3rem;
}

.feature-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1.5rem;
  margin-bottom: 3rem;
}

.feature-card {
  background: #f8fafc;
  padding: 1.5rem;
  border-radius: 0.75rem;
  border: 1px solid #e2e8f0;
  transition: transform 0.2s ease;
}

.feature-card:hover {
  transform: translateY(-2px);
}

.feature-card h3 {
  font-size: 1.25rem;
  color: #1a202c;
  margin-bottom: 0.5rem;
}

.feature-card p {
  color: #4a5568;
  line-height: 1.6;
}

.action-buttons {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 2rem;
}

.action-buttons .btn {
  padding: 0.75rem 2rem;
  font-size: 1.125rem;
  font-weight: 500;
}

.test-note {
  background: #fef3c7;
  padding: 1rem;
  border-radius: 0.5rem;
  border-left: 4px solid #f59e0b;
}

.test-note p {
  margin: 0;
  color: #92400e;
}

@media (max-width: 768px) {
  .title {
    font-size: 2rem;
  }
  
  .subtitle {
    font-size: 1rem;
  }
  
  .tab-nav {
    flex-direction: column;
    align-items: center;
  }
  
  .tab-btn {
    width: 200px;
  }
  
  .feature-grid {
    grid-template-columns: 1fr;
  }
  
  .action-buttons {
    flex-direction: column;
    align-items: center;
  }
  
  .action-buttons .btn {
    width: 200px;
  }
}

@media (max-width: 480px) {
  .main-content {
    padding: 1.5rem 0;
  }
  
  .tab-content {
    padding: 1.5rem;
  }
  
  .title {
    font-size: 1.75rem;
  }
  
  .welcome-section h2 {
    font-size: 1.5rem;
  }
}
</style>