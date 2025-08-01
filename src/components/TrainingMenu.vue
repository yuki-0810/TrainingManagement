<script setup>
import { ref, onMounted, computed } from 'vue'
import { supabase } from '../supabase.js'

const user = ref(null)
const userProfile = ref(null)
const trainingMenus = ref([])
const isCreatingMenu = ref(false)
const isEditingMenu = ref(false)
const editingMenuId = ref(null)
const message = ref('')
const isLoading = ref(false)

// フォーム項目
const menuForm = ref({
  name: '',
  description: '',
  duration_minutes: 30,
  instructions: '',
  difficulty_level: 3
})

const difficultyLabels = {
  1: '初心者',
  2: '初級',
  3: '中級',
  4: '上級',
  5: 'エキスパート'
}

const filteredMenus = ref([])
const filterDifficulty = ref(0) // 0 = all

const displayedMenus = computed(() => {
  if (filterDifficulty.value === 0) {
    return trainingMenus.value
  }
  return trainingMenus.value.filter(menu => menu.difficulty_level === filterDifficulty.value)
})

onMounted(async () => {
  await loadUserData()
  if (userProfile.value?.family_group_id) {
    await loadTrainingMenus()
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
    // まずトレーニングメニューを取得
    const { data: menus, error: menusError } = await supabase
      .from('training_menus')
      .select('*')
      .eq('family_group_id', userProfile.value.family_group_id)
      .order('created_at', { ascending: false })
    
    if (menusError) throw menusError
    
    // 作成者の情報を別途取得
    if (menus && menus.length > 0) {
      const creatorIds = [...new Set(menus.map(menu => menu.created_by).filter(Boolean))]
      
      if (creatorIds.length > 0) {
        const { data: profiles, error: profilesError } = await supabase
          .from('user_profiles')
          .select('id, display_name')
          .in('id', creatorIds)
        
        if (profilesError) {
          console.warn('プロファイル取得エラー:', profilesError)
        }
        
        // メニューに作成者情報をマージ
        trainingMenus.value = menus.map(menu => ({
          ...menu,
          creator_name: profiles?.find(p => p.id === menu.created_by)?.display_name || '不明'
        }))
      } else {
        trainingMenus.value = menus.map(menu => ({ ...menu, creator_name: '不明' }))
      }
    } else {
      trainingMenus.value = []
    }
  } catch (error) {
    message.value = `メニュー読み込みエラー: ${error.message}`
    trainingMenus.value = []
  }
}

const resetForm = () => {
  menuForm.value = {
    name: '',
    description: '',
    duration_minutes: 30,
    instructions: '',
    difficulty_level: 3
  }
}

const startCreating = () => {
  resetForm()
  isCreatingMenu.value = true
  isEditingMenu.value = false
  editingMenuId.value = null
}

const startEditing = (menu) => {
  menuForm.value = {
    name: menu.name,
    description: menu.description || '',
    duration_minutes: menu.duration_minutes || 30,
    instructions: menu.instructions || '',
    difficulty_level: menu.difficulty_level || 3
  }
  isEditingMenu.value = true
  isCreatingMenu.value = false
  editingMenuId.value = menu.id
}

const cancelForm = () => {
  resetForm()
  isCreatingMenu.value = false
  isEditingMenu.value = false
  editingMenuId.value = null
}

const saveMenu = async () => {
  if (!menuForm.value.name.trim()) {
    message.value = 'メニュー名を入力してください'
    return
  }
  
  if (!userProfile.value?.family_group_id) {
    message.value = '家族グループに参加してください'
    return
  }
  
  isLoading.value = true
  try {
    const menuData = {
      name: menuForm.value.name,
      description: menuForm.value.description,
      duration_minutes: parseInt(menuForm.value.duration_minutes),
      instructions: menuForm.value.instructions,
      difficulty_level: parseInt(menuForm.value.difficulty_level),
      family_group_id: userProfile.value.family_group_id,
      created_by: user.value.id,
      updated_at: new Date().toISOString()
    }
    
    let result
    if (isEditingMenu.value) {
      const { data, error } = await supabase
        .from('training_menus')
        .update(menuData)
        .eq('id', editingMenuId.value)
        .select()
        .single()
      result = { data, error }
    } else {
      const { data, error } = await supabase
        .from('training_menus')
        .insert([menuData])
        .select()
        .single()
      result = { data, error }
    }
    
    if (result.error) throw result.error
    
    message.value = isEditingMenu.value ? 'メニューを更新しました' : 'メニューを作成しました'
    cancelForm()
    await loadTrainingMenus()
  } catch (error) {
    message.value = `保存エラー: ${error.message}`
  } finally {
    isLoading.value = false
  }
}

const deleteMenu = async (menuId, menuName) => {
  if (!confirm(`「${menuName}」を削除しますか？`)) return
  
  isLoading.value = true
  try {
    const { error } = await supabase
      .from('training_menus')
      .delete()
      .eq('id', menuId)
    
    if (error) throw error
    
    message.value = 'メニューを削除しました'
    await loadTrainingMenus()
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
</script>

<template>
  <div class="training-menu">
    <div class="header-section">
      <h3>トレーニングメニュー管理</h3>
      <button 
        v-if="userProfile?.family_group_id" 
        @click="startCreating" 
        class="btn btn-primary"
        :disabled="isLoading"
      >
        新しいメニューを作成
      </button>
    </div>
    
    <div v-if="!user" class="no-user">
      <p>ログインが必要です</p>
    </div>
    
    <div v-else-if="!userProfile?.family_group_id" class="no-family">
      <p>家族グループに参加してからメニューを管理できます</p>
    </div>
    
    <div v-else class="menu-content">
      <!-- メニュー作成・編集フォーム -->
      <div v-if="isCreatingMenu || isEditingMenu" class="menu-form">
        <h4>{{ isEditingMenu ? 'メニューを編集' : '新しいメニューを作成' }}</h4>
        
        <div class="form-group">
          <label for="menuName">メニュー名 *</label>
          <input
            id="menuName"
            v-model="menuForm.name"
            type="text"
            placeholder="例：朝の軽運動、筋トレ基本セット"
            maxlength="100"
            class="input"
          />
        </div>
        
        <div class="form-group">
          <label for="menuDescription">説明</label>
          <textarea
            id="menuDescription"
            v-model="menuForm.description"
            placeholder="メニューの目的や効果について"
            rows="3"
            class="textarea"
          ></textarea>
        </div>
        
        <div class="form-row">
          <div class="form-group">
            <label for="duration">目安時間（分）</label>
            <input
              id="duration"
              v-model="menuForm.duration_minutes"
              type="number"
              min="5"
              max="180"
              class="input"
            />
          </div>
          
          <div class="form-group">
            <label for="difficulty">難易度</label>
            <select id="difficulty" v-model="menuForm.difficulty_level" class="select">
              <option v-for="(label, value) in difficultyLabels" :key="value" :value="parseInt(value)">
                {{ label }}
              </option>
            </select>
          </div>
        </div>
        
        <div class="form-group">
          <label for="instructions">実施手順</label>
          <textarea
            id="instructions"
            v-model="menuForm.instructions"
            placeholder="1. ウォーミングアップ（5分）&#10;2. メインエクササイズ（20分）&#10;3. クールダウン（5分）"
            rows="6"
            class="textarea"
          ></textarea>
        </div>
        
        <div class="form-actions">
          <button @click="saveMenu" :disabled="isLoading" class="btn btn-primary">
            {{ isLoading ? '保存中...' : (isEditingMenu ? '更新' : '作成') }}
          </button>
          <button @click="cancelForm" class="btn btn-secondary">キャンセル</button>
        </div>
      </div>
      
      <!-- メニューフィルター -->
      <div v-if="!isCreatingMenu && !isEditingMenu && trainingMenus.length > 0" class="filter-section">
        <label for="difficultyFilter">難易度フィルター:</label>
        <select id="difficultyFilter" v-model="filterDifficulty" class="select">
          <option value="0">すべて</option>
          <option v-for="(label, value) in difficultyLabels" :key="value" :value="parseInt(value)">
            {{ label }}
          </option>
        </select>
      </div>
      
      <!-- メニュー一覧 -->
      <div v-if="!isCreatingMenu && !isEditingMenu" class="menu-list">
        <div v-if="displayedMenus.length === 0" class="no-menus">
          <p>{{ trainingMenus.length === 0 ? 'まだメニューがありません' : 'フィルター条件に合うメニューがありません' }}</p>
        </div>
        
        <div v-else class="menu-grid">
          <div v-for="menu in displayedMenus" :key="menu.id" class="menu-card">
            <div class="menu-header">
              <h4>{{ menu.name }}</h4>
              <div class="menu-meta">
                <span class="duration">{{ formatDuration(menu.duration_minutes) }}</span>
                <span :class="['difficulty', `level-${menu.difficulty_level}`]">
                  {{ difficultyLabels[menu.difficulty_level] }}
                </span>
              </div>
            </div>
            
            <div v-if="menu.description" class="menu-description">
              <p>{{ menu.description }}</p>
            </div>
            
            <div v-if="menu.instructions" class="menu-instructions">
              <h5>実施手順:</h5>
              <pre>{{ menu.instructions }}</pre>
            </div>
            
            <div class="menu-footer">
              <div class="creator-info">
                <span>作成者: {{ menu.creator_name }}</span>
                <span class="created-date">{{ new Date(menu.created_at).toLocaleDateString('ja-JP') }}</span>
              </div>
              
              <div class="menu-actions">
                <button @click="startEditing(menu)" class="btn-edit">編集</button>
                <button @click="deleteMenu(menu.id, menu.name)" class="btn-delete">削除</button>
              </div>
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
.training-menu {
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

.menu-form {
  background: #f8fafc;
  padding: 1.5rem;
  border-radius: 0.5rem;
  margin-bottom: 1.5rem;
}

.menu-form h4 {
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

.textarea {
  resize: vertical;
  font-family: inherit;
}

.form-actions {
  display: flex;
  gap: 0.75rem;
  margin-top: 1.5rem;
}

.filter-section {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
  padding: 1rem;
  background: #f9fafb;
  border-radius: 0.5rem;
}

.filter-section label {
  font-weight: 500;
  color: #374151;
}

.filter-section .select {
  width: auto;
  min-width: 120px;
}

.menu-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
  gap: 1.5rem;
}

.menu-card {
  border: 1px solid #e5e7eb;
  border-radius: 0.75rem;
  padding: 1.5rem;
  background: white;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.menu-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.menu-header {
  margin-bottom: 1rem;
}

.menu-header h4 {
  color: #1f2937;
  margin-bottom: 0.5rem;
  font-size: 1.125rem;
}

.menu-meta {
  display: flex;
  gap: 0.75rem;
  align-items: center;
}

.duration {
  background: #dbeafe;
  color: #1e40af;
  padding: 0.25rem 0.75rem;
  border-radius: 1rem;
  font-size: 0.875rem;
  font-weight: 500;
}

.difficulty {
  padding: 0.25rem 0.75rem;
  border-radius: 1rem;
  font-size: 0.875rem;
  font-weight: 500;
}

.difficulty.level-1 { background: #dcfce7; color: #166534; }
.difficulty.level-2 { background: #fef3c7; color: #92400e; }
.difficulty.level-3 { background: #dbeafe; color: #1e40af; }
.difficulty.level-4 { background: #fde2e8; color: #9f1239; }
.difficulty.level-5 { background: #e0e7ff; color: #3730a3; }

.menu-description {
  margin-bottom: 1rem;
}

.menu-description p {
  color: #4b5563;
  line-height: 1.5;
}

.menu-instructions {
  margin-bottom: 1rem;
}

.menu-instructions h5 {
  color: #374151;
  margin-bottom: 0.5rem;
  font-size: 0.95rem;
}

.menu-instructions pre {
  background: #f9fafb;
  padding: 0.75rem;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  line-height: 1.4;
  color: #4b5563;
  white-space: pre-wrap;
  font-family: inherit;
}

.menu-footer {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #e5e7eb;
}

.creator-info {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  font-size: 0.875rem;
  color: #6b7280;
}

.menu-actions {
  display: flex;
  gap: 0.5rem;
}

.btn-edit, .btn-delete {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn-edit {
  background: #f3f4f6;
  color: #374151;
}

.btn-edit:hover {
  background: #e5e7eb;
}

.btn-delete {
  background: #fef2f2;
  color: #dc2626;
}

.btn-delete:hover {
  background: #fee2e2;
}

.no-user, .no-family, .no-menus {
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
  
  .filter-section {
    flex-direction: column;
    align-items: flex-start;
  }
  
  .menu-grid {
    grid-template-columns: 1fr;
  }
  
  .menu-footer {
    flex-direction: column;
    gap: 1rem;
    align-items: flex-start;
  }
  
  .menu-actions {
    width: 100%;
    justify-content: flex-end;
  }
}
</style>