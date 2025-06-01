# RAG検索機能 詳細設計（続き）

## 4. データモデル

### 4.1 基本データモデル

```typescript
// ドキュメントチャンク
interface DocumentChunk {
  id: string;
  document_id: string;
  content: string;
  metadata: {
    title: string;
    category: string;
    tags: string[];
    created_at: Date;
    updated_at: Date;
    author: string;
  };
  chunk_index: number;
}

// エンベディング付きドキュメントチャンク
interface DocumentChunkWithEmbedding extends DocumentChunk {
  embedding: number[];
}

// 検索クエリ
interface Query {
  id: string;
  original_query: string;
  normalized_query: string;
  user_id?: string;
  created_at: Date;
  filters?: {
    categories?: string[];
    tags?: string[];
    date_range?: {
      start?: Date;
      end?: Date;
    };
    authors?: string[];
  };
}

// エンベディング付き検索クエリ
interface QueryWithEmbedding extends Query {
  embedding: number[];
}

// 検索結果
interface SearchResult {
  chunk_id: string;
  document_id: string;
  content: string;
  metadata: {
    title: string;
    category: string;
    tags: string[];
    created_at: Date;
    updated_at: Date;
    author: string;
  };
  score: number;
}

// 生成された回答
interface Answer {
  id: string;
  query_id: string;
  text: string;
  created_at: Date;
}

// 強化された回答（ソース情報付き）
interface EnhancedAnswer extends Answer {
  sources: {
    document_id: string;
    title: string;
    static_link?: string;
  }[];
  confidence: number;
}

// フィードバックデータ
interface FeedbackData {
  rating: number; // 1-5
  comment?: string;
}

// 検索フィードバック
interface SearchFeedback {
  id: string;
  query_id: string;
  rating: number;
  comment?: string;
  created_at: Date;
}

// フィードバック分析期間
interface TimePeriod {
  start_date: Date;
  end_date: Date;
}

// フィードバック分析結果
interface FeedbackAnalysis {
  period: TimePeriod;
  feedback_count: number;
  average_rating: number;
  low_rated_queries: Query[];
  common_keywords: {
    keyword: string;
    count: number;
  }[];
}
```

### 4.2 Pydanticモデル

```python
from pydantic import BaseModel, Field
from typing import List, Dict, Optional
from datetime import datetime
import uuid

class DocumentMetadata(BaseModel):
    """ドキュメントのメタデータ"""
    title: str
    category: str
    tags: List[str] = Field(default_factory=list)
    created_at: datetime
    updated_at: datetime
    author: str

class DocumentChunk(BaseModel):
    """ドキュメントチャンク"""
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    document_id: str
    content: str
    metadata: DocumentMetadata
    chunk_index: int

class DocumentChunkWithEmbedding(DocumentChunk):
    """エンベディング付きドキュメントチャンク"""
    embedding: List[float]

class DateRange(BaseModel):
    """日付範囲"""
    start: Optional[datetime] = None
    end: Optional[datetime] = None

class QueryFilters(BaseModel):
    """検索フィルター"""
    categories: Optional[List[str]] = None
    tags: Optional[List[str]] = None
    date_range: Optional[DateRange] = None
    authors: Optional[List[str]] = None

class Query(BaseModel):
    """検索クエリ"""
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    original_query: str
    normalized_query: str
    user_id: Optional[str] = None
    created_at: datetime = Field(default_factory=datetime.now)
    filters: QueryFilters = Field(default_factory=QueryFilters)

class QueryWithEmbedding(Query):
    """エンベディング付き検索クエリ"""
    embedding: List[float]

class SearchResult(BaseModel):
    """検索結果"""
    chunk_id: str
    document_id: str
    content: str
    metadata: DocumentMetadata
    score: float

class DocumentSource(BaseModel):
    """回答のソース情報"""
    document_id: str
    title: str
    static_link: Optional[str] = None

class Answer(BaseModel):
    """生成された回答"""
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    query_id: str
    text: str
    created_at: datetime = Field(default_factory=datetime.now)

class EnhancedAnswer(Answer):
    """強化された回答（ソース情報付き）"""
    sources: List[DocumentSource]
    confidence: float

class FeedbackData(BaseModel):
    """フィードバックデータ"""
    rating: int = Field(ge=1, le=5)
    comment: Optional[str] = None

class SearchFeedback(BaseModel):
    """検索フィードバック"""
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    query_id: str
    rating: int = Field(ge=1, le=5)
    comment: Optional[str] = None
    created_at: datetime = Field(default_factory=datetime.now)

class TimePeriod(BaseModel):
    """時間期間"""
    start_date: datetime
    end_date: datetime

class KeywordCount(BaseModel):
    """キーワード出現回数"""
    keyword: str
    count: int

class FeedbackAnalysis(BaseModel):
    """フィードバック分析結果"""
    period: TimePeriod
    feedback_count: int
    average_rating: float
    low_rated_queries: List[Query]
    common_keywords: List[KeywordCount]
```

