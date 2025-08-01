<script setup>
import { ref, onMounted, computed } from 'vue'
import { supabase } from '../supabase.js'

const user = ref(null)
const userProfile = ref(null)
const trainingRecords = ref([])
const familyMembers = ref([])
const message = ref('')
const isLoading = ref(false)

// カレンダー表示設定
const currentDate = ref(new Date())
const selectedDate = ref(null)
const viewMode = ref('month') // 'month', 'week'
const selectedMember = ref('all') // 'all' or user_id

// 月/週の表示用
const monthNames = [
  '1月', '2月', '3月', '4月', '5月', '6月',
  '7月', '8月', '9月', '10月', '11月', '12月'
]

const weekDays = ['日', '月', '火', '水', '木', '金', '土']

// 現在の月の日付配列を生成
const calendarDays = computed(() => {
  const year = currentDate.value.getFullYear()
  const month = currentDate.value.getMonth()
  
  // 月の最初の日と最後の日
  const firstDay = new Date(year, month, 1)
  const lastDay = new Date(year, month + 1, 0)
  
  // 月の最初の週の日曜日から開始
  const startDate = new Date(firstDay)
  startDate.setDate(startDate.getDate() - firstDay.getDay())
  
  // 月の最後の週の土曜日まで
  const endDate = new Date(lastDay)
  endDate.setDate(endDate.getDate() + (6 - lastDay.getDay()))
  
  const days = []
  const currentDay = new Date(startDate)
  
  while (currentDay <= endDate) {
    days.push({
      date: new Date(currentDay),
      isCurrentMonth: currentDay.getMonth() === month,
      isToday: isToday(currentDay),
      records: getRecordsForDate(currentDay)
    })
    currentDay.setDate(currentDay.getDate() + 1)
  }
  
  return days
})

// 現在の週の日付配列を生成
const weekDays_computed = computed(() => {
  const startOfWeek = new Date(currentDate.value)
  startOfWeek.setDate(startOfWeek.getDate() - startOfWeek.getDay())
  
  const days = []
  for (let i = 0; i < 7; i++) {
    const day = new Date(startOfWeek)
    day.setDate(day.getDate() + i)
    days.push({
      date: new Date(day),
      isToday: isToday(day),
      records: getRecordsForDate(day)
    })
  }
  
  return days
})

// 選択された日付の記録
const selectedDateRecords = computed(() => {
  if (!selectedDate.value) return []
  
  return getRecordsForDate(selectedDate.value).filter(record => {
    if (selectedMember.value === 'all') return true
    return record.user_id === selectedMember.value
  })
})

// 統計情報
const monthlyStats = computed(() => {
  const year = currentDate.value.getFullYear()
  const month = currentDate.value.getMonth()
  
  const monthRecords = trainingRecords.value.filter(record => {
    const recordDate = new Date(record.completed_at)
    return recordDate.getFullYear() === year && recordDate.getMonth() === month
  })
  
  const memberStats = familyMembers.value.map(member => {
    const memberRecords = monthRecords.filter(r => r.user_id === member.id)
    const totalDuration = memberRecords.reduce((sum, r) => sum + (r.duration_minutes || 0), 0)
    const avgSatisfaction = memberRecords.length > 0 
      ? memberRecords.reduce((sum, r) => sum + (r.satisfaction_rating || 0), 0) / memberRecords.length
      : 0
    
    return {
      member,
      recordCount: memberRecords.length,
      totalDuration,
      avgSatisfaction: Math.round(avgSatisfaction * 10) / 10
    }
  })
  
  return {
    totalRecords: monthRecords.length,
    totalDuration: monthRecords.reduce((sum, r) => sum + (r.duration_minutes || 0), 0),
    memberStats
  }
})

