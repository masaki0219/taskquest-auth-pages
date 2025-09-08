# TaskQuestFresh 認証（サインアップ/ログイン）要件定義

このドキュメントは、claduecode が本プロジェクトにおける「サインアップ/ログイン」機能を安全・一貫・拡張可能に実装できるようにするための要件定義です。既存コード（Expo + expo-router + Supabase）を前提に、実装契約・UI/UX・エラーハンドリング・検証観点を明確化します。

## 現状サマリ（2025-09 時点）
- 技術スタック: Expo（React Native）+ expo-router、Supabase JS クライアント、AsyncStorage 永続化、PKCE フロー。
- 主な構成:
  - `lib/supabaseClient.js`: Supabase クライアント生成（AsyncStorage, autoRefreshToken 有効, detectSessionInUrl: false, flowType: 'pkce'）。
  - `hooks/useAuthSupabase.js`: 認証ロジックの中核（email/pass、Google、Apple、ゲスト、状態管理）。
  - 画面: `app/(auth)/login.js`, `app/(auth)/signup.js`, `app/(auth)/forgot.js`, `app/(auth)/verify-email.js`。
  - ルーティング: 認証後は `/(tabs)/tasks` に遷移（`expo-router`）。
- Deep Link/Redirect:
  - `app.json` の scheme は `taskquestfresh`（例: `taskquestfresh://auth/callback`）。
  - `hooks/useAuthSupabase.js` は `makeRedirectUri({ scheme: 'taskquestfresh', path: 'auth/callback' })` を利用。
- 想定される改善点:
  - 表示文言に一部文字化けがあるので、最終文言は日本語で統一し直すこと。
  - Redirect URI を1箇所に集約し、Apple/Google 双方で整合を取ること。

## 実装範囲（スコープ）
- サインアップ（email/password+表示名）：入力検証、重複メールの扱い、確認メール送信、未確認時のガイダンス。
- ログイン（email/password）：入力検証、一般的なエラーの日本語化。
- OAuth ログイン（Apple/Google）：キャンセル・失敗のハンドリング、戻り後の状態確定は onAuthStateChange で行う方針を維持。
- ゲストログイン：現実装のインターフェースを踏襲（任意）。
- セッション復元：アプリ起動時にセッション確認し、認証済みならタブ画面へ遷移。
- ログアウト：セッション破棄、ゲスト時は状態のみ破棄。

## 非機能要件
- セキュリティ
  - 秘密鍵はコードに直書きしない（既に `EXPO_PUBLIC_SUPABASE_*` を使用）。
  - Deep Link/Redirect の一致（Supabase のリダイレクト先と `app.json` の `scheme` が一致）。
  - RLS（Row Level Security）が有効である前提。必要に応じて `profiles` 自動作成の SQL トリガを利用。
- ユーザー体験
  - ローディング中はボタンを無効化し、`ActivityIndicator` を表示。
  - 入力欄は `autoCapitalize`/`keyboardType` を適切に設定。パスワード表示切替を提供。
  - エラー文言は日本語で具体的に（ネットワーク・資格情報・既存ユーザーなど）。
- 可観測性/安定性
  - 余分な console 出力は避ける。必要時は `__DEV__` 時のみ。
  - 二重遷移や race condition 回避（`isAuthenticated` 変化時の重複遷移を防止）。

## 画面・フロー要件
- サインアップ
  - 入力: 表示名（必須）, メール（必須、形式チェック）, パスワード（必須、最小6文字）, 確認用パスワード（必須、一致チェック）。
  - 成功時: メール確認が必要な場合はダイアログで案内し、`/(auth)/login` に戻す。確認済なら `/(tabs)/tasks` に遷移。
  - 失敗時: 代表的なメッセージにマッピング（既存メール、弱いパスワード、ネットワーク）。
- ログイン
  - 入力: メール、パスワード（いずれも必須）。
  - 成功時: `/(tabs)/tasks` に遷移（`isAuthenticated` の変更後に実行）。
  - 失敗時: 「メールまたはパスワードが正しくありません」など一般的な文言。
- OAuth（Apple/Google）
  - 成功/失敗は戻り時に `onAuthStateChange` で確定。キャンセルはユーザーへ明示。
