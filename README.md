# Obsidian Noter

[![Python Version](https://img.shields.io/badge/python-3.12%2B-blue)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Obsidian Noter は Model Context Protocol (MCP) を活用したサーバーで、ウェブ記事の要約やチャット会話の内容を整理し、Obsidianで使用できるMarkdown形式で提供します。

## 概要

シンプルなコマンド一つで以下を実行します：

1. **記事要約**: 「この記事を要約してObsidianに保存してください」
   - URLが提供された場合、ウェブページから本文を抽出
   - 記事の内容を要約
   - Obsidianに適したMarkdown形式に変換

2. **チャット要約**: 「このチャットでの議論内容をまとめてObsidianに保存してください」
   - チャット会話の内容を分析
   - トピックと主要なポイントを抽出
   - 議論の要点をObsidian用Markdownに整形

## 特徴

- **APIキー不要**: 基本機能はAPI連携なしで動作します
- **拡張可能**: Anthropic/OpenAI APIとの連携で高度な要約も可能（オプション）
- **シンプルな使用法**: 自然言語の指示だけで操作できます
- **柔軟な入力**: URL、テキスト、チャット履歴など様々な入力に対応

## クイックスタート

### インストール

```bash
# リポジトリをクローン
git clone https://github.com/yourusername/obsidian-noter.git
cd obsidian-noter

# 依存関係をインストール
uv pip install -e .
```

### Claude Desktopに統合

```bash
mcp install obsidian_noter/main.py --name "Obsidian Noter"
```

### 開発モードでテスト

```bash
mcp dev obsidian_noter/main.py
```

## 使用例

### 記事の要約

Claude AIに以下のように指示するだけです：

> この記事を要約してObsidianに保存してください: https://example.com/interesting-article

または

> この記事を要約してObsidianに保存してください:
> 
> 長い記事のテキストをここに貼り付けます...

### チャットの要約

議論の内容をまとめるには：

> このチャットでの議論内容をまとめてObsidianに保存してください

## 詳細ドキュメント

詳細については、以下のドキュメントを参照してください：

- [要件定義](docs/requirements.md)
- [インストール手順](docs/installation.md)
- [使用方法](docs/usage.md)
- [開発ガイド](docs/development.md)
- [アーキテクチャ](docs/architecture.md)
- [API リファレンス](docs/api/)

## ライセンス

MIT License