onMounted(async () => {
  await loadUserData()
  if (userProfile.value?.family_group_id) {
    await loadFamilyMembers()
    await loadTrainingRecords()
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

const loadFamilyMembers = async () => {
  try {
    const { data, error } = await supabase
      .from('user_profiles')
      .select('id, display_name')
      .eq('family_group_id', userProfile.value.family_group_id)
    
    if (error) throw error
    familyMembers.value = data || []
  } catch (error) {
    console.error('家族メンバー読み込みエラー:', error)
  }
}

const loadTrainingRecords = async () => {
  try {
    const familyMemberIds = familyMembers.value.map(m => m.id)
    
    if (familyMemberIds.length === 0) {
      trainingRecords.value = []
      return
    }
    
    // まずトレーニング記録を取得
    const { data: records, error: recordsError } = await supabase
      .from('training_records')
      .select('*')
      .in('user_id', familyMemberIds)
      .order('completed_at', { ascending: false })
    
    if (recordsError) throw recordsError
    
    if (!records || records.length === 0) {
      trainingRecords.value = []
      return
    }
    
    // トレーニングメニュー情報を別途取得
    const menuIds = [...new Set(records.map(r => r.training_menu_id).filter(Boolean))]
    let menus = []
    
    if (menuIds.length > 0) {
      const { data: menusData, error: menusError } = await supabase
        .from('training_menus')
        .select('id, name, difficulty_level')
        .in('id', menuIds)
      
      if (menusError) {
        console.warn('メニュー取得エラー:', menusError)
      } else {
        menus = menusData || []
      }
    }
    
    // データをマージ
    trainingRecords.value = records.map(record => ({
      ...record,
      user_name: familyMembers.value.find(m => m.id === record.user_id)?.display_name || '不明',
      training_menus: menus.find(m => m.id === record.training_menu_id) || null
    }))
    
    console.log('読み込まれた記録数:', trainingRecords.value.length)
  } catch (error) {
    message.value = `記録読み込みエラー: ${error.message}`
    console.error('記録読み込みエラー:', error)
  }
}

const isToday = (date) => {
  const today = new Date()
  return date.toDateString() === today.toDateString()
}

const getRecordsForDate = (date) => {
  const targetYear = date.getFullYear()
  const targetMonth = date.getMonth()
  const targetDate = date.getDate()
  
  return trainingRecords.value.filter(record => {
    const recordDate = new Date(record.completed_at)
    return recordDate.getFullYear() === targetYear &&
           recordDate.getMonth() === targetMonth &&
           recordDate.getDate() === targetDate
  })
}

const selectDate = (day) => {
  selectedDate.value = day.date
}

const navigateMonth = (direction) => {
  const newDate = new Date(currentDate.value)
  newDate.setMonth(newDate.getMonth() + direction)
  currentDate.value = newDate
}

const navigateWeek = (direction) => {
  const newDate = new Date(currentDate.value)
  newDate.setDate(newDate.getDate() + (direction * 7))
  currentDate.value = newDate
}

const goToToday = () => {
  currentDate.value = new Date()
  selectedDate.value = new Date()
}

const debugInfo = () => {
  console.log('=== カレンダーデバッグ情報 ===')
  console.log('家族メンバー:', familyMembers.value)
  console.log('トレーニング記録:', trainingRecords.value)
  console.log('現在の日付:', currentDate.value)
  console.log('選択された日:', selectedDate.value)
  
  if (trainingRecords.value.length > 0) {
    const sampleRecord = trainingRecords.value[0]
    console.log('サンプル記録:', sampleRecord)
    const recordDate = new Date(sampleRecord.completed_at)
    console.log('記録日時:', recordDate)
    console.log('今日の記録:', getRecordsForDate(new Date()))
  }
  
  message.value = `デバッグ: ${familyMembers.value.length}人, ${trainingRecords.value.length}記録`
}

const formatDate = (date) => {
  return date.toLocaleDateString('ja-JP', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

const formatTime = (dateTime) => {
  return new Date(dateTime).toLocaleTimeString('ja-JP', {
    hour: '2-digit',
    minute: '2-digit'
  })
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

const getSatisfactionEmoji = (rating) => {
  const emojis = { 1: '😞', 2: '😕', 3: '😐', 4: '😊', 5: '🤩' }
  return emojis[rating] || '😐'
}
</script>

<template>
  <div class="training-calendar">
    <div class="header-section">
      <h3>トレーニングカレンダー</h3>
      <div class="view-controls">
        <button 
          @click="viewMode = 'month'" 
          :class="['view-btn', { active: viewMode === 'month' }]"
        >
          月表示
        </button>
        <button 
          @click="viewMode = 'week'" 
          :class="['view-btn', { active: viewMode === 'week' }]"
        >
          週表示
        </button>
      </div>
    </div>
    
    <div v-if="!user" class="no-user">
      <p>ログインが必要です</p>
    </div>
    
    <div v-else-if="!userProfile?.family_group_id" class="no-family">
      <p>家族グループに参加してからカレンダーを表示できます</p>
    </div>
    
    <div v-else class="calendar-content">
      <!-- デバッグ情報 -->
      <div v-if="message.includes('デバッグ')" class="debug-info">
        <p>家族メンバー数: {{ familyMembers.length }}</p>
        <p>記録数: {{ trainingRecords.length }}</p>
        <p>選択された日付: {{ selectedDate ? formatDate(selectedDate) : 'なし' }}</p>
      </div>
      
      <!-- ナビゲーション -->
      <div class="calendar-nav">
        <button 
          @click="viewMode === 'month' ? navigateMonth(-1) : navigateWeek(-1)" 
          class="nav-btn"
        >
          ‹
        </button>
        
        <div class="current-period">
          <h4 v-if="viewMode === 'month'">
            {{ currentDate.getFullYear() }}年 {{ monthNames[currentDate.getMonth()] }}
          </h4>
          <h4 v-else>
            {{ formatDate(weekDays_computed[0].date) }} - {{ formatDate(weekDays_computed[6].date) }}
          </h4>
        </div>
        
        <button 
          @click="viewMode === 'month' ? navigateMonth(1) : navigateWeek(1)" 
          class="nav-btn"
        >
          ›
        </button>
        
        <button @click="goToToday" class="today-btn">今日</button>
        <button @click="debugInfo" class="debug-btn">デバッグ</button>
      </div>
      
      <!-- フィルター -->
      <div class="filter-section">
        <div class="filter-group">
          <label for="memberFilter">メンバー:</label>
          <select id="memberFilter" v-model="selectedMember" class="select">
            <option value="all">全員</option>
            <option v-for="member in familyMembers" :key="member.id" :value="member.id">
              {{ member.display_name }}
            </option>
          </select>
        </div>
      </div>
      
      <!-- 月間統計 -->
      <div v-if="viewMode === 'month'" class="monthly-stats">
        <h4>{{ monthNames[currentDate.getMonth()] }}の統計</h4>
        <div class="stats-grid">
          <div class="stat-card">
            <div class="stat-value">{{ monthlyStats.totalRecords }}</div>
            <div class="stat-label">総記録数</div>
          </div>
          <div class="stat-card">
            <div class="stat-value">{{ formatDuration(monthlyStats.totalDuration) }}</div>
            <div class="stat-label">総時間</div>
          </div>
        </div>
        
        <div class="member-stats">
          <div v-for="stat in monthlyStats.memberStats" :key="stat.member.id" class="member-stat">
            <span class="member-name">{{ stat.member.display_name }}</span>
            <span class="member-records">{{ stat.recordCount }}回</span>
            <span class="member-duration">{{ formatDuration(stat.totalDuration) }}</span>
            <span class="member-satisfaction">{{ stat.avgSatisfaction }}⭐</span>
          </div>
        </div>
      </div>
      
      <!-- カレンダーグリッド -->
      <div class="calendar-grid">
        <!-- 曜日ヘッダー -->
        <div class="weekday-header">
          <div v-for="day in weekDays" :key="day" class="weekday">{{ day }}</div>
        </div>
        
        <!-- 月表示 -->
        <div v-if="viewMode === 'month'" class="month-grid">
          <div 
            v-for="day in calendarDays" 
            :key="day.date.toISOString()"
            :class="[
              'calendar-day',
              { 
                'other-month': !day.isCurrentMonth,
                'today': day.isToday,
                'selected': selectedDate && day.date.toDateString() === selectedDate.toDateString(),
                'has-records': day.records.length > 0
              }
            ]"
            @click="selectDate(day)"
          >
            <div class="day-number">{{ day.date.getDate() }}</div>
            <div v-if="day.records.length > 0" class="day-records">
              <div 
                v-for="record in day.records.slice(0, 3)" 
                :key="record.id"
                class="record-dot"
                :style="{ backgroundColor: getDifficultyColor(record.training_menus?.difficulty_level) }"
                :title="`${record.user_name}: ${record.training_menus?.name || '不明'}`"
              ></div>
              <div v-if="day.records.length > 3" class="more-records">
                +{{ day.records.length - 3 }}
              </div>
            </div>
          </div>
        </div>
        
        <!-- 週表示 -->
        <div v-if="viewMode === 'week'" class="week-grid">
          <div 
            v-for="day in weekDays_computed" 
            :key="day.date.toISOString()"
            :class="[
              'week-day',
              { 
                'today': day.isToday,
                'selected': selectedDate && day.date.toDateString() === selectedDate.toDateString()
              }
            ]"
            @click="selectDate(day)"
          >
            <div class="week-day-header">
              <div class="week-day-name">{{ weekDays[day.date.getDay()] }}</div>
              <div class="week-day-number">{{ day.date.getDate() }}</div>
            </div>
            <div class="week-day-records">
              <div 
                v-for="record in day.records" 
                :key="record.id"
                class="week-record-item"
                :style="{ borderLeftColor: getDifficultyColor(record.training_menus?.difficulty_level) }"
              >
                <div class="record-time">{{ formatTime(record.completed_at) }}</div>
                <div class="record-name">{{ record.training_menus?.name || '不明' }}</div>
                <div class="record-user">{{ record.user_name }}</div>
              </div>
            </div>
          </div>
        </div>
      </div>
      
      <!-- 選択された日の詳細 -->
      <div v-if="selectedDate && selectedDateRecords.length > 0" class="selected-date-details">
        <h4>{{ formatDate(selectedDate) }}の記録</h4>
        <div class="selected-records">
          <div v-for="record in selectedDateRecords" :key="record.id" class="selected-record">
            <div class="record-header">
              <div class="record-info">
                <span class="record-title">{{ record.training_menus?.name || '不明' }}</span>
                <span class="record-user">{{ record.user_name }}</span>
              </div>
              <div class="record-meta">
                <span class="record-time">{{ formatTime(record.completed_at) }}</span>
                <span class="record-duration">{{ formatDuration(record.duration_minutes) }}</span>
                <span class="record-satisfaction">{{ getSatisfactionEmoji(record.satisfaction_rating) }}</span>
              </div>
            </div>
            <div v-if="record.notes" class="record-notes">
              {{ record.notes }}
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
.training-calendar {
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

.view-controls {
  display: flex;
  gap: 0.5rem;
}

.view-btn {
  padding: 0.5rem 1rem;
  border: 1px solid #d1d5db;
  background: white;
  color: #374151;
  border-radius: 0.375rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.view-btn:hover {
  border-color: #3b82f6;
}

.view-btn.active {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

.calendar-nav {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.nav-btn {
  background: #f3f4f6;
  border: none;
  width: 40px;
  height: 40px;
  border-radius: 0.375rem;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.25rem;
  cursor: pointer;
  transition: background-color 0.2s;
}

.nav-btn:hover {
  background: #e5e7eb;
}

.current-period h4 {
  color: #1f2937;
  margin: 0;
  font-size: 1.25rem;
}

.today-btn, .debug-btn {
  padding: 0.5rem 1rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 0.375rem;
  cursor: pointer;
  font-size: 0.875rem;
}

.today-btn:hover, .debug-btn:hover {
  background: #2563eb;
}

.debug-btn {
  background: #f59e0b;
}

.debug-btn:hover {
  background: #d97706;
}

.debug-info {
  background: #fef3c7;
  padding: 1rem;
  border-radius: 0.375rem;
  margin-bottom: 1rem;
  font-size: 0.875rem;
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

.select {
  padding: 0.5rem;
  border: 1px solid #d1d5db;
  border-radius: 0.375rem;
  min-width: 120px;
}

.monthly-stats {
  background: #f0f9ff;
  padding: 1.5rem;
  border-radius: 0.5rem;
  margin-bottom: 1.5rem;
}

.monthly-stats h4 {
  color: #1e40af;
  margin-bottom: 1rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 1rem;
  margin-bottom: 1rem;
}

.stat-card {
  background: white;
  padding: 1rem;
  border-radius: 0.375rem;
  text-align: center;
}

.stat-value {
  font-size: 1.5rem;
  font-weight: bold;
  color: #1f2937;
}

.stat-label {
  font-size: 0.875rem;
  color: #6b7280;
}

.member-stats {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.member-stat {
  display: grid;
  grid-template-columns: 1fr auto auto auto;
  gap: 1rem;
  padding: 0.5rem;
  background: white;
  border-radius: 0.375rem;
  align-items: center;
}

.member-name {
  font-weight: 500;
  color: #1f2937;
}

.member-records, .member-duration, .member-satisfaction {
  font-size: 0.875rem;
  color: #6b7280;
}

.calendar-grid {
  margin-bottom: 1.5rem;
}

.weekday-header {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 1px;
  margin-bottom: 1px;
}

.weekday {
  background: #f3f4f6;
  padding: 0.75rem;
  text-align: center;
  font-weight: 500;
  color: #374151;
}

.month-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 1px;
  background: #e5e7eb;
}

.calendar-day {
  background: white;
  min-height: 100px;
  padding: 0.5rem;
  cursor: pointer;
  transition: background-color 0.2s;
  position: relative;
}

.calendar-day:hover {
  background: #f9fafb;
}

.calendar-day.other-month {
  background: #f9fafb;
  color: #9ca3af;
}

.calendar-day.today {
  background: #dbeafe;
}

.calendar-day.selected {
  background: #3b82f6;
  color: white;
}

.calendar-day.has-records {
  border-left: 3px solid #10b981;
}

.day-number {
  font-weight: 500;
  margin-bottom: 0.25rem;
}

.day-records {
  display: flex;
  flex-wrap: wrap;
  gap: 2px;
}

.record-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
}

.more-records {
  font-size: 0.75rem;
  color: #6b7280;
}

.week-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 1px;
  background: #e5e7eb;
}

.week-day {
  background: white;
  min-height: 200px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.week-day:hover {
  background: #f9fafb;
}

.week-day.today {
  background: #dbeafe;
}

.week-day.selected {
  background: #3b82f6;
  color: white;
}

.week-day-header {
  padding: 0.75rem;
  border-bottom: 1px solid #e5e7eb;
  text-align: center;
}

.week-day-name {
  font-size: 0.875rem;
  color: #6b7280;
}

.week-day-number {
  font-weight: 500;
  font-size: 1.125rem;
  color: #1f2937;
}

.week-day-records {
  padding: 0.5rem;
}

.week-record-item {
  background: #f9fafb;
  border-left: 3px solid #6b7280;
  padding: 0.5rem;
  margin-bottom: 0.5rem;
  border-radius: 0.25rem;
  font-size: 0.875rem;
}

.record-time {
  font-weight: 500;
  color: #1f2937;
}

.record-name {
  color: #4b5563;
}

.record-user {
  color: #6b7280;
  font-size: 0.75rem;
}

.selected-date-details {
  background: #f8fafc;
  padding: 1.5rem;
  border-radius: 0.5rem;
}

.selected-date-details h4 {
  color: #1f2937;
  margin-bottom: 1rem;
}

.selected-records {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.selected-record {
  background: white;
  padding: 1rem;
  border-radius: 0.375rem;
  border-left: 3px solid #3b82f6;
}

.selected-record .record-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 0.5rem;
}

.record-title {
  font-weight: 500;
  color: #1f2937;
}

.selected-record .record-user {
  font-size: 0.875rem;
  color: #6b7280;
}

.record-meta {
  display: flex;
  gap: 0.75rem;
  font-size: 0.875rem;
  color: #6b7280;
}

.record-notes {
  font-size: 0.875rem;
  color: #4b5563;
  font-style: italic;
}

.no-user, .no-family {
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
  
  .calendar-nav {
    flex-wrap: wrap;
    gap: 0.5rem;
  }
  
  .stats-grid {
    grid-template-columns: 1fr;
  }
  
  .member-stat {
    grid-template-columns: 1fr;
    text-align: center;
    gap: 0.25rem;
  }
  
  .calendar-day {
    min-height: 80px;
    padding: 0.25rem;
  }
  
  .week-day {
    min-height: 150px;
  }
  
  .selected-record .record-header {
    flex-direction: column;
    gap: 0.5rem;
  }
  
  .record-meta {
    flex-direction: column;
    gap: 0.25rem;
  }
}
</style>