## 5. API設計

### 5.1 RESTful API

```
# ドキュメント検索API
POST   /api/v1/search                    # セマンティック検索
POST   /api/v1/search/rag                # RAG検索（質問応答）
GET    /api/v1/search/history            # 検索履歴取得
GET    /api/v1/search/history/{id}       # 特定の検索結果取得

# フィードバックAPI
POST   /api/v1/search/feedback           # フィードバック送信
GET    /api/v1/search/feedback/stats     # フィードバック統計取得

# 管理API
POST   /api/v1/admin/reindex             # ドキュメントの再インデックス
GET    /api/v1/admin/index/status        # インデックスステータス取得
POST   /api/v1/admin/embeddings/update   # エンベディングモデル更新
```

### 5.2 API仕様

#### 5.2.1 セマンティック検索API

**リクエスト**

```json
POST /api/v1/search
{
  "query": "クラウドストレージのセキュリティ対策について教えてください",
  "filters": {
    "categories": ["セキュリティ", "クラウド"],
    "tags": ["AWS", "セキュリティ対策"],
    "date_range": {
      "start": "2024-01-01T00:00:00Z"
    },
    "authors": ["security_team"]
  },
  "limit": 5
}
```

**レスポンス**

```json
{
  "query_id": "550e8400-e29b-41d4-a716-446655440000",
  "results": [
    {
      "document_id": "doc-123",
      "title": "AWSクラウドストレージのセキュリティベストプラクティス",
      "snippet": "AWSのS3バケットに対するセキュリティ対策として、以下の点が重要です...",
      "score": 0.92,
      "metadata": {
        "category": "セキュリティ",
        "tags": ["AWS", "S3", "セキュリティ対策"],
        "created_at": "2024-02-15T10:30:00Z",
        "updated_at": "2024-05-20T14:45:00Z",
        "author": "security_team"
      },
      "static_link": "https://example.com/docs/security/aws-storage-security"
    },
    // 他の検索結果...
  ],
  "total_count": 15,
  "execution_time_ms": 120
}
```

#### 5.2.2 RAG検索API（質問応答）

**リクエスト**

```json
POST /api/v1/search/rag
{
  "question": "AWSのS3バケットで暗号化を設定する方法を教えてください",
  "filters": {
    "categories": ["セキュリティ", "クラウド"]
  },
  "include_sources": true
}
```

**レスポンス**

