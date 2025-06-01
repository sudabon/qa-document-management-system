# 静的コンテンツ出力機能 詳細設計

## 1. 機能概要

静的コンテンツ出力機能は、QAドキュメント管理システムで管理されているドキュメントをHTML/CSS/JSの静的ウェブサイトとして出力し、AWS S3にアップロードして公開するための機能です。この機能により、内部で管理しているQAドキュメントを外部向けのナレッジベースとして簡単に公開できます。

## 2. 主要機能

### 2.1 静的サイト生成

- Markdownドキュメントから静的HTMLへの変換
- カテゴリ・タグに基づいたナビゲーション構造の自動生成
- レスポンシブデザインのテーマ適用
- 検索機能の組み込み（クライアントサイド検索）
- メタデータ（SEO対応）の自動生成

### 2.2 カスタマイズ

- テーマ選択（複数のプリセットテーマ）
- ナビゲーション構造のカスタマイズ
- ロゴ・ブランディングの適用
- カスタムCSS/JSの追加
- ページレイアウトのカスタマイズ

### 2.3 AWS S3デプロイ

- S3バケット設定管理
- IAM認証情報の安全な管理
- 差分アップロード（変更されたファイルのみ）
- CloudFrontキャッシュの自動無効化
- デプロイ履歴の管理

### 2.4 プレビュー・検証

- 生成前のプレビュー機能
- リンク検証（リンク切れチェック）
- モバイル表示確認
- パフォーマンス最適化（画像圧縮、CSSミニファイなど）

### 2.5 スケジュール設定

- 定期的な自動生成・デプロイ
- 特定のイベント（ドキュメント更新など）をトリガーとした生成
- スケジュール管理と通知

## 3. 技術設計

### 3.1 静的サイト生成エンジン

静的サイト生成には、Next.jsのエクスポート機能を活用します。これにより、Reactコンポーネントを使った高品質なUIと、静的サイト生成の両方のメリットを享受できます。

#### 3.1.1 生成プロセス

1. ドキュメントデータの取得
   - データベースからドキュメント、カテゴリ、タグ情報を取得
   - 公開設定されたドキュメントのみを対象

2. Next.jsプロジェクトの構築
   - 動的にページコンポーネントを生成
   - ナビゲーション構造の構築
   - メタデータの設定

3. 静的サイトのビルド
   - `next build && next export`によるHTML/CSS/JS生成
   - アセットの最適化

4. 出力ディレクトリの整理
   - 不要なファイルの削除
   - ファイル構造の最適化

#### 3.1.2 テンプレートシステム

テンプレートシステムは、以下のコンポーネントで構成されます：

- **ベーステンプレート**: すべてのページで共通の構造（ヘッダー、フッター、サイドバーなど）
- **ページテンプレート**: 各ページタイプ（ドキュメント、カテゴリ一覧、タグ一覧など）に特化したテンプレート
- **パーシャルテンプレート**: 再利用可能な小さなコンポーネント（ナビゲーション、パンくずリスト、検索ボックスなど）

#### 3.1.3 Markdownレンダリング

Markdownのレンダリングには、以下の機能を実装します：

- シンタックスハイライト（コードブロック用）
- 数式サポート（MathJax/KaTeX）
- 図表サポート（Mermaid.js）
- カスタムコンポーネント（警告ボックス、ノートなど）
- 目次自動生成

### 3.2 AWS S3デプロイシステム

#### 3.2.1 S3バケット設定

- 静的ウェブサイトホスティング設定
- CORS設定
- パブリックアクセス設定
- バケットポリシー

#### 3.2.2 デプロイプロセス

1. AWS認証情報の取得
   - 安全に保存された認証情報の復号
   - 一時的な認証情報の取得（必要に応じて）

2. 差分検出
   - 前回デプロイ時のファイルハッシュと比較
   - 変更されたファイルのリスト作成

3. S3アップロード
   - 並列アップロード
   - メタデータ設定（Content-Type, Cache-Control）
   - エラーハンドリングとリトライ

4. CloudFront無効化
   - 変更されたパスの無効化リクエスト
   - 無効化ステータスの監視

5. デプロイ結果の検証
   - サンプルURLへのアクセス確認
   - レスポンスコードの確認

