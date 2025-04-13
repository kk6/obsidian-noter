# チャット要約エンジン実装詳細

このドキュメントでは、Obsidian Noterのチャット要約エンジンの実装詳細について説明します。

## 概要

`chat_summarizer.py` モジュールはチャット会話を分析し、トピックや重要なポイント、結論などを抽出して要約を生成します。このコンポーネントは以下を担当します：

- チャットメッセージの解析と構造化
- 会話のトピック識別
- 重要なポイントの抽出
- 結論や合意点の特定
- ルールベースの要約生成
- （オプション）AI APIを使った高度な要約

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
        Exception: 処理中のエラー
    """
    # APIキーの取得（環境変数またはデフォルト設定から）
    api_key = os.environ.get("ANTHROPIC_API_KEY")
    
    # チャット形式の検出と分解
    messages = parse_chat_messages(chat_content)
    
    # APIキーが設定されていない場合はルールベース要約を使用
    if not api_key:
        return _generate_rule_based_chat_summary(messages)
    
    # APIキーが設定されている場合はAPIを使用（オプション機能）
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
    except Exception as e:
        # APIエラー時はフォールバックとしてルールベース要約を使用
        return _generate_rule_based_chat_summary(messages)
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

### ルールベースのチャット要約

APIに依存しないルールベースのチャット要約機能：

```python
def _generate_rule_based_chat_summary(messages: List[Dict[str, str]]) -> str:
    """
    ルールベースのチャット要約を生成
    
    Args:
        messages: パース済みのチャットメッセージ
        
    Returns:
        要約テキスト
    """
    # メッセージが少ない場合は簡易的な要約
    if len(messages) <= 1:
        if messages:
            return f"""## トピック
- {messages[0]["content"][:50]}...

## 主要なポイント
- チャットの主要なポイント（会話が短いため詳細な分析はできません）

