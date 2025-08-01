<script setup>
import { ref, onMounted } from 'vue'
import { supabase } from '../supabase.js'

const emit = defineEmits(['profile-updated'])

const user = ref(null)
const profile = ref(null)
const isEditing = ref(false)
const displayName = ref('')
const message = ref('')
const isLoading = ref(false)

onMounted(async () => {
  await loadUserProfile()
})

const loadUserProfile = async () => {
  try {
    const { data: { user: currentUser } } = await supabase.auth.getUser()
    user.value = currentUser
    
    if (currentUser) {
      const { data, error } = await supabase
        .from('user_profiles')
        .select('*, family_groups(name)')
        .eq('id', currentUser.id)
        .single()
      
      if (error && error.code !== 'PGRST116') throw error
      
      profile.value = data
      if (data) {
        displayName.value = data.display_name
      }
    }
  } catch (error) {
    message.value = `プロファイル読み込みエラー: ${error.message}`
  }
}

const saveProfile = async () => {
  if (!displayName.value.trim()) {
    message.value = '表示名を入力してください'
    return
  }
  
  isLoading.value = true
  try {
    const profileData = {
      id: user.value.id,
      display_name: displayName.value,
      updated_at: new Date().toISOString()
    }
    
    const { data, error } = await supabase
      .from('user_profiles')
      .upsert(profileData)
      .select()
      .single()
    
    if (error) throw error
    
    profile.value = data
    isEditing.value = false
    message.value = 'プロファイルを保存しました'
    emit('profile-updated', data)
  } catch (error) {
    message.value = `保存エラー: ${error.message}`
  } finally {
    isLoading.value = false
  }
}

const startEditing = () => {
  isEditing.value = true
  displayName.value = profile.value?.display_name || ''
}

const cancelEditing = () => {
  isEditing.value = false
  displayName.value = profile.value?.display_name || ''
}
</script>

<template>
  <div class="user-profile">
    <h3>ユーザープロファイル</h3>
    
    <div v-if="user" class="profile-content">
      <div class="user-info">
        <p><strong>メールアドレス:</strong> {{ user.email }}</p>
        <p><strong>登録日:</strong> {{ new Date(user.created_at).toLocaleDateString('ja-JP') }}</p>
      </div>
      
      <div v-if="!isEditing && profile" class="profile-display">
        <p><strong>表示名:</strong> {{ profile.display_name }}</p>
        <p v-if="profile.family_groups">
          <strong>家族グループ:</strong> {{ profile.family_groups.name }}
        </p>
        <button @click="startEditing" class="btn btn-primary">編集</button>
      </div>
      
      <div v-else-if="!isEditing && !profile" class="profile-setup">
        <p>プロファイルを設定してください</p>
        <button @click="startEditing" class="btn btn-primary">プロファイル作成</button>
      </div>
      
      <div v-if="isEditing" class="profile-form">
        <div class="form-group">
          <label for="displayName">表示名:</label>
          <input
            id="displayName"
            v-model="displayName"
            type="text"
            placeholder="家族に表示される名前"
            maxlength="50"
            class="input"
          />
        </div>
        
        <div class="form-actions">
          <button @click="saveProfile" :disabled="isLoading" class="btn btn-primary">
            {{ isLoading ? '保存中...' : '保存' }}
          </button>
          <button @click="cancelEditing" class="btn btn-secondary">キャンセル</button>
        </div>
      </div>
      
      <div v-if="message" :class="['message', message.includes('エラー') ? 'error' : 'success']">
        {{ message }}
      </div>
    </div>
    
    <div v-else class="no-user">
      <p>ログインが必要です</p>
    </div>
  </div>
</template>

<style scoped>
.user-profile {
  background: white;
  padding: 1.5rem;
  border-radius: 0.75rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  margin-bottom: 1.5rem;
}

.user-profile h3 {
  color: #1a202c;
  margin-bottom: 1rem;
  font-size: 1.25rem;
}

.user-info {
  background: #f8fafc;
  padding: 1rem;
  border-radius: 0.5rem;
  margin-bottom: 1rem;
}

.user-info p {
  margin-bottom: 0.5rem;
  color: #4a5568;
}

.profile-display p {
  margin-bottom: 0.5rem;
  color: #2d3748;
}

.profile-setup {
  text-align: center;
  padding: 1rem;
  background: #fef3c7;
  border-radius: 0.5rem;
  margin-bottom: 1rem;
}

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 500;
  color: #2d3748;
}

.input {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 0.375rem;
  font-size: 1rem;
}

.input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-actions {
  display: flex;
  gap: 0.75rem;
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

.no-user {
  text-align: center;
  color: #6b7280;
  padding: 2rem;
}

@media (max-width: 768px) {
  .form-actions {
    flex-direction: column;
  }
  
  .form-actions .btn {
    width: 100%;
  }
}
</style>