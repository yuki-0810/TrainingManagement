<script setup>
import { ref, onMounted, computed } from 'vue'
import { supabase } from '../supabase.js'

const user = ref(null)
const userProfile = ref(null)
const trainingMenus = ref([])
const trainingRecords = ref([])
const isRecordingTraining = ref(false)
const selectedMenu = ref(null)
const message = ref('')
const isLoading = ref(false)

// 記録フォーム
const recordForm = ref({
  training_menu_id: '',
  completed_at: '',
  duration_minutes: 0,
  notes: '',
  satisfaction_rating: 3
})

// フィルター
const filterPeriod = ref('week') // 'today', 'week', 'month', 'all'
const filterMember = ref('all') // 'all' or user_id

const satisfactionLabels = {
  1: '😞 とても不満',
  2: '😕 不満',
  3: '😐 普通',
  4: '😊 満足',
  5: '🤩 とても満足'
}

const filteredRecords = computed(() => {
  let filtered = trainingRecords.value

  // 期間フィルター
  const now = new Date()
  if (filterPeriod.value === 'today') {
    const today = new Date(now.getFullYear(), now.getMonth(), now.getDate())
    filtered = filtered.filter(record => new Date(record.completed_at) >= today)
  } else if (filterPeriod.value === 'week') {
    const weekAgo = new Date(now.getTime() - 7 * 24 * 60 * 60 * 1000)
    filtered = filtered.filter(record => new Date(record.completed_at) >= weekAgo)
  } else if (filterPeriod.value === 'month') {
    const monthAgo = new Date(now.getTime() - 30 * 24 * 60 * 60 * 1000)
    filtered = filtered.filter(record => new Date(record.completed_at) >= monthAgo)
  }

  // メンバーフィルター
  if (filterMember.value !== 'all') {
    filtered = filtered.filter(record => record.user_id === filterMember.value)
  }

  return filtered.sort((a, b) => new Date(b.completed_at) - new Date(a.completed_at))
})

onMounted(async () => {
  await loadUserData()
  if (userProfile.value?.family_group_id) {
    await Promise.all([loadTrainingMenus(), loadTrainingRecords()])
  }
})

const loadUserData = async () => {
  try {
    const { data: { user: currentUser } } = await supabase.auth.getUser()
    user.value = currentUser
    
    if (currentUser) {
      const { data: profile } = await supabase
        .from('user_profiles')
        .select('*, family_groups(*)')
        .eq('id', currentUser.id)
        .single()
      
      userProfile.value = profile
    }
  } catch (error) {
    console.error('ユーザーデータ読み込みエラー:', error)
  }
}

const loadTrainingMenus = async () => {
  try {
    const { data, error } = await supabase
      .from('training_menus')
      .select('id, name, duration_minutes, difficulty_level')
      .eq('family_group_id', userProfile.value.family_group_id)
      .order('name')
    
    if (error) throw error
    trainingMenus.value = data || []
  } catch (error) {
    console.error('メニュー読み込みエラー:', error)
  }
}

const loadTrainingRecords = async () => {
  try {
    // トレーニング記録を取得
    const { data: records, error: recordsError } = await supabase
      .from('training_records')
      .select(`
        *,
        training_menus(name, difficulty_level)
      `)
      .in('user_id', await getFamilyMemberIds())
      .order('completed_at', { ascending: false })
    
    if (recordsError) throw recordsError

    // ユーザー情報を別途取得
    const userIds = [...new Set(records.map(r => r.user_id))]
    const { data: profiles, error: profilesError } = await supabase
      .from('user_profiles')
      .select('id, display_name')
      .in('id', userIds)
    
    if (profilesError) {
      console.warn('プロファイル取得エラー:', profilesError)
    }

    // 記録にユーザー名を追加
    trainingRecords.value = records.map(record => ({
      ...record,
      user_name: profiles?.find(p => p.id === record.user_id)?.display_name || '不明'
    }))
  } catch (error) {
    message.value = `記録読み込みエラー: ${error.message}`
  }
}

const getFamilyMemberIds = async () => {
  try {
    const { data, error } = await supabase
      .from('user_profiles')
      .select('id')
      .eq('family_group_id', userProfile.value.family_group_id)
    
    if (error) throw error
    return data.map(profile => profile.id)
  } catch (error) {
    console.error('家族メンバーID取得エラー:', error)
    return [user.value.id] // フォールバック
  }
}

