# 要約エンジン実装詳細

このドキュメントでは、Obsidian Noterの要約エンジンの実装詳細について説明します。

## 概要

`summarizer.py` モジュールは記事のテキストを受け取り、その重要な内容を抽出して簡潔な要約を生成します。このコンポーネントは以下を担当します：

- 記事の構造分析
- 重要な情報の特定
- 簡潔で一貫性のある要約の生成
- （オプション）外部AI APIとの連携

## 実装詳細

### 要約関数

記事を要約するメイン関数は以下のように定義されています：

```python
import re
from typing import Optional, Dict, Any, List
import os
import httpx

async def summarize_text(text: str, max_length: Optional[int] = None) -> str:
    """
    テキストを要約する
    
    Args:
        text: 要約対象のテキスト
        max_length: 要約の最大長（文字数）。Noneの場合は自動的に判断
    
    Returns:
        要約されたテキスト
    
    Raises:
        Exception: 処理中のエラー
    """
    # APIキーの取得（環境変数またはデフォルト設定から）
    api_key = os.environ.get("ANTHROPIC_API_KEY")
    
    # APIキーが設定されていない場合はルールベース要約を使用
    if not api_key:
        return _generate_rule_based_summary(text, max_length)
    
    # APIキーが設定されている場合はAPIを使用（オプション機能）
    try:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                "https://api.anthropic.com/v1/messages",
                headers={"x-api-key": api_key, "anthropic-version": "2023-06-01"},
                json={
                    "model": "claude-3-sonnet-20240229",
                    "max_tokens": 1000,
                    "messages": [
                        {
                            "role": "user",
                            "content": _generate_summary_prompt(text, max_length)
                        }
                    ]
                },
                timeout=60.0  # 60秒タイムアウト
            )
            
            # レスポンスの検証
            response.raise_for_status()
            result = response.json()
            
            # レスポンスから要約テキストを抽出
            return result["content"][0]["text"]
    except Exception as e:
        # APIエラー時はフォールバックとしてルールベース要約を使用
        return _generate_rule_based_summary(text, max_length)
```

### ルールベース要約の実装

外部APIに依存しないルールベースの要約生成：

```python
def _generate_rule_based_summary(text: str, max_length: Optional[int] = None) -> str:
    """
    ルールベースの要約を生成
    
    Args:
        text: 要約対象のテキスト
        max_length: 要約の最大長（文字数）
        
    Returns:
        要約テキスト
    """
    # テキストを文に分割
    sentences = [s.strip() for s in re.split(r'[.!?。！？]', text) if s.strip()]
    
    if not sentences:
        return "テキストが空か、解析できませんでした。"
    
    # 重要な文を選択するためのロジック
    important_sentences = []
    
    # 1. 最初の文は常に重要
    if sentences:
        important_sentences.append(sentences[0])
    
    # 2. キーワードを含む文を重要とみなす
    keywords = ["重要", "主要", "key", "main", "primary", "significant", "crucial", 
                "essential", "vital", "critical", "結論", "まとめ", "conclusion", 
                "summary", "in short", "therefore", "thus", "結果", "result"]
    
    for sentence in sentences[1:]:
        if any(keyword.lower() in sentence.lower() for keyword in keywords):
            if sentence not in important_sentences:
                important_sentences.append(sentence)
    
    # 3. 段落の最初の文（シンプルな近似として、長めの間隔の後の文）を重要とみなす
    paragraphs = text.split("\n\n")
    for paragraph in paragraphs[1:]:  # 最初の段落はすでに処理済み
        if paragraph.strip():
            first_sentence_match = re.search(r'^[^.!?。！？]+[.!?。！？]', paragraph.strip())
            if first_sentence_match:
                first_sentence = first_sentence_match.group(0).strip()
                # 既に追加されていなければ追加
                if first_sentence not in important_sentences and len(first_sentence) > 15:
                    important_sentences.append(first_sentence)
    
    # 4. 十分な重要文が見つからない場合は、追加の文を選択
    if len(important_sentences) < 3 and len(sentences) >= 3:
        # 最初と最後の文、そして真ん中あたりの文を選択
        middle_idx = len(sentences) // 2
        if len(sentences) > 1 and sentences[-1] not in important_sentences:
            important_sentences.append(sentences[-1])
        if middle_idx > 0 and middle_idx < len(sentences) and sentences[middle_idx] not in important_sentences:
            important_sentences.append(sentences[middle_idx])
    
    # 要約の構成
    summary_parts = []
    
    # 導入部（最初の文または2文）
    introduction = " ".join(important_sentences[:min(2, len(important_sentences))])
    summary_parts.append(f"## 要約\n\n{introduction}\n")
    
    # 主要ポイント（残りの重要文を箇条書きに）
    if len(important_sentences) > 2:
        summary_parts.append("## 主要ポイント\n")
        for sentence in important_sentences[2:]:
            summary_parts.append(f"- {sentence}")
    
    # 最終的な要約テキスト
    summary = "\n".join(summary_parts)
    
    # 最大長の制限
    if max_length and len(summary) > max_length:
        return summary[:max_length-3] + "..."
    
    return summary
```

### 要約用プロンプトの生成

外部APIを使用する場合のプロンプト生成関数：

```python
def _generate_summary_prompt(text: str, max_length: Optional[int] = None) -> str:
    """
    要約用のプロンプトを生成（外部API使用時）
    
    Args:
        text: 要約対象のテキスト
        max_length: 要約の最大長
        
    Returns:
        プロンプトテキスト
    """
    length_instruction = f"約{max_length}文字以内で" if max_length else "簡潔に"
    
    return f"""以下の記事を{length_instruction}要約してください。

要約には以下の要素を含めてください:
1. 記事の主題と目的
2. 主要な論点や発見（3-5点）
3. 重要な結論や示唆

要約はマークダウン形式で、以下の構造に従ってください:
```
## 要約
（全体の概要を2-3文で）