#### 3.2.3 セキュリティ対策

- 最小権限の原則に基づいたIAMポリシー
- 認証情報の安全な管理（AWS Secrets Managerの活用）
- アクセスログの有効化と監視
- バージョニングの有効化（ロールバック対応）

### 3.3 プレビューシステム

#### 3.3.1 ローカルプレビュー

- 一時的なHTTPサーバーによるプレビュー
- WebSocketを使ったリアルタイム更新
- 複数デバイス表示のシミュレーション

#### 3.3.2 検証機能

- リンク検証ツール
- HTML/CSS検証
- アクセシビリティチェック
- パフォーマンス測定

### 3.4 スケジューラ

#### 3.4.1 スケジュール設定

- cron形式による柔軟なスケジュール設定
- 一回限りのスケジュール
- 繰り返しスケジュール

#### 3.4.2 イベントトリガー

- ドキュメント更新イベント
- カテゴリ/タグ変更イベント
- 手動トリガー

#### 3.4.3 通知システム

- メール通知
- Slack/Teams通知
- システム内通知

## 4. データモデル

### 4.1 静的サイト設定

```typescript
interface StaticSiteConfig {
  id: string;
  name: string;
  description: string;
  theme: string;
  customCss?: string;
  customJs?: string;
  logo?: string;
  favicon?: string;
  navigation: NavigationItem[];
  metadata: SiteMetadata;
  s3Config: S3Config;
  cloudFrontConfig?: CloudFrontConfig;
  createdAt: Date;
  updatedAt: Date;
}

interface NavigationItem {
  id: string;
  title: string;
  path?: string;
  children?: NavigationItem[];
  order: number;
}

interface SiteMetadata {
  title: string;
  description: string;
  keywords: string[];
  author: string;
  ogImage?: string;
  twitterCard?: string;
}

interface S3Config {
  bucketName: string;
  region: string;
  accessKeyId?: string; // 暗号化して保存
  secretAccessKey?: string; // 暗号化して保存
  useRoleBasedAuth: boolean;
}

interface CloudFrontConfig {
  distributionId: string;
  domain: string;
}
```

### 4.2 デプロイ履歴

```typescript
interface DeployHistory {
  id: string;
  configId: string;
  status: 'pending' | 'in_progress' | 'completed' | 'failed';
  startedAt: Date;
  completedAt?: Date;
  fileCount: number;
  totalSize: number;
  changedFiles: string[];
  errorMessage?: string;
  triggeredBy: string; // ユーザーIDまたは'scheduler'
  s3Url: string;
  cloudFrontUrl?: string;
}
```

### 4.3 スケジュール設定

```typescript
interface DeploySchedule {
  id: string;
  configId: string;
  name: string;
  cronExpression?: string;
  oneTime?: Date;
  enabled: boolean;
  lastRun?: Date;
  nextRun?: Date;
  createdBy: string;
  createdAt: Date;
  updatedAt: Date;
}
```

## 5. コンポーネント設計

### 5.1 フロントエンドコンポーネント

#### 5.1.1 設定管理UI

```tsx
// 静的サイト設定フォーム
const StaticSiteConfigForm: React.FC<{
  initialConfig?: StaticSiteConfig;
  onSave: (config: StaticSiteConfig) => Promise<void>;
}> = ({ initialConfig, onSave }) => {
  // フォーム実装
};

// テーマ選択コンポーネント
const ThemeSelector: React.FC<{
  selectedTheme: string;
  onChange: (theme: string) => void;
}> = ({ selectedTheme, onChange }) => {
  // テーマ選択UI実装
};

// ナビゲーション編集コンポーネント
const NavigationEditor: React.FC<{
  items: NavigationItem[];
  onChange: (items: NavigationItem[]) => void;
}> = ({ items, onChange }) => {
  // ドラッグ&ドロップによるナビゲーション編集UI
};

// S3設定コンポーネント
const S3ConfigForm: React.FC<{
  config: S3Config;
  onChange: (config: S3Config) => void;
}> = ({ config, onChange }) => {
  // S3設定フォーム
};
```

#### 5.1.2 プレビュー・デプロイUI

