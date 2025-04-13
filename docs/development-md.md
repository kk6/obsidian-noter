# 開発ガイド

このドキュメントでは、Obsidian Noterの開発環境のセットアップと、プロジェクトへの貢献方法について説明します。

## 開発環境のセットアップ

### 前提条件

- Python 3.12以上
- [uv](https://docs.astral.sh/uv/) パッケージマネージャー
- Git

### 環境構築

1. リポジトリをクローン：

```bash
git clone https://github.com/yourusername/obsidian-noter.git
cd obsidian-noter
```

2. 開発依存関係をインストール：

```bash
uv pip install -e ".[dev]"
```

これにより、テスト・リンター・タイプチェックなどの開発ツールもインストールされます。

### 開発サーバーの実行

MCPの開発モードを使用して、サーバーをテストできます：

```bash
mcp dev src/obsidian_noter/main.py
```

これにより、MCPインスペクターが起動し、リアルタイムでサーバーとやり取りできます。

## 実装の優先順位

プロジェクトは以下の優先順位で実装を進めます：

1. **フェーズ1**: 記事要約機能
   - 記事抽出エンジン
   - 要約エンジン
   - Markdown変換
   - MCPサーバー基本実装

2. **フェーズ2**: チャット要約機能
   - チャット解析・要約機能
   - チャット専用Markdown変換
   - MCPサーバー拡張

3. **フェーズ3**: 機能強化と最適化
   - エラーハンドリングの強化
   - パフォーマンス最適化
   - ユーザーフィードバックに基づく改善

## プロジェクト構造

```
obsidian-noter/
├── .gitignore
├── README.md
├── pyproject.toml
├── src/
│   └── obsidian_noter/
│       ├── __init__.py
│       ├── main.py              # エントリーポイント
│       ├── server.py            # MCPサーバー実装
│       ├── summarizer.py        # 記事要約機能
│       ├── chat_summarizer.py   # チャット要約機能（フェーズ2）
│       ├── extractor.py         # 記事抽出機能
│       ├── markdown.py          # Markdown変換
│       └── utils.py             # ユーティリティ関数
└── tests/
    ├── __init__.py
    ├── test_server.py
    ├── test_summarizer.py
    ├── test_chat_summarizer.py  # フェーズ2
    ├── test_extractor.py
    ├── test_markdown.py
    └── test_utils.py
```

## コーディング規約

このプロジェクトでは以下のツールを使用しています：

- **Ruff**: リンティングとフォーマット
- **mypy**: 静的型チェック
- **pytest**: テスト

### コードスタイル

- PEP 8ガイドラインに従うこと
- タイプヒントを必ず使用すること
- ドキュメンテーション文字列（docstring）をGoogle styleで記述すること

### 型チェック

コードベース全体に対して型チェックを実行：

```bash
mypy src tests
```

### リンティング

コードスタイルをチェック：

```bash
ruff check src tests
```

自動修正を適用：

```bash
ruff format src tests
```

## テスト

### テストの実行

```bash
pytest
```

カバレッジレポートの生成：

```bash
pytest --cov=src
```

### テストの書き方

- 各モジュールに対応するテストファイルを `tests/` ディレクトリに作成してください
- 非同期関数のテストには `@pytest.mark.asyncio` デコレータを使用してください
- モックを適切に使用し、外部依存関係をテストから分離してください

テストの例：

```python
@pytest.mark.asyncio
async def test_extract_article():
    # モックアップセットアップ
    with patch('httpx.AsyncClient.get') as mock_get:
        mock_response = Mock()
        mock_response.text = "<html><body><article>テスト記事</article></body></html>"
        mock_response.raise_for_status = Mock()
        mock_get.return_value = mock_response
        
        # 関数を実行
        result = await extract_article("https://example.com/article")
        
        # 検証
        assert "テスト記事" in result
        mock_get.assert_called_once()
```

## プルリクエスト

プルリクエストを提出する前に：

1. すべてのテストが通ることを確認
2. リンターとフォーマッターを実行
3. 型チェックを実行
4. 変更に関連するドキュメントを更新

### プルリクエストのプロセス

1. フォークを作成し、機能ブランチを作成してください
2. コードを実装し、テストを追加してください
3. すべてのチェックが通ることを確認してください
4. プルリクエストを提出してください

## リリースプロセス

1. バージョン番号を `pyproject.toml` で更新
2. CHANGELOGを更新
3. リリースブランチを作成
4. テストが通ることを確認
5. タグを作成
6. PyPIにパッケージを公開（設定済みの場合）

