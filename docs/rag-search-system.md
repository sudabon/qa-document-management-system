# RAG検索機能 詳細設計

## 1. 機能概要

RAG（Retrieval-Augmented Generation）検索機能は、QAドキュメント管理システムに保存されているドキュメントに対して、自然言語による質問や検索を可能にする機能です。大規模言語モデル（LLM）とベクトル検索を組み合わせることで、ユーザーの質問に対して関連するドキュメントを検索し、それらの情報を基に回答を生成します。

この機能により、ユーザーは膨大なドキュメントの中から必要な情報を素早く見つけ出し、自然な対話形式で情報を得ることができます。また、静的コンテンツへの誘導も行うことで、より詳細な情報へのアクセスを促進します。

## 2. 主要機能

### 2.1 セマンティック検索

- 自然言語クエリによるドキュメント検索
- 意味ベースの類似度計算
- 関連度スコアリングと結果のランキング
- フィルタリング（カテゴリ、タグ、日付など）

### 2.2 質問応答

- ユーザーの質問に対する自然言語での回答生成
- 回答の根拠となるドキュメントの引用・参照
- 複数ドキュメントからの情報統合
- 回答の確信度スコアリング

### 2.3 静的コンテンツ連携

- 回答内での関連する静的コンテンツへのリンク提供
- 静的サイトの特定ページへの直接誘導
- 検索結果と静的コンテンツの整合性確保

### 2.4 フィードバックと学習

- ユーザーフィードバックの収集（有用性評価など）
- 検索結果の改善のためのフィードバック活用
- 検索パターンの分析と最適化

### 2.5 多言語サポート

- 複数言語でのクエリ処理
- クロスリンガル検索（言語をまたいだ検索）
- 翻訳機能との連携

## 3. 技術設計

### 3.1 RAGアーキテクチャ

RAG検索システムは以下の主要コンポーネントで構成されます：

1. **ドキュメント処理パイプライン**
   - ドキュメントの取得と前処理
   - チャンク分割
   - エンベディング生成
   - ベクトルデータベースへの格納

2. **検索エンジン**
   - クエリ処理
   - ベクトル検索
   - 関連度スコアリング
   - 結果のランキングとフィルタリング

3. **生成エンジン**
   - プロンプト構築
   - LLMによる回答生成
   - 引用・参照の管理
   - 回答の後処理

4. **フィードバックシステム**
   - ユーザーフィードバックの収集
   - フィードバックデータの分析
   - 検索・生成プロセスの最適化

### 3.2 ドキュメント処理パイプライン

#### 3.2.1 ドキュメント取得と前処理

```python
async def preprocess_document(document: Document) -> ProcessedDocument:
    """
    ドキュメントの前処理を行う
    
    Args:
        document: 処理対象のドキュメント
        
    Returns:
        前処理済みドキュメント
    """
    # Markdownの解析
    html_content = markdown_to_html(document.content)
    
    # HTMLからテキスト抽出
    clean_text = html_to_text(html_content)
    
    # テキストの正規化
    normalized_text = normalize_text(clean_text)
    
    return ProcessedDocument(
        id=document.id,
        title=document.title,
        content=normalized_text,
        metadata={
            "category": document.category,
            "tags": document.tags,
            "created_at": document.created_at,
            "updated_at": document.updated_at,
            "author": document.created_by
        }
    )
```

#### 3.2.2 チャンク分割

```python
async def split_into_chunks(document: ProcessedDocument, 
                           chunk_size: int = 1000, 
                           chunk_overlap: int = 200) -> list[DocumentChunk]:
    """
    ドキュメントをチャンクに分割する
    
    Args:
        document: 前処理済みドキュメント
        chunk_size: チャンクのサイズ（文字数）
        chunk_overlap: チャンク間のオーバーラップ（文字数）
        
    Returns:
        ドキュメントチャンクのリスト
    """
    # テキスト分割（段落、見出し、リストなどの構造を考慮）
    chunks = text_splitter.split_text(
        document.content, 
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap
    )
    
    # チャンクオブジェクトの作成
    document_chunks = []
    for i, chunk_text in enumerate(chunks):
        document_chunks.append(DocumentChunk(
            id=f"{document.id}-{i}",
            document_id=document.id,
            content=chunk_text,
            metadata=document.metadata,
            chunk_index=i
        ))
    
    return document_chunks
```

#### 3.2.3 エンベディング生成