## 主要ポイント
- （主要ポイント1）
- （主要ポイント2）
- （主要ポイント3）
...
```

元の記事:
-----------
{text}
-----------

要約のみを返してください。追加の説明や元記事への言及は不要です。
"""
```

## 高度なテキスト分析機能

より高度な要約のためのテキスト分析機能：

```python
def analyze_text_structure(text: str) -> Dict[str, Any]:
    """
    テキストの構造を分析
    
    Args:
        text: 分析するテキスト
        
    Returns:
        構造分析結果の辞書
    """
    # 段落の検出
    paragraphs = [p.strip() for p in text.split('\n\n') if p.strip()]
    
    # 見出しの検出
    heading_pattern = r'^#+\s+(.+)
    headings = []
    
    for line in text.split('\n'):
        heading_match = re.match(heading_pattern, line)
        if heading_match:
            headings.append(heading_match.group(1))
    
    # 箇条書きの検出
    list_pattern = r'^\s*[-*]\s+(.+)
    list_items = []
    
    for line in text.split('\n'):
        list_match = re.match(list_pattern, line)
        if list_match:
            list_items.append(list_match.group(1))
    
    # 文の数
    sentences = [s.strip() for s in re.split(r'[.!?。！？]', text) if s.strip()]
    
    return {
        "paragraph_count": len(paragraphs),
        "headings": headings,
        "list_items": list_items,
        "sentence_count": len(sentences),
        "avg_sentence_length": sum(len(s) for s in sentences) / len(sentences) if sentences else 0
    }
```

### キーワード抽出

テキストから重要なキーワードを抽出する機能：

```python
def extract_keywords(text: str, max_keywords: int = 10) -> List[str]:
    """
    テキストから重要なキーワードを抽出
    
    Args:
        text: 分析するテキスト
        max_keywords: 抽出する最大キーワード数
        
    Returns:
        重要キーワードのリスト
    """
    # 単語の頻度を計算
    word_pattern = r'\b[a-zA-Z\u3040-\u309F\u30A0-\u30FF\u4E00-\u9FFF]{2,}\b'
    words = re.findall(word_pattern, text.lower())
    
    # ストップワード（除外する一般的な単語）
    stop_words = ["the", "and", "to", "of", "a", "in", "for", "is", "on", "that", "by", 
                 "this", "with", "are", "be", "as", "an", "ます", "です", "ました", "ません"]
    
    # 単語カウント
    word_count = {}
    for word in words:
        if word.lower() not in stop_words:
            word_count[word] = word_count.get(word, 0) + 1
    
    # 出現頻度でソート
    sorted_words = sorted(word_count.items(), key=lambda x: x[1], reverse=True)
    
    # 上位のキーワードを返す
    return [word for word, count in sorted_words[:max_keywords]]
```

## チャンク処理

長いテキストを効率的に処理するためのチャンク分割：

```python
def split_into_chunks(text: str, max_chunk_size: int = 8000) -> List[str]:
    """
    テキストを処理可能なチャンクに分割
    
    Args:
        text: 分割するテキスト
        max_chunk_size: チャンクの最大サイズ（文字数）
        
    Returns:
        チャンクのリスト
    """
    # テキストが十分短い場合はそのまま返す
    if len(text) <= max_chunk_size:
        return [text]
    
    # 段落でテキストを分割
    paragraphs = text.split("\n\n")
    
    chunks = []
    current_chunk = ""
    
    for paragraph in paragraphs:
        # 現在のチャンクにパラグラフを追加すると最大サイズを超える場合
        if len(current_chunk) + len(paragraph) + 2 > max_chunk_size:
            # 現在のチャンクがすでに存在する場合はリストに追加
            if current_chunk:
                chunks.append(current_chunk)
            
            # パラグラフ自体が最大サイズを超える場合は分割
            if len(paragraph) > max_chunk_size:
                # 文で分割
                sentences = [s.strip() + "." for s in re.split(r'[.!?。！？]', paragraph) if s.strip()]
                
                sub_chunk = ""
                for sentence in sentences:
                    if len(sub_chunk) + len(sentence) + 1 > max_chunk_size:
                        chunks.append(sub_chunk)
                        sub_chunk = sentence
                    else:
                        if sub_chunk:
                            sub_chunk += " " + sentence
                        else:
                            sub_chunk = sentence
                
                if sub_chunk:
                    current_chunk = sub_chunk
                else:
                    current_chunk = ""
            else:
                current_chunk = paragraph
        else:
            # 現在のチャンクに段落を追加
            if current_chunk:
                current_chunk += "\n\n" + paragraph
            else:
                current_chunk = paragraph
    
    # 最後のチャンクをリストに追加
    if current_chunk:
        chunks.append(current_chunk)
    
    return chunks
```

## まとめ

要約エンジンは以下の特徴を持っています：

1. **API非依存の基本機能**:
   - ルールベースのテキスト分析と要約生成
   - キーワードベースの重要文抽出
   - 段落構造を考慮した要約

2. **オプションのAPI連携**:
   - 外部AI APIを使用した高度な要約（設定している場合のみ）
   - エラーハンドリングとフォールバックメカニズム

3. **テキスト分析機能**:
   - 文書構造の解析
   - キーワード抽出
   - 長いテキストのチャンク処理

これにより、外部サービスに依存せずに基本的な要約機能を提供しながら、必要に応じて高度なAI機能で拡張できるように設計されています。
