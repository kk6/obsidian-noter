# チャット要約エンジン実装詳細

このドキュメントでは、Obsidian Noterのチャット要約エンジンの実装詳細について説明します。

## 概要

`chat_summarizer.py` モジュールはチャット会話を分析し、トピックや重要なポイント、結論などを抽出して要約を生成します。このコンポーネントは以下を担当します：

- チャットメッセージの解析と構造化
- 会話のトピック識別
- 重要なポイントの抽出
- 結論や合意点の特定
- 一貫性のある要約の生成

## 実装詳細

### チャット要約関数

チャットを要約するメイン関数は以下のように定義されています：

```python
from typing import List, Dict, Any, Optional
import re
import os
import httpx

async def summarize_chat(chat_content: str) -> str:
    """
    チャットの会話内容を要約する
    
    Args:
        chat_content: チャットの会話内容
        
    Returns:
        要約されたテキスト
    
    Raises:
        Exception: API呼び出しや処理中のエラー
    """
    # APIキーの取得（環境変数またはデフォルト設定から）
    api_key = os.environ.get("ANTHROPIC_API_KEY")
    
    # チャット形式の検出と分解
    messages = parse_chat_messages(chat_content)
    
    # APIキーが設定されていない場合の開発用モック
    if not api_key:
        return _mock_chat_summary(messages)
    
    # Anthropic Claude APIを使用した実際の要約処理
    try:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                "https://api.anthropic.com/v1/messages",
                headers={"x-api-key": api_key, "anthropic-version": "2023-06-01"},
                json={
                    "model": "claude-3-sonnet-20240229",
                    "max_tokens": 1500,
                    "messages": [
                        {
                            "role": "user",
                            "content": _generate_chat_summary_prompt(messages)
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
    except httpx.HTTPStatusError as e:
        raise Exception(f"API呼び出しエラー: {e.response.status_code} - {e.response.text}")
    except httpx.RequestError as e:
        raise Exception(f"リクエストエラー: {str(e)}")
    except Exception as e:
        raise Exception(f"チャット要約処理中にエラーが発生しました: {str(e)}")
```

### チャットメッセージの解析

チャットテキストをメッセージの配列に変換する関数：

```python
def parse_chat_messages(chat_content: str) -> List[Dict[str, str]]:
    """
    チャットの内容をメッセージの配列に分解
    
    複数の一般的なチャット形式に対応します:
    - "User: メッセージ" 形式
    - "Human: メッセージ" 形式
    - "Assistant/AI/Claude: メッセージ" 形式
    
    Args:
        chat_content: チャットの生テキスト
        
    Returns:
        メッセージの配列 [{"role": "user/assistant", "content": "..."}]
    """
    # メッセージパターンの検出
    messages = []
    
    # 一般的なチャット形式のパターン
    pattern = r'(User|Assistant|Human|AI|Claude):\s*(.*?)(?=(?:User|Assistant|Human|AI|Claude):|$)'
    matches = re.findall(pattern, chat_content, re.DOTALL)
    
    # 一致するものがなければ、テキスト全体を単一のユーザーメッセージとして扱う
    if not matches:
        return [{"role": "user", "content": chat_content.strip()}]
    
    for role, content in matches:
        # ロール名の正規化（userまたはassistant）
        role_normalized = "user" if role.lower() in ["user", "human"] else "assistant"
        messages.append({
            "role": role_normalized,
            "content": content.strip()
        })
    
    return messages
```

### チャット要約プロンプトの生成

要約のためのプロンプトを生成する関数：

```python
def _generate_chat_summary_prompt(messages: List[Dict[str, str]]) -> str:
    """
    チャット要約用のプロンプトを生成
    
    Args:
        messages: チャットメッセージのリスト
        
    Returns:
        要約のためのプロンプト
    """
    # メッセージを整形
    formatted_messages = []
    for msg in messages:
        role_display = "User" if msg["role"] == "user" else "Assistant"
        formatted_messages.append(f"{role_display}: {msg['content']}")
    
    chat_content = "\n\n".join(formatted_messages)
    
    return f"""以下のチャット会話を分析し、主要なトピック、重要なポイント、そして結論や合意点を抽出して要約してください。

会話：
-----------
{chat_content}
-----------

以下の形式で要約を作成してください：

## トピック
（会話で議論された主要トピックのリスト）

## 主要なポイント
（議論の中で共有された重要な情報や意見のリスト）

## 結論
（到達した結論、合意点、または次のステップのリスト）

各セクションはMarkdown形式で、箇条書きリストとして整理してください。
フォーマットだけでなく、内容も十分に詳細であることが重要です。
"""
```