const startRecording = (menu = null) => {
  const now = new Date()
  const localISOTime = new Date(now.getTime() - now.getTimezoneOffset() * 60000).toISOString().slice(0, 16)
  
  recordForm.value = {
    training_menu_id: menu?.id || '',
    completed_at: localISOTime,
    duration_minutes: menu?.duration_minutes || 30,
    notes: '',
    satisfaction_rating: 3
  }
  
  selectedMenu.value = menu
  isRecordingTraining.value = true
}

const cancelRecording = () => {
  isRecordingTraining.value = false
  selectedMenu.value = null
  recordForm.value = {
    training_menu_id: '',
    completed_at: '',
    duration_minutes: 0,
    notes: '',
    satisfaction_rating: 3
  }
}

const saveRecord = async () => {
  if (!recordForm.value.training_menu_id) {
    message.value = 'トレーニングメニューを選択してください'
    return
  }
  
  if (!recordForm.value.completed_at) {
    message.value = '実施日時を入力してください'
    return
  }
  
  isLoading.value = true
  try {
    const recordData = {
      user_id: user.value.id,
      training_menu_id: recordForm.value.training_menu_id,
      completed_at: new Date(recordForm.value.completed_at).toISOString(),
      duration_minutes: parseInt(recordForm.value.duration_minutes),
      notes: recordForm.value.notes,
      satisfaction_rating: parseInt(recordForm.value.satisfaction_rating)
    }
    
    const { error } = await supabase
      .from('training_records')
      .insert([recordData])
    
    if (error) throw error
    
    message.value = 'トレーニング記録を保存しました！'
    cancelRecording()
    await loadTrainingRecords()
  } catch (error) {
    message.value = `記録保存エラー: ${error.message}`
  } finally {
    isLoading.value = false
  }
}

const deleteRecord = async (recordId) => {
  if (!confirm('この記録を削除しますか？')) return
  
  isLoading.value = true
  try {
    const { error } = await supabase
      .from('training_records')
      .delete()
      .eq('id', recordId)
    
    if (error) throw error
    
    message.value = '記録を削除しました'
    await loadTrainingRecords()
  } catch (error) {
    message.value = `削除エラー: ${error.message}`
  } finally {
    isLoading.value = false
  }
}

const formatDuration = (minutes) => {
  if (minutes < 60) {
    return `${minutes}分`
  } else {
    const hours = Math.floor(minutes / 60)
    const mins = minutes % 60
    return mins > 0 ? `${hours}時間${mins}分` : `${hours}時間`
  }
}

const formatDateTime = (dateTime) => {
  const date = new Date(dateTime)
  return date.toLocaleString('ja-JP', {
    month: 'short',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  })
}

const getDifficultyColor = (level) => {
  const colors = {
    1: '#10b981', // green
    2: '#f59e0b', // yellow
    3: '#3b82f6', // blue
    4: '#ef4444', // red
    5: '#8b5cf6'  // purple
  }
  return colors[level] || '#6b7280'
}
</script>