- 共通
  - `isLoading` に基づき全ボタンを無効化して二重送信を防止。
  - エラーは `Alert.alert` でわかりやすく提示。

## フック/API の契約（useAuthSupabase）
- 状態
  - `status`: `checking | authenticated | unauthenticated`
  - `isAuthenticated`: boolean（`status === 'authenticated'`）
  - `user`: `{ id, email, name, avatar, provider, isGuest? } | null`
- メソッドと返り値（既存実装を前提）
  - `login(email, password) -> { success, error? }`
  - `signup(name, email, password) -> { success, needsVerification?, error? }`
  - `loginWithGoogle() -> { success, error? }`
  - `loginWithApple() -> { success, error? }`
  - `loginAsGuest() -> { success, error? }`
  - `logout() -> void`

## バリデーション仕様
- メール: 簡易正規表現で形式チェック、trim 前提。
- パスワード: 最小6文字、必要なら将来ルール追加に対応できる構造。
- 表示名: 1〜50 文字程度、先頭/末尾の空白除去。

## ナビゲーション仕様
- `isAuthenticated` が `true` に変化したタイミングで `router.replace('/(tabs)/tasks')`。
- 二重遷移防止のためフラグまたは `useRef` で連打を抑制（既存 `handledRef` 流儀を維持）。

## エラーハンドリング仕様（代表例の日本語化）
- 認証情報エラー: 「メールアドレスまたはパスワードが正しくありません」。
- 既存メール: 「このメールアドレスは既に登録されています」。
- 弱いパスワード: 「パスワードは6文字以上にしてください」。
- ネットワーク: 「ネットワークエラーが発生しました」。
- OAuth キャンセル: 「キャンセルされました」。

## 外部設定/依存
- 環境変数: `EXPO_PUBLIC_SUPABASE_URL`, `EXPO_PUBLIC_SUPABASE_ANON_KEY`（`lib/supabaseClient.js` 参照）。
- Redirect/Deep Link:
  - `app.json` の `scheme` を `taskquestfresh` に設定済みであること。
  - Supabase の Auth 設定に同じ Redirect URL（例: `taskquestfresh://auth/callback`）を登録。
- Supabase 側: `profiles` 自動作成（`sql/profile_auto_creation.sql` 参照）などが有効。

## 受け入れ条件（Acceptance Criteria）
- [ ] サインアップで未確認メールの場合、確認案内を表示し login へ誘導する。
- [ ] ログイン/サインアップ中はボタン無効化とローディング表示が行われる。
- [ ] 認証後は確実に `/(tabs)/tasks` に遷移し、重複遷移は発生しない。
- [ ] 代表的なエラーが日本語で分かりやすく表示される。
- [ ] アプリ再起動時にセッションが復元される。
- [ ] Google/Apple での認証フローが端末で完結し、戻り後に状態が確定する。

## テスト観点（抜粋）
- 正常系
  - 新規登録（確認メール要/不要）、ログイン成功、OAuth 成功、ゲストログイン成功、セッション復元。
- 異常系
  - 空入力、形式不正、パスワード不一致、既存メール、資格情報エラー、ネットワークエラー、OAuth キャンセル。
- UX/遷移
  - ローディング表示とボタン無効化、二重遷移なし、エラー文言の可読性。

## 実装タスク（claduecode 向け）
1) `hooks/useAuthSupabase.js` の返却フォーマット（success/error/needsVerification）を本要件に合わせて確認・微調整。
2) `app/(auth)/login.js`, `app/(auth)/signup.js` の文言・バリデーション・ローディング制御の微修正（文字化け箇所の修正含む）。
3) Deep Link/Redirect URI の一貫性確認（`makeRedirectUri` と Supabase コンソール設定の一致）。
4) 代表エラーの日本語マッピング追加（既に存在する処理の強化）。
5) 二重遷移防止の最終チェック（`useRef` フラグ等）。

## アウトオブスコープ
- 2段階認証（MFA）、パスワード強度メータなど高度機能の導入。
- UI の大規模リデザイン（軽微な文言/動作調整のみ）。

---
補足: 既存コードは基本的に要件を満たす構成になっています。本ドキュメントは、仕上げに必要な観点（文言の統一、エッジケース、契約の明確化）を凝縮した最終ガイドです。

