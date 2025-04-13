# アーキテクチャ設計

Obsidian Noterは、Model Context Protocol (MCP)を活用したモジュール式アーキテクチャを採用しています。この文書では、システムの全体像と各コンポーネントの役割について説明します。

## 全体構成

![アーキテクチャ図](../assets/architecture.png)

システムは以下の主要コンポーネントで構成されています：

1. **MCPサーバー**: クライアント（Claude AI）とのインターフェース
2. **記事抽出エンジン**: URLやテキストからコンテンツを抽出
3. **要約エンジン**: 
   - 記事の要約: 抽出されたコンテンツを要約
   - チャットの要約: 会話内容を分析・整理
4. **Markdown変換**: 要約をObsidianで使いやすいMarkdown形式に変換

## データフロー

### 記事要約フロー
1. ユーザーがURLまたはテキストをClaude AIに入力
2. Claude AIがMCPプロトコルを通じてサーバーにリクエストを送信
3. サーバーが記事を抽出・要約し、Markdown形式で応答
4. Claude AIが応答をユーザーに表示
5. ユーザーがMarkdownをObsidianにコピー＆ペースト

### チャット要約フロー
1. ユーザーがチャット要約をClaude AIにリクエスト
2. Claude AIがチャット履歴とリクエストをMCPサーバーに送信
3. サーバーがチャット内容を分析・要約し、Markdown形式で応答
4. Claude AIが整形された要約をユーザーに表示
5. ユーザーがMarkdownをObsidianにコピー＆ペースト

## コンポーネントの詳細

### MCPサーバー (`server.py`)

- Model Context Protocol の実装
- FastMCPフレームワークを使用
- 複数のツールとプロンプトをエクスポート
- クライアントからのリクエストを処理

```python
# 基本的な実装構造
mcp = FastMCP("Obsidian Noter")

@mcp.tool()
async def summarize_to_obsidian(...): ...

@mcp.tool()
async def summarize_chat_to_obsidian(...): ...

@mcp.prompt()
def summarize_prompt(...): ...

@mcp.prompt()
def summarize_chat_prompt(...): ...
```

### 記事抽出エンジン (`extractor.py`)

- URLの検証
- ウェブページからのコンテンツ抽出
- ノイズ（広告、ナビゲーションなど）の除去
- 本文特定のアルゴリズム

```python
async def extract_article(url_or_text: str) -> str: ...
async def extract_from_url(url: str) -> str: ...
```

### 要約エンジン

#### 記事要約 (`summarizer.py`)

- テキスト要約のロジック
- AI APIとの統合（Anthropic Claude API等）
- 要約品質の最適化

```python
async def summarize_text(text: str, max_length: Optional[int] = None) -> str: ...
```

#### チャット要約 (`chat_summarizer.py`)

- チャット会話の解析
- トピックとキーポイントの抽出
- 議論の結論や合意点の特定

```python
async def summarize_chat(chat_content: str) -> str: ...
def parse_chat_messages(chat_content: str) -> List[Dict[str, str]]: ...
```

### Markdown変換 (`markdown.py`)

- 要約をObsidian互換のMarkdownに変換
- YAMLフロントマターの生成
- メタデータの付与
- 記事用・チャット用それぞれの変換機能

```python
def convert_to_markdown(summary: str, source: str, file_name: Optional[str] = None) -> str: ...
def convert_chat_to_markdown(summary: str, source: str, file_name: Optional[str] = None) -> str: ...
```

## エラーハンドリング

システムは以下のエラーケースを考慮します：

1. 無効なURL
2. アクセスできないウェブページ
3. 記事内容の抽出失敗
4. 要約APIのエラー
5. チャット内容の解析エラー
6. タイムアウト

各エラーは適切にハンドリングされ、ユーザーに明確なメッセージが表示されます。

## 拡張性

アーキテクチャは以下の点で拡張可能です：

1. **複数の入力形式**: URL、プレーンテキスト、チャット以外にもPDFやその他の形式に対応可能
2. **要約エンジンの交換**: 異なるAI APIやアルゴリズムを使用可能
3. **出力形式のカスタマイズ**: Markdownの構造やメタデータをカスタマイズ可能
4. **バッチ処理**: 複数記事・複数会話の一括処理
5. **チャット分析の高度化**: 感情分析やアクション項目の抽出など

## 開発優先順位

1. **フェーズ1**: 記事要約機能
   - 基本的な記事抽出と要約
   - Markdown変換
   - MCPサーバー統合

2. **フェーズ2**: チャット要約機能
   - チャット解析と要約
   - チャット専用のMarkdown変換
   - MCPサーバーへの統合

3. **フェーズ3**: 機能拡張
   - ユーザー設定のカスタマイズ
   - 高度な抽出アルゴリズム
   - 追加のコンテンツ形式サポート

## 性能考慮事項

- ウェブスクレイピングのタイムアウト設定
- 大きな記事のチャンク処理
- 長いチャット履歴の効率的な処理
- AI APIへのリクエスト最適化