```tsx
// プレビューコンポーネント
const SitePreview: React.FC<{
  configId: string;
}> = ({ configId }) => {
  // iframeによるプレビュー表示
};

// デプロイ履歴コンポーネント
const DeployHistory: React.FC<{
  configId: string;
}> = ({ configId }) => {
  // デプロイ履歴一覧表示
};

// デプロイアクションコンポーネント
const DeployActions: React.FC<{
  configId: string;
  onDeploy: () => Promise<void>;
}> = ({ configId, onDeploy }) => {
  // デプロイボタンとステータス表示
};
```

#### 5.1.3 スケジュール管理UI

```tsx
// スケジュール設定コンポーネント
const ScheduleForm: React.FC<{
  initialSchedule?: DeploySchedule;
  onSave: (schedule: DeploySchedule) => Promise<void>;
}> = ({ initialSchedule, onSave }) => {
  // スケジュール設定フォーム
};

// スケジュール一覧コンポーネント
const ScheduleList: React.FC<{
  configId: string;
}> = ({ configId }) => {
  // スケジュール一覧表示
};
```

### 5.2 バックエンドコンポーネント

#### 5.2.1 静的サイト生成サービス

```python
class StaticSiteGenerator:
    """静的サイト生成を担当するサービスクラス"""
    
    async def generate_site(self, config_id: str) -> str:
        """
        指定された設定に基づいて静的サイトを生成する
        
        Args:
            config_id: 静的サイト設定ID
            
        Returns:
            生成された静的サイトの一時ディレクトリパス
        """
        # 実装
        
    async def _fetch_documents(self, config_id: str) -> list[Document]:
        """公開対象のドキュメントを取得"""
        # 実装
        
    async def _setup_next_project(self, config: StaticSiteConfig, documents: list[Document]) -> str:
        """Next.jsプロジェクトのセットアップ"""
        # 実装
        
    async def _build_static_site(self, project_dir: str) -> str:
        """静的サイトのビルド実行"""
        # 実装
        
    async def _optimize_assets(self, output_dir: str) -> None:
        """アセットの最適化"""
        # 実装
```

#### 5.2.2 S3デプロイサービス

```python
class S3Deployer:
    """S3へのデプロイを担当するサービスクラス"""
    
    async def deploy(self, config_id: str, site_dir: str) -> DeployHistory:
        """
        静的サイトをS3にデプロイする
        
        Args:
            config_id: 静的サイト設定ID
            site_dir: 静的サイトのディレクトリパス
            
        Returns:
            デプロイ履歴
        """
        # 実装
        
    async def _get_aws_credentials(self, config: S3Config) -> dict:
        """AWS認証情報の取得"""
        # 実装
        
    async def _detect_changes(self, site_dir: str, bucket_name: str) -> list[str]:
        """変更されたファイルの検出"""
        # 実装
        
    async def _upload_files(self, site_dir: str, bucket_name: str, changed_files: list[str]) -> None:
        """ファイルのアップロード"""
        # 実装
        
    async def _invalidate_cloudfront(self, config: CloudFrontConfig, changed_paths: list[str]) -> str:
        """CloudFrontキャッシュの無効化"""
        # 実装
        
    async def _verify_deployment(self, config: S3Config) -> bool:
        """デプロイの検証"""
        # 実装
```

#### 5.2.3 スケジューラサービス

```python
class DeployScheduler:
    """デプロイスケジュールを管理するサービスクラス"""
    
    async def register_schedule(self, schedule: DeploySchedule) -> DeploySchedule:
        """スケジュールの登録"""
        # 実装
        
    async def update_schedule(self, schedule_id: str, schedule: DeploySchedule) -> DeploySchedule:
        """スケジュールの更新"""
        # 実装
        
    async def delete_schedule(self, schedule_id: str) -> None:
        """スケジュールの削除"""
        # 実装
        
    async def get_schedules(self, config_id: str) -> list[DeploySchedule]:
        """スケジュール一覧の取得"""
        # 実装
        
    async def execute_scheduled_deploys(self) -> list[str]:
        """スケジュールされたデプロイの実行（定期的に呼び出される）"""
        # 実装
```

## 6. API設計

### 6.1 静的サイト設定API

