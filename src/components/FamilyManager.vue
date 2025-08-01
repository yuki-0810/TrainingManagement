<script setup>
import { ref, onMounted } from 'vue'
import { supabase } from '../supabase.js'

const emit = defineEmits(['family-updated'])

const user = ref(null)
const userProfile = ref(null)
const familyGroup = ref(null)
const familyMembers = ref([])
const isCreatingGroup = ref(false)
const isJoiningGroup = ref(false)
const groupName = ref('')
const inviteCode = ref('')
const message = ref('')
const isLoading = ref(false)

onMounted(async () => {
  await loadUserData()
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
      
      if (profile?.family_group_id) {
        familyGroup.value = profile.family_groups
        await loadFamilyMembers(profile.family_group_id)
      }
    }
  } catch (error) {
    console.error('データ読み込みエラー:', error)
  }
}

const loadFamilyMembers = async (familyGroupId) => {
  try {
    const { data, error } = await supabase
      .from('user_profiles')
      .select('id, display_name, created_at')
      .eq('family_group_id', familyGroupId)
      .order('created_at', { ascending: true })
    
    if (error) throw error
    familyMembers.value = data || []
  } catch (error) {
    message.value = `家族メンバー読み込みエラー: ${error.message}`
  }
}

const generateInviteCode = () => {
  return Math.random().toString(36).substr(2, 8).toUpperCase()
}

const createFamilyGroup = async () => {
  if (!groupName.value.trim()) {
    message.value = 'グループ名を入力してください'
    return
  }
  
  if (!userProfile.value) {
    message.value = '先にユーザープロファイルを作成してください'
    return
  }
  
  isLoading.value = true
  try {
    const newInviteCode = generateInviteCode()
    
    // 家族グループを作成
    const { data: newGroup, error: groupError } = await supabase
      .from('family_groups')
      .insert([{
        name: groupName.value,
        invite_code: newInviteCode
      }])
      .select()
      .single()
    
    if (groupError) throw groupError
    
    // ユーザープロファイルを更新
    const { error: profileError } = await supabase
      .from('user_profiles')
      .update({ family_group_id: newGroup.id })
      .eq('id', user.value.id)
    
    if (profileError) throw profileError
    
    familyGroup.value = newGroup
    userProfile.value.family_group_id = newGroup.id
    isCreatingGroup.value = false
    groupName.value = ''
    message.value = '家族グループを作成しました！'
    
    await loadFamilyMembers(newGroup.id)
    emit('family-updated', newGroup)
  } catch (error) {
    message.value = `グループ作成エラー: ${error.message}`
  } finally {
    isLoading.value = false
  }
}

const joinFamilyGroup = async () => {
  if (!inviteCode.value.trim()) {
    message.value = '招待コードを入力してください'
    return
  }
  
  if (!userProfile.value) {
    message.value = '先にユーザープロファイルを作成してください'
    return
  }
  
  isLoading.value = true
  try {
    // 招待コードで家族グループを検索
    const { data: group, error: groupError } = await supabase
      .from('family_groups')
      .select('*')
      .eq('invite_code', inviteCode.value.toUpperCase())
      .single()
    
    if (groupError) {
      if (groupError.code === 'PGRST116') {
        throw new Error('招待コードが見つかりません')
      }
      throw groupError
    }
    
    // ユーザープロファイルを更新
    const { error: profileError } = await supabase
      .from('user_profiles')
      .update({ family_group_id: group.id })
      .eq('id', user.value.id)
    
    if (profileError) throw profileError
    
    familyGroup.value = group
    userProfile.value.family_group_id = group.id
    isJoiningGroup.value = false
    inviteCode.value = ''
    message.value = `「${group.name}」に参加しました！`
    
    await loadFamilyMembers(group.id)
    emit('family-updated', group)
  } catch (error) {
    message.value = `参加エラー: ${error.message}`
  } finally {
    isLoading.value = false
  }
}

const leaveFamilyGroup = async () => {
  if (!confirm('本当に家族グループから脱退しますか？')) return
  
  isLoading.value = true
  try {
    const { error } = await supabase
      .from('user_profiles')
      .update({ family_group_id: null })
      .eq('id', user.value.id)
    
    if (error) throw error
    
    familyGroup.value = null
    familyMembers.value = []
    userProfile.value.family_group_id = null
    message.value = '家族グループから脱退しました'
    emit('family-updated', null)
  } catch (error) {
    message.value = `脱退エラー: ${error.message}`
  } finally {
    isLoading.value = false
  }
}

const copyInviteCode = () => {
  if (familyGroup.value?.invite_code) {
    navigator.clipboard.writeText(familyGroup.value.invite_code)
    message.value = '招待コードをコピーしました'
  }
}
</script>