<template>
  <div class="training-record">
    <div class="header-section">
      <h3>トレーニング記録</h3>
      <button 
        v-if="userProfile?.family_group_id" 
        @click="startRecording()" 
        class="btn btn-primary"
        :disabled="isLoading"
      >
        記録を追加
      </button>
    </div>
    
    <div v-if="!user" class="no-user">
      <p>ログインが必要です</p>
    </div>
    
    <div v-else-if="!userProfile?.family_group_id" class="no-family">
      <p>家族グループに参加してから記録を管理できます</p>
    </div>
    
    <div v-else class="record-content">
      <!-- 記録追加フォーム -->
      <div v-if="isRecordingTraining" class="record-form">
        <h4>トレーニング記録を追加</h4>
        
        <div class="form-group">
          <label for="menuSelect">トレーニングメニュー *</label>
          <select 
            id="menuSelect" 
            v-model="recordForm.training_menu_id" 
            class="select"
            @change="selectedMenu = trainingMenus.find(m => m.id === recordForm.training_menu_id); recordForm.duration_minutes = selectedMenu?.duration_minutes || 30"
          >
            <option value="">メニューを選択してください</option>
            <option v-for="menu in trainingMenus" :key="menu.id" :value="menu.id">
              {{ menu.name }} ({{ formatDuration(menu.duration_minutes) }})
            </option>
          </select>
        </div>
        
        <div class="form-row">
          <div class="form-group">
            <label for="completedAt">実施日時 *</label>
            <input
              id="completedAt"
              v-model="recordForm.completed_at"
              type="datetime-local"
              class="input"
            />
          </div>
          
          <div class="form-group">
            <label for="duration">実際の時間（分）</label>
            <input
              id="duration"
              v-model="recordForm.duration_minutes"
              type="number"
              min="1"
              max="300"
              class="input"
            />
          </div>
        </div>
        
        <div class="form-group">
          <label for="satisfaction">満足度</label>
          <div class="satisfaction-selector">
            <label 
              v-for="(label, value) in satisfactionLabels" 
              :key="value"
              class="satisfaction-option"
            >
              <input
                type="radio"
                :value="parseInt(value)"
                v-model="recordForm.satisfaction_rating"
                class="satisfaction-radio"
              />
              <span class="satisfaction-label">{{ label }}</span>
            </label>
          </div>
        </div>
        
        <div class="form-group">
          <label for="notes">メモ・感想</label>
          <textarea
            id="notes"
            v-model="recordForm.notes"
            placeholder="今日のトレーニングの感想、気づいたこと、次回への改善点など"
            rows="4"
            class="textarea"
          ></textarea>
        </div>
        
        <div class="form-actions">
          <button @click="saveRecord" :disabled="isLoading" class="btn btn-primary">
            {{ isLoading ? '保存中...' : '記録を保存' }}
          </button>
          <button @click="cancelRecording" class="btn btn-secondary">キャンセル</button>
        </div>
      </div>
      
      <!-- クイック記録（メニューから直接） -->
      <div v-if="!isRecordingTraining && trainingMenus.length > 0" class="quick-record">
        <h4>よく使うメニューから記録</h4>
        <div class="menu-quick-list">
          <button 
            v-for="menu in trainingMenus.slice(0, 4)" 
            :key="menu.id"
            @click="startRecording(menu)"
            class="quick-menu-btn"
          >
            <span class="menu-name">{{ menu.name }}</span>
            <span class="menu-duration">{{ formatDuration(menu.duration_minutes) }}</span>
          </button>
        </div>
      </div>
      
      <!-- フィルター -->
      <div v-if="!isRecordingTraining && trainingRecords.length > 0" class="filter-section">
        <div class="filter-group">
          <label for="periodFilter">期間:</label>
          <select id="periodFilter" v-model="filterPeriod" class="select">
            <option value="today">今日</option>
            <option value="week">過去1週間</option>
            <option value="month">過去1ヶ月</option>
            <option value="all">すべて</option>
          </select>
        </div>
      </div>
      
      <!-- 記録一覧 -->
      <div v-if="!isRecordingTraining" class="records-list">
        <div v-if="filteredRecords.length === 0" class="no-records">
          <p>{{ trainingRecords.length === 0 ? 'まだ記録がありません' : 'フィルター条件に合う記録がありません' }}</p>
        </div>
        
        <div v-else class="records-grid">
          <div v-for="record in filteredRecords" :key="record.id" class="record-card">
            <div class="record-header">
              <div class="record-info">
                <h5>{{ record.training_menus?.name || '削除されたメニュー' }}</h5>
                <div class="record-meta">
                  <span class="user-name">{{ record.user_name }}</span>
                  <span class="record-date">{{ formatDateTime(record.completed_at) }}</span>
                </div>
              </div>
              <div 
                v-if="record.training_menus?.difficulty_level"
                class="difficulty-badge"
                :style="{ backgroundColor: getDifficultyColor(record.training_menus.difficulty_level) }"
              >
                {{ record.training_menus.difficulty_level }}
              </div>
            </div>
            
            <div class="record-details">
              <div class="detail-item">
                <span class="detail-label">実施時間:</span>
                <span class="detail-value">{{ formatDuration(record.duration_minutes) }}</span>
              </div>
              
              <div class="detail-item">
                <span class="detail-label">満足度:</span>
                <span class="detail-value">{{ satisfactionLabels[record.satisfaction_rating] }}</span>
              </div>
            </div>
            
            <div v-if="record.notes" class="record-notes">
              <p>{{ record.notes }}</p>
            </div>
            
            <div class="record-actions">
              <button 
                v-if="record.user_id === user?.id"
                @click="deleteRecord(record.id)"
                class="btn-delete"
              >
                削除
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
    
    <div v-if="message" :class="['message', message.includes('エラー') ? 'error' : 'success']">
      {{ message }}
    </div>
  </div>
</template>

<style scoped>
.training-record {
  background: white;
  padding: 1.5rem;
  border-radius: 0.75rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  margin-bottom: 1.5rem;
}