### 開発用モック実装

APIキーがない場合などの開発時に使用するモック関数：

```python
def _mock_chat_summary(messages: List[Dict[str, str]]) -> str:
    """
    開発用のモックチャット要約機能
    
    実際のAPI呼び出しを行わず、簡易的な要約を返します。
    
    Args:
        messages: パース済みのチャットメッセージ
        
    Returns:
        模擬的な要約テキスト
    """
    # ユーザーとアシスタントのメッセージを分ける
    user_messages = [msg["content"] for msg in messages if msg["role"] == "user"]
    assistant_messages = [msg["content"] for msg in messages if msg["role"] == "assistant"]
    
    # メッセージが少ない場合は簡易的な要約
    if len(messages) <= 2:
        return """## トピック
- チャットの主要トピック

## 主要なポイント
- チャットの主要なポイント1
- チャットの主要なポイント2

## 結論
- チャットの結論または次のステップ"""
    
    # 模擬的なトピック抽出（最初のユーザーメッセージから）
    topics = []
    if user_messages:
        first_msg = user_messages[0]
        topics = [f"- {first_msg[:50]}..." if len(first_msg) > 50 else f"- {first_msg}"]
    
    # 模擬的な主要ポイント抽出（ユーザーメッセージから）
    key_points = []
    for msg in user_messages[1:3]:  # 最初の2-3メッセージを使用
        point = msg[:60] + "..." if len(msg) > 60 else msg
        key_points.append(f"- {point}")
    
    # 模擬的な結論抽出（最後のアシスタントメッセージから）
    conclusions = []
    if assistant_messages:
        last_msg = assistant_messages[-1]
        conclusion = last_msg[:60] + "..." if len(last_msg) > 60 else last_msg
        conclusions.append(f"- {conclusion}")
    
    # 模擬的な要約を構築
    summary_parts = []
    
    summary_parts.append("## トピック")
    if topics:
        summary_parts.extend(topics)
    else:
        summary_parts.append("- チャットの主要トピック")
    
    summary_parts.append("\n## 主要なポイント")
    if key_points:
        summary_parts.extend(key_points)
    else:
        summary_parts.append("- チャットの主要なポイント")
    
    summary_parts.append("\n## 結論")
    if conclusions:
        summary_parts.extend(conclusions)
    else:
        summary_parts.append("- チャットの結論または次のステップ")
    
    return "\n".join(summary_parts)
```

## 高度な機能（フェーズ2以降）

### トピック識別

会話のトピックを特定する機能：

```python
def identify_topics(messages: List[Dict[str, str]]) -> List[str]:
    """
    会話のトピックを特定
    
    キーワード頻度分析やAI APIを使用して会話の主要トピックを識別します。
    
    Args:
        messages: パース済みのチャットメッセージ
        
    Returns:
        トピックのリスト
    """
    # 初期実装: AIモデルを使用してトピックを特定
    # 将来的には: キーワード抽出、トピックモデリングなども組み合わせる
    
    # すべてのメッセージのコンテンツを連結
    all_content = " ".join([msg["content"] for msg in messages])
    
    # この部分はAI APIを使用する実装に置き換える
    topics = []
    
    return topics or ["会話の主要トピック"]
```

### キーポイント抽出

会話の重要なポイントを抽出する機能：

```python
def extract_key_points(messages: List[Dict[str, str]]) -> List[str]:
    """
    会話の主要ポイントを抽出
    
    意味的な重要性、反復、強調などに基づいて主要ポイントを抽出します。
    
    Args:
        messages: パース済みのチャットメッセージ
        
    Returns:
        主要ポイントのリスト
    """
    # この部分はAI APIを使用する実装に置き換える
    key_points = []
    
    return key_points or ["会話の主要ポイント1", "会話の主要ポイント2"]
```

### アクションアイテムの特定

会話から次のアクションを特定する機能：