```
# 設定管理
GET    /api/v1/static-site/configs       # 設定一覧取得
POST   /api/v1/static-site/configs       # 設定作成
GET    /api/v1/static-site/configs/{id}  # 設定取得
PUT    /api/v1/static-site/configs/{id}  # 設定更新
DELETE /api/v1/static-site/configs/{id}  # 設定削除

# テーマ
GET    /api/v1/static-site/themes        # 利用可能なテーマ一覧取得
GET    /api/v1/static-site/themes/{id}   # テーマ詳細取得
```

### 6.2 生成・デプロイAPI

```
# 生成・デプロイ
POST   /api/v1/static-site/configs/{id}/generate  # 静的サイト生成
POST   /api/v1/static-site/configs/{id}/deploy    # S3デプロイ
POST   /api/v1/static-site/configs/{id}/preview   # プレビュー生成

# デプロイ履歴
GET    /api/v1/static-site/configs/{id}/history   # デプロイ履歴取得
GET    /api/v1/static-site/deploys/{id}           # デプロイ詳細取得
```

### 6.3 スケジュールAPI

```
# スケジュール管理
GET    /api/v1/static-site/configs/{id}/schedules    # スケジュール一覧取得
POST   /api/v1/static-site/configs/{id}/schedules    # スケジュール作成
GET    /api/v1/static-site/schedules/{id}            # スケジュール取得
PUT    /api/v1/static-site/schedules/{id}            # スケジュール更新
DELETE /api/v1/static-site/schedules/{id}            # スケジュール削除
POST   /api/v1/static-site/schedules/{id}/toggle     # スケジュール有効/無効切替
```

## 7. 実装ガイドライン

### 7.1 パフォーマンス最適化

- **並列処理**: ファイル生成・アップロードの並列化
- **差分デプロイ**: 変更されたファイルのみをアップロード
- **アセット最適化**: 画像圧縮、CSSミニファイ、JSバンドル最適化
- **キャッシュ戦略**: 適切なCache-Controlヘッダーの設定
- **遅延ロード**: 画像・コンテンツの遅延ロード

### 7.2 セキュリティ対策

- **認証情報管理**: AWS認証情報の安全な保管と使用
- **最小権限**: 必要最小限のIAM権限設定
- **入力検証**: すべてのユーザー入力の検証
- **CORS設定**: 適切なCORS設定によるセキュリティ強化
- **機密情報保護**: 環境変数や暗号化によるシークレット管理

### 7.3 エラーハンドリング

- **リトライ機構**: 一時的な障害に対するリトライ
- **ロールバック**: デプロイ失敗時の自動ロールバック
- **エラー通知**: 管理者への通知
- **詳細なログ**: トラブルシューティングのための詳細ログ
- **ユーザーフレンドリーなエラーメッセージ**: エンドユーザー向けエラー表示

### 7.4 テスト戦略

- **単体テスト**: 各コンポーネントの機能テスト
- **統合テスト**: 生成からデプロイまでの一連のフロー
- **モック**: AWS S3/CloudFrontのモック
- **E2Eテスト**: 実際のデプロイと検証
- **負荷テスト**: 大量のドキュメントがある場合のパフォーマンス

## 8. 将来の拡張性

### 8.1 追加機能候補

- **マルチ言語サポート**: 複数言語でのコンテンツ提供
- **バージョン管理**: 静的サイトのバージョン管理とロールバック
- **A/Bテスト**: 異なるデザイン・レイアウトのテスト
- **アナリティクス統合**: Google Analyticsなどの分析ツール統合
- **PWA対応**: Progressive Web Appとしての機能強化
- **コメント機能**: 静的サイトへのコメント機能追加
- **フィードバック収集**: ユーザーからのフィードバック収集機能

### 8.2 技術的拡張

- **CDNプロバイダー拡張**: AWS CloudFront以外のCDNサポート
- **ホスティングプロバイダー拡張**: S3以外のホスティングサポート
- **静的サイトジェネレーター拡張**: 他の静的サイトジェネレーターのサポート
- **テーママーケットプレイス**: カスタムテーマの共有・販売
- **プラグインシステム**: 機能拡張のためのプラグインアーキテクチャ