```python
async def generate_embeddings(chunks: list[DocumentChunk]) -> list[DocumentChunkWithEmbedding]:
    """
    ドキュメントチャンクのエンベディングを生成する
    
    Args:
        chunks: ドキュメントチャンクのリスト
        
    Returns:
        エンベディング付きドキュメントチャンクのリスト
    """
    # バッチ処理のための準備
    texts = [chunk.content for chunk in chunks]
    
    # エンベディングモデルによるベクトル化
    embeddings = await embedding_model.embed_documents(texts)
    
    # エンベディング付きチャンクの作成
    chunks_with_embeddings = []
    for chunk, embedding in zip(chunks, embeddings):
        chunks_with_embeddings.append(DocumentChunkWithEmbedding(
            **chunk.dict(),
            embedding=embedding
        ))
    
    return chunks_with_embeddings
```

#### 3.2.4 ベクトルデータベースへの格納

```python
async def store_in_vector_db(chunks_with_embeddings: list[DocumentChunkWithEmbedding]) -> None:
    """
    エンベディング付きチャンクをベクトルデータベースに格納する
    
    Args:
        chunks_with_embeddings: エンベディング付きドキュメントチャンクのリスト
    """
    # ベクトルDBへの一括挿入
    documents = [
        {
            "id": chunk.id,
            "document_id": chunk.document_id,
            "content": chunk.content,
            "embedding": chunk.embedding,
            "metadata": chunk.metadata
        }
        for chunk in chunks_with_embeddings
    ]
    
    await vector_db.upsert(documents)
```

### 3.3 検索エンジン

#### 3.3.1 クエリ処理

```python
async def process_query(query: str, filters: dict = None) -> QueryWithEmbedding:
    """
    検索クエリを処理する
    
    Args:
        query: ユーザーの検索クエリ
        filters: 検索フィルター（カテゴリ、タグなど）
        
    Returns:
        エンベディング付きクエリ
    """
    # クエリの前処理
    normalized_query = normalize_text(query)
    
    # クエリのエンベディング生成
    query_embedding = await embedding_model.embed_query(normalized_query)
    
    return QueryWithEmbedding(
        original_query=query,
        normalized_query=normalized_query,
        embedding=query_embedding,
        filters=filters or {}
    )
```

#### 3.3.2 ベクトル検索

```python
async def vector_search(query: QueryWithEmbedding, 
                       top_k: int = 5) -> list[SearchResult]:
    """
    ベクトル検索を実行する
    
    Args:
        query: エンベディング付きクエリ
        top_k: 取得する上位結果数
        
    Returns:
        検索結果のリスト
    """
    # フィルターの構築
    filter_expression = None
    if query.filters:
        filter_expression = build_filter_expression(query.filters)
    
    # ベクトル検索の実行
    search_results = await vector_db.search(
        query_embedding=query.embedding,
        filter=filter_expression,
        top_k=top_k
    )
    
    # 検索結果の変換
    results = []
    for result in search_results:
        results.append(SearchResult(
            chunk_id=result["id"],
            document_id=result["document_id"],
            content=result["content"],
            metadata=result["metadata"],
            score=result["score"]
        ))
    
    return results
```

#### 3.3.3 関連度スコアリングとランキング

```python
async def rank_search_results(results: list[SearchResult], 
                             query: QueryWithEmbedding) -> list[SearchResult]:
    """
    検索結果をランキングする
    
    Args:
        results: 検索結果のリスト
        query: エンベディング付きクエリ
        
    Returns:
        ランキング済みの検索結果
    """
    # ベクトル類似度以外の要素も考慮したスコアリング
    for result in results:
        # 基本スコアはベクトル類似度
        final_score = result.score
        
        # タイトルマッチボーナス
        if query.normalized_query.lower() in result.metadata.get("title", "").lower():
            final_score += 0.1
        
        # 最近更新されたドキュメントボーナス
        days_since_update = (datetime.now() - result.metadata.get("updated_at", datetime.now())).days
        recency_bonus = max(0, 0.05 * (1 - min(days_since_update, 30) / 30))
        final_score += recency_bonus
        
        # タグマッチボーナス
        query_terms = set(query.normalized_query.lower().split())
        document_tags = set([tag.lower() for tag in result.metadata.get("tags", [])])
        tag_overlap = len(query_terms.intersection(document_tags))
        tag_bonus = 0.02 * tag_overlap
        final_score += tag_bonus
        
        # 最終スコアの更新
        result.score = final_score
    
    # スコアでソート
    ranked_results = sorted(results, key=lambda x: x.score, reverse=True)
    
    return ranked_results
```

