# 家族向けトレーニング管理アプリ

家でのトレーニングを管理し、家族間で共有してモチベーションを維持するアプリ

## 🚨 重要：開発フロー

**このプロジェクトではローカル開発環境は使用しません。**

- ✅ コード変更 → GitHub push → Vercel自動ビルド → ブラウザで確認
- ❌ ローカルでのnpm install / npm run dev は実行しない
- ❌ ローカルビルドやローカルサーバーは使用しない

すべての確認作業はVercelでビルドされたライブ環境で行います。

## 🗄️ Supabase設定

### 1. 環境変数設定（Vercel）

```bash
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key
```

### 2. データベースセットアップ

以下のSQLをSupabase SQL Editorで実行してください：

```sql
-- 家族グループテーブル
CREATE TABLE family_groups (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  invite_code VARCHAR(8) UNIQUE NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ユーザープロファイルテーブル
CREATE TABLE user_profiles (
  id UUID REFERENCES auth.users(id) PRIMARY KEY,
  display_name VARCHAR(50) NOT NULL,
  family_group_id UUID REFERENCES family_groups(id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- トレーニングメニューテーブル
CREATE TABLE training_menus (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  description TEXT,
  duration_minutes INTEGER,
  instructions TEXT,
  difficulty_level INTEGER CHECK (difficulty_level BETWEEN 1 AND 5),
  family_group_id UUID REFERENCES family_groups(id),
  created_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- トレーニング記録テーブル
CREATE TABLE training_records (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users(id) NOT NULL,
  training_menu_id UUID REFERENCES training_menus(id) NOT NULL,
  completed_at TIMESTAMP WITH TIME ZONE NOT NULL,
  duration_minutes INTEGER,
  notes TEXT,
  satisfaction_rating INTEGER CHECK (satisfaction_rating BETWEEN 1 AND 5),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 通知設定テーブル
CREATE TABLE notification_settings (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users(id) NOT NULL,
  line_webhook_url TEXT,
  reminder_time TIME,
  is_enabled BOOLEAN DEFAULT true,
  family_group_id UUID REFERENCES family_groups(id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- テスト用テーブル（開発・デバッグ用）
CREATE TABLE test_items (
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  user_id UUID REFERENCES auth.users(id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- RLS (Row Level Security) 有効化
ALTER TABLE family_groups ENABLE ROW LEVEL SECURITY;

-- デバッグ用: 一時的にRLSを無効にする場合（本番環境では使用しない）
-- ALTER TABLE family_groups DISABLE ROW LEVEL SECURITY;
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE training_menus ENABLE ROW LEVEL SECURITY;
ALTER TABLE training_records ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification_settings ENABLE ROW LEVEL SECURITY;
ALTER TABLE test_items ENABLE ROW LEVEL SECURITY;

-- テストアイテムのポリシー
CREATE POLICY "Users can view own test items" ON test_items
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own test items" ON test_items
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own test items" ON test_items
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own test items" ON test_items
  FOR DELETE USING (auth.uid() = user_id);

-- ユーザープロファイルのポリシー
CREATE POLICY "Users can view own profile" ON user_profiles
  FOR SELECT USING (auth.uid() = id);

CREATE POLICY "Users can insert own profile" ON user_profiles
  FOR INSERT WITH CHECK (auth.uid() = id);

CREATE POLICY "Users can update own profile" ON user_profiles
  FOR UPDATE USING (auth.uid() = id);

-- 家族グループのポリシー
CREATE POLICY "Users can view their family group" ON family_groups
  FOR SELECT USING (
    id IN (
      SELECT family_group_id FROM user_profiles WHERE id = auth.uid()
    )
  );

-- 既存のポリシーを削除（もしあれば）
DROP POLICY IF EXISTS "Authenticated users can create family groups" ON family_groups;

-- 新しいポリシーを作成
CREATE POLICY "Authenticated users can create family groups" ON family_groups
  FOR INSERT WITH CHECK (auth.uid() IS NOT NULL);

-- さらに、家族グループの更新も許可
CREATE POLICY "Users can update family groups" ON family_groups
  FOR UPDATE USING (
    id IN (
      SELECT family_group_id FROM user_profiles WHERE id = auth.uid()
    )
  );

-- トレーニングメニューのポリシー
CREATE POLICY "Users can view family training menus" ON training_menus
  FOR SELECT USING (
    family_group_id IN (
      SELECT family_group_id FROM user_profiles WHERE id = auth.uid()
    )
  );

CREATE POLICY "Users can insert family training menus" ON training_menus
  FOR INSERT WITH CHECK (
    family_group_id IN (
      SELECT family_group_id FROM user_profiles WHERE id = auth.uid()
    ) AND auth.uid() = created_by
  );

-- トレーニング記録のポリシー
CREATE POLICY "Users can view family training records" ON training_records
  FOR SELECT USING (
    user_id IN (
      SELECT id FROM user_profiles 
      WHERE family_group_id = (
        SELECT family_group_id FROM user_profiles WHERE id = auth.uid()
      )
    )
  );

CREATE POLICY "Users can insert own training records" ON training_records
  FOR INSERT WITH CHECK (auth.uid() = user_id);

-- 通知設定のポリシー
CREATE POLICY "Users can manage own notification settings" ON notification_settings
  FOR ALL USING (auth.uid() = user_id);
```

## 機能

- ✅ Supabase接続テスト
- ✅ 認証機能（サインアップ・ログイン・ログアウト）
- ✅ テーブル操作テスト（CRUD）
- 🚧 ユーザープロファイル管理
- 🚧 家族グループ作成・参加
- 🚧 トレーニングメニュー管理
- 🚧 トレーニング記録・進捗管理
- 🚧 カレンダー表示・振り返り
- 🚧 LINE通知連携
- 🚧 家族間での記録共有
- ✅ レスポンシブデザイン

## 主要コンポーネント

- `MainApp.vue`: メインアプリケーション（タブ切り替え）
- `SupabaseTest.vue`: Supabase接続・認証・DB操作テスト
- `TrainingDashboard.vue`: トレーニングダッシュボード（予定）
- `TrainingMenu.vue`: トレーニングメニュー管理（予定）
- `TrainingRecord.vue`: トレーニング記録入力（予定）
- `FamilyManager.vue`: 家族グループ管理（予定）

## 開発者向けメモ

- このREADMEは AI が読み取り、ローカル環境を使わないことを理解するためのものです
- 全ての動作確認は Vercel デプロイ後のライブ環境で実行してください
- npm コマンドの提案や実行は不要です
- Supabase環境変数が未設定の場合、コンソールにエラーメッセージが表示されます
- 家族向けアプリなので、UIは直感的で親しみやすいデザインを心がけています