```json
{
  "query_id": "550e8400-e29b-41d4-a716-446655440001",
  "answer": {
    "text": "AWSのS3バケットで暗号化を設定するには、以下の方法があります：\n\n1. サーバーサイド暗号化（SSE）：\n   - SSE-S3: AWSが管理する鍵を使用\n   - SSE-KMS: AWS KMSで管理する鍵を使用\n   - SSE-C: お客様が提供する鍵を使用\n\n2. バケットポリシーで暗号化を強制：\n```json\n{\n  \"Version\": \"2012-10-17\",\n  \"Statement\": [\n    {\n      \"Effect\": \"Deny\",\n      \"Principal\": \"*\",\n      \"Action\": \"s3:PutObject\",\n      \"Resource\": \"arn:aws:s3:::your-bucket-name/*\",\n      \"Condition\": {\n        \"StringNotEquals\": {\n          \"s3:x-amz-server-side-encryption\": \"AES256\"\n        }\n      }\n    }\n  ]\n}\n```\n\n3. デフォルト暗号化の設定：\n   S3バケットのプロパティから「デフォルト暗号化」を有効にし、暗号化タイプを選択します。\n\n詳細な手順は[Document 1]のAWS公式ドキュメントを参照してください。",
    "confidence": 0.95,
    "sources": [
      {
        "document_id": "doc-456",
        "title": "AWS S3暗号化ガイド",
        "static_link": "https://example.com/docs/security/s3-encryption-guide"
      },
      {
        "document_id": "doc-789",
        "title": "クラウドストレージセキュリティベストプラクティス",
        "static_link": "https://example.com/docs/security/cloud-storage-best-practices"
      }
    ]
  },
  "execution_time_ms": 850
}
```

#### 5.2.3 フィードバックAPI

**リクエスト**

```json
POST /api/v1/search/feedback
{
  "query_id": "550e8400-e29b-41d4-a716-446655440001",
  "rating": 4,
  "comment": "回答は役立ちましたが、もう少し具体的な手順があると良かったです"
}
```

**レスポンス**

```json
{
  "success": true,
  "feedback_id": "550e8400-e29b-41d4-a716-446655440002",
  "message": "フィードバックを受け付けました。ありがとうございます。"
}
```

## 6. コンポーネント設計

### 6.1 フロントエンドコンポーネント

#### 6.1.1 検索インターフェース

```tsx
// 検索フォームコンポーネント
const SearchForm: React.FC<{
  onSearch: (query: string, filters: SearchFilters) => Promise<void>;
}> = ({ onSearch }) => {
  // 検索フォーム実装
};

// フィルターコンポーネント
const SearchFilters: React.FC<{
  filters: SearchFilters;
  onChange: (filters: SearchFilters) => void;
}> = ({ filters, onChange }) => {
  // フィルターUI実装
};

// 検索結果リストコンポーネント
const SearchResults: React.FC<{
  results: SearchResult[];
  onResultClick: (result: SearchResult) => void;
}> = ({ results, onResultClick }) => {
  // 検索結果リスト表示
};
```

#### 6.1.2 質問応答インターフェース

```tsx
// 質問入力コンポーネント
const QuestionInput: React.FC<{
  onSubmit: (question: string, filters: SearchFilters) => Promise<void>;
}> = ({ onSubmit }) => {
  // 質問入力フォーム実装
};

// 回答表示コンポーネント
const AnswerDisplay: React.FC<{
  answer: EnhancedAnswer;
  onSourceClick: (source: DocumentSource) => void;
}> = ({ answer, onSourceClick }) => {
  // 回答表示実装
};

// ソース参照コンポーネント
const SourceReferences: React.FC<{
  sources: DocumentSource[];
  onSourceClick: (source: DocumentSource) => void;
}> = ({ sources, onSourceClick }) => {
  // ソース参照リスト表示
};
```

#### 6.1.3 フィードバックコンポーネント

```tsx
// フィードバックフォームコンポーネント
const FeedbackForm: React.FC<{
  queryId: string;
  onSubmit: (feedback: FeedbackData) => Promise<void>;
}> = ({ queryId, onSubmit }) => {
  // フィードバックフォーム実装
};

// 評価スターコンポーネント
const RatingStars: React.FC<{
  value: number;
  onChange: (value: number) => void;
}> = ({ value, onChange }) => {
  // 評価スターUI実装
};
```

### 6.2 バックエンドコンポーネント

#### 6.2.1 検索サービス

```python
class SearchService:
    """検索機能を提供するサービスクラス"""
    
    def __init__(self, 
                vector_db_client: VectorDBClient,
                embedding_service: EmbeddingService,
                document_repository: DocumentRepository):
        self.vector_db_client = vector_db_client
        self.embedding_service = embedding_service
        self.document_repository = document_repository
    
    async def semantic_search(self, 
                             query: str, 
                             filters: Optional[dict] = None, 
                             limit: int = 5) -> tuple[str, list[SearchResult]]:
        """
        セマンティック検索を実行する
        
        Args:
            query: 検索クエリ
            filters: 検索フィルター
            limit: 結果の最大数
            
        Returns:
            クエリIDと検索結果のリスト
        """
        # クエリ処理
        query_with_embedding = await self.embedding_service.process_query(query, filters)
        
        # クエリの保存
        query_id = await self._save_query(query_with_embedding)
        
        # ベクトル検索
        search_results = await self.vector_db_client.search(
            query_embedding=query_with_embedding.embedding,
            filters=query_with_embedding.filters,
            limit=limit
        )
        
        # 結果のランキング
        ranked_results = await self._rank_results(search_results, query_with_embedding)
        
        # 静的サイトリンクの追加
        enhanced_results = await self._add_static_links(ranked_results)
        
        return query_id, enhanced_results
    
    async def _save_query(self, query: QueryWithEmbedding) -> str:
        """クエリを保存してIDを返す"""
        # 実装
        
    async def _rank_results(self, 
                           results: list[SearchResult], 
                           query: QueryWithEmbedding) -> list[SearchResult]:
        """検索結果をランキングする"""
        # 実装
        
    async def _add_static_links(self, results: list[SearchResult]) -> list[SearchResult]:
        """検索結果に静的サイトリンクを追加する"""
        # 実装