### 3.4 生成エンジン

#### 3.4.1 プロンプト構築

```python
def build_prompt(query: str, search_results: list[SearchResult]) -> str:
    """
    LLMへのプロンプトを構築する
    
    Args:
        query: ユーザーの質問
        search_results: 検索結果のリスト
        
    Returns:
        構築されたプロンプト
    """
    # コンテキスト情報の準備
    context_parts = []
    for i, result in enumerate(search_results):
        context_parts.append(f"[Document {i+1}] {result.content}")
    
    context = "\n\n".join(context_parts)
    
    # プロンプトテンプレート
    prompt = f"""
    あなたは質問応答AIアシスタントです。
    以下の情報源に基づいて、ユーザーの質問に答えてください。
    
    情報源:
    {context}
    
    質問: {query}
    
    回答の際は以下のガイドラインに従ってください:
    1. 情報源に含まれる情報のみを使用してください。
    2. 情報源に情報がない場合は、「その情報は提供されたドキュメントには含まれていません」と答えてください。
    3. 回答の根拠となる情報源を引用してください（例: [Document 1]によると...）。
    4. 簡潔かつ明確に回答してください。
    
    回答:
    """
    
    return prompt
```

#### 3.4.2 LLMによる回答生成

```python
async def generate_answer(prompt: str) -> str:
    """
    LLMを使用して回答を生成する
    
    Args:
        prompt: LLMへのプロンプト
        
    Returns:
        生成された回答
    """
    # LLM APIの呼び出し
    response = await llm_client.generate(
        prompt=prompt,
        max_tokens=1000,
        temperature=0.3,
        top_p=0.95,
        stop=None
    )
    
    return response.text
```

#### 3.4.3 回答の後処理

```python
async def postprocess_answer(answer: str, 
                           search_results: list[SearchResult],
                           static_site_config: StaticSiteConfig) -> EnhancedAnswer:
    """
    生成された回答を後処理する
    
    Args:
        answer: 生成された回答
        search_results: 検索に使用された結果
        static_site_config: 静的サイトの設定
        
    Returns:
        後処理された回答
    """
    # 引用の検証と強調
    processed_answer = highlight_citations(answer)
    
    # 静的サイトへのリンク追加
    document_ids = [result.document_id for result in search_results]
    static_links = await get_static_site_links(document_ids, static_site_config)
    
    # 回答の信頼度評価
    confidence_score = calculate_confidence(answer, search_results)
    
    return EnhancedAnswer(
        text=processed_answer,
        sources=[{
            "document_id": result.document_id,
            "title": result.metadata.get("title", ""),
            "static_link": static_links.get(result.document_id)
        } for result in search_results],
        confidence=confidence_score
    )
```

### 3.5 フィードバックシステム

#### 3.5.1 フィードバック収集

```python
async def record_feedback(query_id: str, 
                         feedback: FeedbackData) -> None:
    """
    ユーザーフィードバックを記録する
    
    Args:
        query_id: 検索クエリのID
        feedback: フィードバックデータ
    """
    await feedback_repository.save(SearchFeedback(
        id=str(uuid.uuid4()),
        query_id=query_id,
        rating=feedback.rating,
        comment=feedback.comment,
        created_at=datetime.now()
    ))
```

#### 3.5.2 フィードバック分析

```python
async def analyze_feedback(time_period: TimePeriod) -> FeedbackAnalysis:
    """
    特定期間のフィードバックを分析する
    
    Args:
        time_period: 分析対象の期間
        
    Returns:
        フィードバック分析結果
    """
    # 期間内のフィードバックを取得
    feedbacks = await feedback_repository.get_by_period(
        start_date=time_period.start_date,
        end_date=time_period.end_date
    )
    
    # 評価の集計
    ratings = [feedback.rating for feedback in feedbacks]
    avg_rating = sum(ratings) / len(ratings) if ratings else 0
    
    # 低評価のクエリ分析
    low_rated_queries = await query_repository.get_by_ids([
        feedback.query_id for feedback in feedbacks 
        if feedback.rating < 3
    ])
    
    # 頻出キーワード分析
    keywords = extract_keywords_from_queries(low_rated_queries)
    
    return FeedbackAnalysis(
        period=time_period,
        feedback_count=len(feedbacks),
        average_rating=avg_rating,
        low_rated_queries=low_rated_queries,
        common_keywords=keywords
    )
```