.header-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.header-section h3 {
  color: #1a202c;
  font-size: 1.25rem;
  margin: 0;
}

.record-form {
  background: #f8fafc;
  padding: 1.5rem;
  border-radius: 0.5rem;
  margin-bottom: 1.5rem;
}

.record-form h4 {
  color: #1f2937;
  margin-bottom: 1rem;
}

.form-group {
  margin-bottom: 1rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 500;
  color: #374151;
}

.input, .textarea, .select {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 0.375rem;
  font-size: 1rem;
}

.input:focus, .textarea:focus, .select:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.satisfaction-selector {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.satisfaction-option {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  cursor: pointer;
  padding: 0.5rem;
  border-radius: 0.375rem;
  transition: background-color 0.2s;
}

.satisfaction-option:hover {
  background: #f3f4f6;
}

.satisfaction-radio {
  margin: 0;
}

.satisfaction-label {
  font-size: 0.95rem;
}

.form-actions {
  display: flex;
  gap: 0.75rem;
  margin-top: 1.5rem;
}

.quick-record {
  background: #f0f9ff;
  padding: 1.5rem;
  border-radius: 0.5rem;
  margin-bottom: 1.5rem;
}

.quick-record h4 {
  color: #1e40af;
  margin-bottom: 1rem;
}

.menu-quick-list {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 0.75rem;
}

.quick-menu-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 1rem;
  background: white;
  border: 2px solid #dbeafe;
  border-radius: 0.5rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.quick-menu-btn:hover {
  border-color: #3b82f6;
  transform: translateY(-1px);
}

.menu-name {
  font-weight: 500;
  color: #1f2937;
  margin-bottom: 0.25rem;
}

.menu-duration {
  font-size: 0.875rem;
  color: #6b7280;
}

.filter-section {
  display: flex;
  gap: 1rem;
  margin-bottom: 1.5rem;
  padding: 1rem;
  background: #f9fafb;
  border-radius: 0.5rem;
}

.filter-group {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.filter-group label {
  font-weight: 500;
  color: #374151;
}

.filter-group .select {
  width: auto;
  min-width: 120px;
}

.records-grid {
  display: grid;
  gap: 1rem;
}

.record-card {
  border: 1px solid #e5e7eb;
  border-radius: 0.75rem;
  padding: 1.5rem;
  background: white;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.record-card:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.record-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 1rem;
}

.record-info h5 {
  color: #1f2937;
  margin-bottom: 0.25rem;
  font-size: 1.125rem;
}

.record-meta {
  display: flex;
  gap: 0.75rem;
  font-size: 0.875rem;
  color: #6b7280;
}

.difficulty-badge {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-weight: bold;
  font-size: 0.75rem;
}

.record-details {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
  margin-bottom: 1rem;
}

.detail-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.detail-label {
  font-size: 0.875rem;
  color: #6b7280;
}

.detail-value {
  font-weight: 500;
  color: #1f2937;
}

.record-notes {
  background: #f9fafb;
  padding: 0.75rem;
  border-radius: 0.375rem;
  margin-bottom: 1rem;
}

.record-notes p {
  color: #4b5563;
  line-height: 1.5;
  margin: 0;
}

.record-actions {
  display: flex;
  justify-content: flex-end;
}

.btn-delete {
  background: #fef2f2;
  color: #dc2626;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  cursor: pointer;
  font-size: 0.875rem;
  transition: background-color 0.2s;
}

.btn-delete:hover {
  background: #fee2e2;
}

.no-user, .no-family, .no-records {
  text-align: center;
  color: #6b7280;
  padding: 3rem 2rem;
}

.message {
  margin-top: 1rem;
  padding: 0.75rem;
  border-radius: 0.375rem;
  font-weight: 500;
}

.message.success {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #a7f3d0;
}

.message.error {
  background: #fee2e2;
  color: #991b1b;
  border: 1px solid #fca5a5;
}

@media (max-width: 768px) {
  .header-section {
    flex-direction: column;
    gap: 1rem;
    text-align: center;
  }
  
  .form-row {
    grid-template-columns: 1fr;
  }
  
  .form-actions {
    flex-direction: column;
  }
  
  .menu-quick-list {
    grid-template-columns: 1fr;
  }
  
  .filter-section {
    flex-direction: column;
    gap: 0.75rem;
  }
  
  .record-details {
    grid-template-columns: 1fr;
  }
  
  .record-header {
    flex-direction: column;
    gap: 0.5rem;
  }
}
</style>