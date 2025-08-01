<script setup>
import { ref, onMounted } from 'vue'
import { supabase, testConnection } from '../supabase.js'

const connectionStatus = ref({ success: null, message: '' })
const user = ref(null)
const email = ref('')
const password = ref('')
const authMessage = ref('')
const testItems = ref([])
const newItemName = ref('')
const dbMessage = ref('')

onMounted(async () => {
  await checkConnection()
  await checkUser()
})

const checkConnection = async () => {
  connectionStatus.value = await testConnection()
}

const checkUser = async () => {
  const { data: { user: currentUser } } = await supabase.auth.getUser()
  user.value = currentUser
}

const signUp = async () => {
  try {
    const { data, error } = await supabase.auth.signUp({
      email: email.value,
      password: password.value,
    })
    if (error) throw error
    authMessage.value = 'サインアップ成功！確認メールをチェックしてください'
    email.value = ''
    password.value = ''
  } catch (error) {
    authMessage.value = `サインアップエラー: ${error.message}`
  }
}

const signIn = async () => {
  try {
    const { data, error } = await supabase.auth.signInWithPassword({
      email: email.value,
      password: password.value,
    })
    if (error) throw error
    authMessage.value = 'ログイン成功！'
    user.value = data.user
    email.value = ''
    password.value = ''
    await loadTestItems()
  } catch (error) {
    authMessage.value = `ログインエラー: ${error.message}`
  }
}

const signOut = async () => {
  try {
    const { error } = await supabase.auth.signOut()
    if (error) throw error
    authMessage.value = 'ログアウト成功'
    user.value = null
    testItems.value = []
  } catch (error) {
    authMessage.value = `ログアウトエラー: ${error.message}`
  }
}

const loadTestItems = async () => {
  try {
    const { data, error } = await supabase
      .from('test_items')
      .select('*')
      .order('created_at', { ascending: false })
    
    if (error) throw error
    testItems.value = data || []
    dbMessage.value = `${data?.length || 0}件のアイテムを取得しました`
  } catch (error) {
    dbMessage.value = `データ取得エラー: ${error.message}`
  }
}

const addTestItem = async () => {
  if (!newItemName.value.trim()) return
  
  try {
    const { data, error } = await supabase
      .from('test_items')
      .insert([{ name: newItemName.value, user_id: user.value.id }])
      .select()
    
    if (error) throw error
    newItemName.value = ''
    await loadTestItems()
    dbMessage.value = 'アイテムを追加しました'
  } catch (error) {
    dbMessage.value = `追加エラー: ${error.message}`
  }
}

const deleteTestItem = async (id) => {
  try {
    const { error } = await supabase
      .from('test_items')
      .delete()
      .eq('id', id)
    
    if (error) throw error
    await loadTestItems()
    dbMessage.value = 'アイテムを削除しました'
  } catch (error) {
    dbMessage.value = `削除エラー: ${error.message}`
  }
}
</script>

<template>
  <div class="supabase-test">
    <h2>Supabase接続テスト</h2>
    
    <!-- 接続テスト -->
    <div class="test-section">
      <h3>1. 接続確認</h3>
      <div :class="['status', connectionStatus.success ? 'success' : 'error']">
        {{ connectionStatus.message }}
      </div>
      <button @click="checkConnection" class="btn btn-secondary">再接続テスト</button>
    </div>

    <!-- 認証テスト -->
    <div class="test-section">
      <h3>2. 認証テスト</h3>
      <div v-if="!user" class="auth-form">
        <input v-model="email" type="email" placeholder="メールアドレス" class="input" />
        <input v-model="password" type="password" placeholder="パスワード" class="input" />
        <div class="button-group">
          <button @click="signUp" class="btn btn-primary">サインアップ</button>
          <button @click="signIn" class="btn btn-primary">ログイン</button>
        </div>
      </div>
      <div v-else class="user-info">
        <p>ログイン中: {{ user.email }}</p>
        <button @click="signOut" class="btn btn-secondary">ログアウト</button>
      </div>
      <div v-if="authMessage" class="message">{{ authMessage }}</div>
    </div>

    <!-- データベーステスト -->
    <div v-if="user" class="test-section">
      <h3>3. データベーステスト</h3>
      <div class="db-form">
        <input v-model="newItemName" placeholder="テストアイテム名" class="input" />
        <button @click="addTestItem" class="btn btn-primary">追加</button>
        <button @click="loadTestItems" class="btn btn-secondary">読み込み</button>
      </div>
      <div v-if="dbMessage" class="message">{{ dbMessage }}</div>
      <div v-if="testItems.length > 0" class="item-list">
        <h4>テストアイテム一覧:</h4>
        <div v-for="item in testItems" :key="item.id" class="item">
          <span>{{ item.name }}</span>
          <button @click="deleteTestItem(item.id)" class="btn-delete">削除</button>
        </div>
      </div>
    </div>

    <!-- 説明 -->
    <div class="instructions">
      <h3>テスト手順</h3>
      <ol>
        <li>接続確認で「Supabase接続成功」と表示されることを確認</li>
        <li>メールアドレスとパスワードでサインアップ（確認メール要）</li>
        <li>同じ認証情報でログイン</li>
        <li>テストアイテムの追加・表示・削除を確認</li>
      </ol>
    </div>
  </div>
</template>

<style scoped>
.supabase-test {
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
}

.test-section {
  margin-bottom: 2rem;
  padding: 1rem;
  border: 1px solid #e2e8f0;
  border-radius: 0.5rem;
  background: white;
}

.status.success {
  color: #10b981;
  font-weight: bold;
}

.status.error {
  color: #ef4444;
  font-weight: bold;
}

.auth-form, .db-form {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.input {
  padding: 0.5rem;
  border: 1px solid #d1d5db;
  border-radius: 0.375rem;
  flex: 1;
  min-width: 200px;
}

.button-group {
  display: flex;
  gap: 0.5rem;
}

.user-info {
  padding: 1rem;
  background: #f0f9ff;
  border-radius: 0.375rem;
  margin-bottom: 1rem;
}

.message {
  padding: 0.5rem;
  margin-top: 0.5rem;
  border-radius: 0.375rem;
  background: #f3f4f6;
}

.item-list {
  margin-top: 1rem;
}

.item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem;
  border-bottom: 1px solid #e5e7eb;
}

.btn-delete {
  background: #ef4444;
  color: white;
  border: none;
  padding: 0.25rem 0.5rem;
  border-radius: 0.25rem;
  cursor: pointer;
  font-size: 0.875rem;
}

.btn-delete:hover {
  background: #dc2626;
}

.instructions {
  margin-top: 2rem;
  padding: 1rem;
  background: #fef3c7;
  border-radius: 0.5rem;
}

.instructions ol {
  margin-left: 1.5rem;
  margin-top: 0.5rem;
}

@media (max-width: 768px) {
  .auth-form, .db-form {
    flex-direction: column;
  }
  
  .button-group {
    flex-direction: column;
  }
  
  .item {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.5rem;
  }
}
</style>