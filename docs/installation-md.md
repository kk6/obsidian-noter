# インストールガイド

このドキュメントでは、Obsidian Noterのインストール手順とセットアップ方法を詳しく説明します。

## 前提条件

- Python 3.12以上
- [uv](https://docs.astral.sh/uv/) (推奨) または pip
- [Claude Desktop](https://claude.ai/download) (MCPサーバーとの統合に必要)

## インストール方法

### 方法1: ソースからインストール (推奨)

1. リポジトリをクローンします：

```bash
git clone https://github.com/yourusername/obsidian-noter.git
cd obsidian-noter
```

2. パッケージをインストールします：

uvを使用:
```bash
uv pip install -e .
```

pipを使用:
```bash
pip install -e .
```

### 方法2: PyPIからインストール (将来的なオプション)

```bash
uv pip install obsidian-noter
```

または

```bash
pip install obsidian-noter
```

## Claude Desktopへの統合

Obsidian NoterをClaude Desktopに統合するには、MCPコマンドラインツールを使用します。

1. Claude Desktopがインストールされていることを確認します

2. 以下のコマンドを実行して、サーバーをインストールします：

```bash
mcp install src/obsidian_noter/main.py --name "Obsidian Noter"
```

3. Claude Desktopを起動し、新しいツールが利用可能になっていることを確認します

### カスタム環境変数の設定

APIキーなどの環境変数を設定する場合：

```bash
mcp install src/obsidian_noter/main.py --name "Obsidian Noter" \
    -v ANTHROPIC_API_KEY=your_api_key \
    -v CUSTOM_OPTION=value
```

または環境変数ファイルを使用：

```bash
mcp install src/obsidian_noter/main.py --name "Obsidian Noter" -f .env
```

## トラブルシューティング

### よくある問題

#### インストール時の依存関係エラー

問題:
```
ERROR: Could not find a version that satisfies the requirement...
```

解決策:
- Pythonのバージョンが3.12以上であることを確認してください
- 依存関係を個別にインストールしてみてください: `uv pip install httpx beautifulsoup4 pyyaml mcp`

#### MCPサーバーが見つからない

問題:
```
Error: Cannot find MCP server...
```

解決策:
- パッケージが正しくインストールされていることを確認してください
- パスが正しいことを確認してください: `src/obsidian_noter/main.py`

#### Claude Desktopとの接続エラー

問題:
Claude Desktopがサーバーに接続できない

解決策:
- Claude Desktopを再起動してみてください
- サーバーを再インストールしてください: `mcp install src/obsidian_noter/main.py --name "Obsidian Noter" --force`

### システム要件の確認

現在の Python バージョンを確認：

```bash
python --version
```

MCPのインストール状況を確認：

```bash
mcp --version
```

## アンインストール

Claude Desktopからサーバーを削除：

```bash
mcp uninstall "Obsidian Noter"
```

パッケージをアンインストール：

```bash
uv pip uninstall obsidian-noter
```