```

#### 6.2.2 RAG検索サービス

```python
class RAGService:
    """RAG検索（質問応答）機能を提供するサービスクラス"""
    
    def __init__(self,
                search_service: SearchService,
                llm_service: LLMService,
                answer_repository: AnswerRepository):
        self.search_service = search_service
        self.llm_service = llm_service
        self.answer_repository = answer_repository
    
    async def answer_question(self, 
                             question: str, 
                             filters: Optional[dict] = None,
                             include_sources: bool = True) -> tuple[str, EnhancedAnswer]:
        """
        質問に回答する
        
        Args:
            question: ユーザーの質問
            filters: 検索フィルター
            include_sources: ソース情報を含めるかどうか
            
        Returns:
            クエリIDと強化された回答
        """
        # 関連ドキュメントの検索
        query_id, search_results = await self.search_service.semantic_search(
            query=question,
            filters=filters,
            limit=5
        )
        
        # プロンプトの構築
        prompt = self.llm_service.build_prompt(question, search_results)
        
        # 回答の生成
        raw_answer = await self.llm_service.generate_answer(prompt)
        
        # 回答の後処理
        enhanced_answer = await self.llm_service.postprocess_answer(
            answer=raw_answer,
            search_results=search_results,
            include_sources=include_sources
        )
        
        # 回答の保存
        answer_id = await self._save_answer(query_id, enhanced_answer)
        
        return query_id, enhanced_answer
    
    async def _save_answer(self, query_id: str, answer: EnhancedAnswer) -> str:
        """回答を保存してIDを返す"""
        # 実装
```

#### 6.2.3 フィードバックサービス

```python
class FeedbackService:
    """フィードバック機能を提供するサービスクラス"""
    
    def __init__(self, feedback_repository: FeedbackRepository):
        self.feedback_repository = feedback_repository
    
    async def save_feedback(self, 
                           query_id: str, 
                           feedback_data: FeedbackData) -> str:
        """
        フィードバックを保存する
        
        Args:
            query_id: 検索クエリのID
            feedback_data: フィードバックデータ
            
        Returns:
            フィードバックID
        """
        feedback = SearchFeedback(
            query_id=query_id,
            rating=feedback_data.rating,
            comment=feedback_data.comment
        )
        
        feedback_id = await self.feedback_repository.save(feedback)
        
        # 低評価の場合は通知
        if feedback.rating <= 2:
            await self._notify_low_rating(query_id, feedback)
        
        return feedback_id
    
    async def get_feedback_stats(self, 
                                period: Optional[TimePeriod] = None) -> FeedbackAnalysis:
        """
        フィードバック統計を取得する
        
        Args:
            period: 分析対象の期間
            
        Returns:
            フィードバック分析結果
        """
        # デフォルトは過去30日
        if period is None:
            end_date = datetime.now()
            start_date = end_date - timedelta(days=30)
            period = TimePeriod(start_date=start_date, end_date=end_date)
        
        return await self.feedback_repository.analyze(period)
    
    async def _notify_low_rating(self, query_id: str, feedback: SearchFeedback) -> None:
        """低評価フィードバックの通知"""
        # 実装