```python
def extract_action_items(messages: List[Dict[str, str]]) -> List[str]:
    """
    会話からアクションアイテムを抽出
    
    会話内で言及された次のステップ、タスク、やるべきことを特定します。
    
    Args:
        messages: パース済みのチャットメッセージ
        
    Returns:
        アクションアイテムのリスト
    """
    action_items = []
    
    # アクションを示す表現のパターン
    action_patterns = [
        r'(?:する必要があります|しなければなりません|やるべきです)',
        r'(?:次のステップ|次にすべきこと)',
        r'(?:タスク|TODO|To-Do)',
        r'(?:期限|締め切り)',
        r'(?:担当者|責任者)'
    ]
    
    # 各メッセージを検査
    for msg in messages:
        content = msg["content"]
        for pattern in action_patterns:
            matches = re.finditer(pattern, content)
            for match in matches:
                # マッチした前後のコンテキストを取得
                start = max(0, match.start() - 50)
                end = min(len(content), match.end() + 50)
                context = content[start:end]
                
                # 文の境界で切り取り
                context = re.sub(r'^[^。.]*?[。.]', '', context)
                context = re.sub(r'[。.][^。.]*$', '', context)
                
                if context:
                    action_items.append(context.strip())
    
    return action_items
```

## エラーハンドリング

チャット要約処理中のエラーを適切に処理する例外クラス：

```python
class ChatSummarizerError(Exception):
    """チャット要約エンジンのエラー基底クラス"""
    pass

class ParseError(ChatSummarizerError):
    """チャットメッセージの解析エラー"""
    pass

class APIConnectionError(ChatSummarizerError):
    """API接続エラー"""
    pass

class APIResponseError(ChatSummarizerError):
    """API応答エラー"""
    pass

# エラーハンドリングの例
async def safe_summarize_chat(chat_content: str) -> str:
    """エラーハンドリングを含む安全なチャット要約処理"""
    try:
        if not chat_content.strip():
            raise ParseError("チャット内容が空です")
        
        return await summarize_chat(chat_content)
    except httpx.ConnectError:
        raise APIConnectionError("API接続エラー: ネットワーク接続を確認してください")
    except httpx.HTTPStatusError as e:
        raise APIResponseError(f"APIエラー: {e.response.status_code} - {e.response.text}")
    except Exception as e:
        raise ChatSummarizerError(f"チャット要約処理エラー: {str(e)}")
```

## 感情分析機能（将来的な拡張）

チャット会話の感情トーンを分析する機能：

```python
def analyze_sentiment(messages: List[Dict[str, str]]) -> Dict[str, Any]:
    """
    チャットの感情トーンを分析（将来的な実装）
    
    会話全体および各参加者の感情的な傾向を分析します。
    
    Args:
        messages: パース済みのチャットメッセージ
        
    Returns:
        感情分析の結果
    """
    # 将来的なAI APIを使用した感情分析の実装
    return {
        "overall": "neutral",
        "user": "interested",
        "assistant": "helpful",
        "progression": "positive"
    }
```

## パフォーマンス最適化

要約エンジンのパフォーマンスを最適化するための戦略:

1. **メッセージのフィルタリング**: 重要でないメッセージ（挨拶だけなど）を前処理でフィルタリング
2. **長い会話の分割処理**: 非常に長い会話を複数のチャンクに分割して処理
3. **キャッシング**: 類似の会話パターンに対する要約結果をキャッシング
4. **オフラインフォールバック**: API接続がない場合のルールベース要約機能

```python
async def optimize_chat_processing(chat_content: str) -> str:
    """長いチャットの最適化処理"""
    messages = parse_chat_messages(chat_content)
    
    # 重要でないメッセージのフィルタリング
    filtered_messages = filter_non_essential_messages(messages)
    
    # 会話が非常に長い場合は分割処理
    if len(filtered_messages) > 30:  # 閾値は調整可能
        return await process_long_conversation(filtered_messages)
    
    # 通常の処理
    return await summarize_chat_with_messages(filtered_messages)
```

## まとめ

チャット要約エンジンは、シンプルなテキスト解析から始まり、将来的にはAI APIを活用した高度な会話分析と要約生成を提供します。フェーズ1では基本的なチャットパース機能と単純な要約が実装され、フェーズ2以降で機能が大幅に拡張される予定です。