<template>
  <div class="family-manager">
    <h3>家族グループ管理</h3>
    
    <div v-if="user && userProfile">
      <!-- 家族グループに参加している場合 -->
      <div v-if="familyGroup" class="family-group-info">
        <div class="group-header">
          <h4>{{ familyGroup.name }}</h4>
          <span class="member-count">{{ familyMembers.length }}人</span>
        </div>
        
        <div class="invite-section">
          <p><strong>招待コード:</strong></p>
          <div class="invite-code-display">
            <code>{{ familyGroup.invite_code }}</code>
            <button @click="copyInviteCode" class="btn-copy">📋</button>
          </div>
          <p class="invite-help">このコードを家族に共有して招待してください</p>
        </div>
        
        <div class="family-members">
          <h5>メンバー</h5>
          <div class="member-list">
            <div v-for="member in familyMembers" :key="member.id" class="member-item">
              <span class="member-name">{{ member.display_name }}</span>
              <span class="member-joined">{{ new Date(member.created_at).toLocaleDateString('ja-JP') }}参加</span>
            </div>
          </div>
        </div>
        
        <button @click="leaveFamilyGroup" :disabled="isLoading" class="btn btn-danger">
          グループから脱退
        </button>
      </div>
      
      <!-- 家族グループに参加していない場合 -->
      <div v-else class="no-family-group">
        <p>家族グループに参加していません</p>
        
        <div class="group-actions">
          <button @click="isCreatingGroup = true" class="btn btn-primary">
            新しいグループを作成
          </button>
          <button @click="isJoiningGroup = true" class="btn btn-secondary">
            既存のグループに参加
          </button>
        </div>
        
        <!-- グループ作成フォーム -->
        <div v-if="isCreatingGroup" class="form-section">
          <h4>新しい家族グループを作成</h4>
          <div class="form-group">
            <input
              v-model="groupName"
              type="text"
              placeholder="グループ名（例：田中家）"
              maxlength="100"
              class="input"
            />
          </div>
          <div class="form-actions">
            <button @click="createFamilyGroup" :disabled="isLoading" class="btn btn-primary">
              {{ isLoading ? '作成中...' : '作成' }}
            </button>
            <button @click="isCreatingGroup = false" class="btn btn-secondary">
              キャンセル
            </button>
          </div>
        </div>
        
        <!-- グループ参加フォーム -->
        <div v-if="isJoiningGroup" class="form-section">
          <h4>家族グループに参加</h4>
          <div class="form-group">
            <input
              v-model="inviteCode"
              type="text"
              placeholder="招待コードを入力"
              maxlength="8"
              class="input"
              style="text-transform: uppercase"
            />
          </div>
          <div class="form-actions">
            <button @click="joinFamilyGroup" :disabled="isLoading" class="btn btn-primary">
              {{ isLoading ? '参加中...' : '参加' }}
            </button>
            <button @click="isJoiningGroup = false" class="btn btn-secondary">
              キャンセル
            </button>
          </div>
        </div>
      </div>
    </div>
    
    <div v-else-if="!userProfile" class="no-profile">
      <p>先にユーザープロファイルを作成してください</p>
    </div>
    
    <div v-else class="no-user">
      <p>ログインが必要です</p>
    </div>
    
    <div v-if="message" :class="['message', message.includes('エラー') ? 'error' : 'success']">
      {{ message }}
    </div>
  </div>
</template>

<style scoped>
.family-manager {
  background: white;
  padding: 1.5rem;
  border-radius: 0.75rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  margin-bottom: 1.5rem;
}

.family-manager h3 {
  color: #1a202c;
  margin-bottom: 1rem;
  font-size: 1.25rem;
}

.family-group-info {
  space-y: 1rem;
}

.group-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  background: #f0f9ff;
  border-radius: 0.5rem;
  margin-bottom: 1rem;
}

.group-header h4 {
  color: #1e40af;
  margin: 0;
}

.member-count {
  background: #3b82f6;
  color: white;
  padding: 0.25rem 0.75rem;
  border-radius: 1rem;
  font-size: 0.875rem;
}

.invite-section {
  background: #fef3c7;
  padding: 1rem;
  border-radius: 0.5rem;
  margin-bottom: 1rem;
}

.invite-code-display {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin: 0.5rem 0;
}

.invite-code-display code {
  background: white;
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  font-size: 1.125rem;
  font-weight: bold;
  letter-spacing: 0.1em;
  border: 2px solid #f59e0b;
}

.btn-copy {
  background: #f59e0b;
  color: white;
  border: none;
  padding: 0.5rem;
  border-radius: 0.375rem;
  cursor: pointer;
  font-size: 1rem;
}

.btn-copy:hover {
  background: #d97706;
}

.invite-help {
  font-size: 0.875rem;
  color: #92400e;
  margin: 0;
}

.family-members {
  margin-bottom: 1rem;
}

.family-members h5 {
  color: #374151;
  margin-bottom: 0.5rem;
}

.member-list {
  background: #f9fafb;
  border-radius: 0.5rem;
  overflow: hidden;
}

.member-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem 1rem;
  border-bottom: 1px solid #e5e7eb;
}

.member-item:last-child {
  border-bottom: none;
}

.member-name {
  font-weight: 500;
  color: #1f2937;
}

.member-joined {
  font-size: 0.875rem;
  color: #6b7280;
}

.no-family-group {
  text-align: center;
  padding: 2rem 1rem;
}

.group-actions {
  display: flex;
  gap: 1rem;
  justify-content: center;
  margin: 1.5rem 0;
}

.form-section {
  background: #f8fafc;
  padding: 1.5rem;
  border-radius: 0.5rem;
  margin-top: 1rem;
}

.form-section h4 {
  color: #1f2937;
  margin-bottom: 1rem;
}

.form-group {
  margin-bottom: 1rem;
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

.btn-danger {
  background: #ef4444;
  color: white;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 0.375rem;
  cursor: pointer;
  font-size: 1rem;
}

.btn-danger:hover {
  background: #dc2626;
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

.no-profile, .no-user {
  text-align: center;
  color: #6b7280;
  padding: 2rem;
}

@media (max-width: 768px) {
  .group-header {
    flex-direction: column;
    gap: 0.5rem;
    text-align: center;
  }
  
  .group-actions {
    flex-direction: column;
    align-items: center;
  }
  
  .group-actions .btn {
    width: 200px;
  }
  
  .form-actions {
    flex-direction: column;
  }
  
  .form-actions .btn {
    width: 100%;
  }
  
  .member-item {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.25rem;
  }
}
</style>