```

## 7. 実装ガイドライン

### 7.1 パフォーマンス最適化

- **バッチ処理**: ドキュメントのエンベディング生成やベクトルDBへの保存は、バッチ処理で効率化
- **キャッシュ戦略**: 頻出クエリや回答のキャッシュによる応答時間短縮
- **非同期処理**: 検索と回答生成の並列化
- **インデックス最適化**: ベクトルDBのインデックス設計の最適化
- **クエリ最適化**: 効率的なフィルタリングとベクトル検索の組み合わせ

### 7.2 エンベディングモデル選択

- **モデル選択基準**:
  - 精度（類似度計算の質）
  - 速度（エンベディング生成時間）
  - 次元数（ストレージとクエリ効率）
  - 多言語サポート
  - コスト

- **推奨モデル**:
  - OpenAI Ada Embeddings: 高品質だが外部API依存
  - Sentence Transformers: オープンソースで自己ホスト可能
  - BERT/RoBERTa: 特定ドメイン向けにファインチューニング可能

### 7.3 LLMプロンプトエンジニアリング

- **効果的なプロンプト設計**:
  - 明確な指示と制約の提供
  - 関連コンテキストの適切な提供
  - 回答フォーマットの指定
  - 引用方法の明示

- **プロンプトテンプレート例**:
```
あなたは専門的なQAシステムのアシスタントです。
以下の情報源に基づいて、ユーザーの質問に正確に答えてください。

情報源:
{context}

質問: {question}

回答の際は以下のルールに従ってください:
1. 情報源に含まれる情報のみを使用する
2. 情報源に情報がない場合は「その情報は提供されたドキュメントには含まれていません」と答える
3. 回答の根拠となる情報源を引用する（例: [Document 1]によると...）
4. 簡潔かつ明確に答える
5. 回答の最後に関連する静的ドキュメントへのリンクを提案する

回答:
```

### 7.4 エラーハンドリング

- **LLM API障害対策**:
  - タイムアウト設定
  - リトライ機構
  - フォールバックモデル

- **ベクトルDB障害対策**:
  - 読み取り専用レプリカの活用
  - キャッシュによる一時的な代替
  - 段階的デグラデーション

- **エラー通知と監視**:
  - 重要なエラーの即時通知
  - パフォーマンス指標の監視
  - 異常検知

### 7.5 セキュリティ対策

- **ユーザー入力のサニタイズ**:
  - インジェクション攻撃対策
  - 悪意あるプロンプトの検出

- **APIキー管理**:
  - LLM APIキーの安全な保管
  - 最小権限の原則

- **データ保護**:
  - 機密情報のフィルタリング
  - PII（個人識別情報）の検出と処理

### 7.6 テスト戦略

- **単体テスト**:
  - 各コンポーネントの機能テスト
  - モックを使用したLLM/ベクトルDBのテスト

- **統合テスト**:
  - エンドツーエンドのRAGフロー
  - 異なる質問タイプでの応答品質

- **評価メトリクス**:
  - 正確性（人間評価）
  - 関連性（検索結果の適合性）
  - 応答時間
  - トークン使用量

## 8. 将来の拡張性

### 8.1 追加機能候補

- **マルチモーダル検索**: 画像や図表を含むドキュメントの検索と理解
- **会話コンテキスト**: 過去の質問と回答を考慮した会話的検索
- **パーソナライズ**: ユーザーの過去の検索履歴や行動に基づく検索結果のパーソナライズ
- **説明可能性**: 検索結果や回答の理由付けの提供
- **アクティブラーニング**: ユーザーフィードバックを活用した継続的な改善

### 8.2 技術的拡張

- **複数LLMサポート**: 異なるLLMプロバイダーやモデルの統合
- **ローカルLLM**: オンプレミスでのLLM実行オプション
- **ハイブリッドRAG**: キーワード検索とセマンティック検索の組み合わせ
- **クエリリライト**: ユーザークエリの自動リファインメント
- **ファインチューニング**: 特定ドメイン向けのエンベディングモデルのカスタマイズ

### 8.3 スケーラビリティ拡張

- **分散ベクトルDB**: 大規模ドキュメントコレクション対応
- **シャーディング**: 地理的または機能的なシャーディング
- **バッチ処理パイプライン**: 大量ドキュメント処理の効率化
- **リアルタイム更新**: ドキュメント更新時の即時インデックス更新