## 結論
- 明確な結論は導き出せません（会話が短いため）"""
        else:
            return "チャットメッセージが見つかりませんでした。"
    
    # トピックの抽出（最初のユーザーメッセージから）
    topics = []
    first_user_message = next((msg["content"] for msg in messages if msg["role"] == "user"), "")
    if first_user_message:
        # 最初の文を抽出
        first_sentence = re.split(r'[.!?。！？]', first_user_message)[0]
        if first_sentence:
            topics.append(f"- {first_sentence}")
    
    if not topics:
        topics.append("- チャットの主要トピック")
    
    # 主要ポイントの抽出
    key_points = []
    
    # ユーザーメッセージから重要なポイントを抽出
    user_messages = [msg["content"] for msg in messages if msg["role"] == "user"]
    for msg in user_messages[1:min(4, len(user_messages))]:  # 最初のメッセージを除く
        sentences = re.split(r'[.!?。！？]', msg)
        if sentences:
            # 長い文は分割
            point = sentences[0][:60] + ("..." if len(sentences[0]) > 60 else "")
            if point and len(point) > 10:  # 十分に意味のある長さのポイントのみ
                key_points.append(f"- {point}")
    
    # キーワードを含む文を探す
    important_keywords = ["重要", "必要", "main", "key", "critical", "essential", "should", "must", "need"]
    for msg in messages:
        content = msg["content"]
        for keyword in important_keywords:
            if keyword.lower() in content.lower():
                # キーワードを含む文を抽出
                sentences = re.split(r'[.!?。！？]', content)
                for sentence in sentences:
                    if keyword.lower() in sentence.lower():
                        point = sentence.strip()[:60] + ("..." if len(sentence.strip()) > 60 else "")
                        if point and point not in key_points and len(point) > 10:
                            key_points.append(f"- {point}")
    
    # アシスタントの応答からポイントを抽出
    assistant_messages = [msg["content"] for msg in messages if msg["role"] == "assistant"]
    for msg in assistant_messages[:min(3, len(assistant_messages))]:
        sentences = re.split(r'[.!?。！？]', msg)
        if len(sentences) > 1:  # 少なくとも2文以上あるメッセージ
            point = sentences[1][:60] + ("..." if len(sentences[1]) > 60 else "")  # 2文目を使用
            if point and point not in key_points and len(point) > 10:
                key_points.append(f"- {point}")
    
    # 十分なポイントがない場合
    if len(key_points) < 2:
        key_points.append("- チャットの重要なポイント")
    
    # 結論の抽出（最後のメッセージから）
    conclusions = []
    
    # 最後のアシスタントメッセージを結論として使用
    last_assistant_msg = next((msg["content"] for msg in reversed(messages) 
                              if msg["role"] == "assistant"), "")
    if last_assistant_msg:
        # 最後の文を抽出（結論として）
        sentences = re.split(r'[.!?。！？]', last_assistant_msg)
        if sentences:
            conclusion = sentences[-1].strip()
            if conclusion and len(conclusion) > 10:
                conclusions.append(f"- {conclusion}")
    
    # 結論キーワードを含む文を探す
    conclusion_keywords = ["結論", "まとめ", "therefore", "conclusion", "finally", "in summary", 
                         "thus", "結果", "次のステップ", "next steps"]
    for msg in messages:
        content = msg["content"]
        for keyword in conclusion_keywords:
            if keyword.lower() in content.lower():
                # 結論キーワードを含む文を抽出
                sentences = re.split(r'[.!?。！？]', content)
                for sentence in sentences:
                    if keyword.lower() in sentence.lower():
                        point = sentence.strip()
                        if point and point not in conclusions and len(point) > 10:
                            conclusions.append(f"- {point}")
    
    # 十分な結論がない場合
    if not conclusions:
        conclusions.append("- 明確な結論は導き出せません")
    
    # 要約の構築
    summary_parts = [
        "## トピック",
        "\n".join(topics[:2]),
        "\n## 主要なポイント",
        "\n".join(key_points[:4]),
        "\n## 結論",
        "\n".join(conclusions[:2])
    ]
    
    return "\n".join(summary_parts)
```

### チャット要約プロンプトの生成

APIを使用する場合のプロンプト生成関数：

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

## 高度なチャット分析機能

より効果的なチャット分析のための拡張機能：

### メッセージの前処理

チャットメッセージの前処理と正規化：

```python
def preprocess_chat_messages(messages: List[Dict[str, str]]) -> List[Dict[str, str]]:
    """
    チャットメッセージの前処理と正規化
    
    - 空のメッセージの削除
    - 挨拶だけのメッセージのフィルタリング
    - メッセージの結合（短いメッセージをまとめる）
    
    Args:
        messages: 解析済みメッセージのリスト
        
    Returns:
        処理済みメッセージのリスト
    """
    # 結果のメッセージリスト
    processed_messages = []
    
    # 挨拶パターン
    greeting_patterns = [
        r'^(こんにちは|よろしく|はじめまして|hello|hi|hey|good morning|good afternoon|greetings)[\s\.,!]',
        r'^(how are you|nice to meet you)[\s\.,!]'
    ]
    
    # 現在のメッセージとロール
    current_content = ""
    current_role = None
    
    for msg in messages:
        content = msg["content"].strip()
        role = msg["role"]
        
        # 空のメッセージをスキップ
        if not content:
            continue
        
        # 挨拶だけのメッセージをスキップ
        is_greeting_only = False
        for pattern in greeting_patterns:
            if re.match(pattern, content.lower()) and len(content) < 30:
                is_greeting_only = True
                break
        
        if is_greeting_only:
            continue
        
        # メッセージの結合（同じロールの短いメッセージを連結）
        if role == current_role and len(current_content) < 200 and len(content) < 100:
            current_content += "\n" + content
        else:
            # 前のメッセージを保存
            if current_content and current_role:
                processed_messages.append({
                    "role": current_role,
                    "content": current_content
                })
            
            # 新しいメッセージを開始
            current_content = content
            current_role = role
    
    # 最後のメッセージを保存
    if current_content and current_role:
        processed_messages.append({
            "role": current_role,
            "content": current_content
        })
    
    return processed_messages
```

### トピックの検出

会話のトピックを検出する拡張機能：

```python
def detect_topics(messages: List[Dict[str, str]]) -> List[str]:
    """
    会話のトピックを検出
    
    Args:
        messages: 解析済みメッセージのリスト
        
    Returns:
        検出したトピックのリスト
    """
    # 頻出単語分析で最初のユーザーメッセージのキーワードを抽出
    first_user_message = next((msg["content"] for msg in messages if msg["role"] == "user"), "")
    
    # トピック候補
    topics = []
    
    # 最初のメッセージからトピックを推測
    if first_user_message:
        # 最初の文を取得
        first_sentence = re.split(r'[.!?。！？]', first_user_message)[0].strip()
        if first_sentence:
            topics.append(first_sentence[:50] + ("..." if len(first_sentence) > 50 else ""))
    
    # 質問文を検出
    question_pattern = r'([^.!?。！？]*\?[^.!?。！？]*)'
    for msg in messages[:min(3, len(messages))]:  # 最初の数メッセージのみ
        if msg["role"] == "user":
            questions = re.findall(question_pattern, msg["content"])
            for question in questions:
                if question.strip() and len(question.strip()) > 15:
                    topics.append(question.strip()[:50] + ("..." if len(question.strip()) > 50 else ""))
                    break  # 1つのメッセージから最大1つの質問を取得
    
    # 重複トピックの削除
    unique_topics = []
    for topic in topics:
        is_duplicate = False
        for existing in unique_topics:
            # 80%以上重複する場合はスキップ
            similarity = len(set(topic.lower().split()) & set(existing.lower().split())) / \