# public-docs

公開・共有してよいドキュメントの置き場です。

いまあるのは、個人開発のエコシステム（Go の公開ライブラリ群と、それを統合した Cloud Run 上の
アプリケーション）についての**リファレンス**と、複数のリポジトリが揃えるべき**規約**です。

## Documents

### リファレンス

| Path | Description |
| --- | --- |
| [docs/libraries.md](docs/libraries.md) | 公開 Go ライブラリ 19 本の境界の地図（担当範囲、隣との線引き、層の位置） |
| [docs/applications.md](docs/applications.md) | アプリケーション 4 本と MCP サーバー 2 本のリファレンス |

### 規約

| Path | Description |
| --- | --- |
| [docs/url-naming-convention.md](docs/url-naming-convention.md) | Web アプリケーションの URL 命名規約（リソースの骨格、採らなかった案、既存アプリの寄せ方） |
| [docs/worker-convention.md](docs/worker-convention.md) | Cloud Tasks ワーカーの規約（ジョブの一生の順序、ペイロード、タスク名、採らなかった案） |
| [docs/app-readme-convention.md](docs/app-readme-convention.md) | アプリ README の規約（README と CLAUDE.md の役割分担、固定の骨格、書かないもの） |
| [docs/library-readme-convention.md](docs/library-readme-convention.md) | ライブラリ README の規約（godoc / README / CLAUDE.md の三分割、見出しの語彙、書かないもの） |

### 書き方

| Path | Description |
| --- | --- |
| [docs/documentation-guideline.md](docs/documentation-guideline.md) | このリポジトリに置くドキュメントの書き方（構成、決定の残し方、公開前チェック） |

## 置くもの・置かないもの

**迷ったら「複数のリポジトリから引かれるか」で決めます。** 1 つのリポジトリでしか使わない説明は、
そこに置いたほうがコードと一緒に直せるぶん腐りません。使い方は各リポジトリの README、API は
godoc（pkg.go.dev）、内部の設計判断は各リポジトリの CLAUDE.md が持ちます。

**公開リポジトリです。** 非公開のリポジトリ名、認証情報、個人情報は置きません。書き方の決まりと
公開前のチェックリストは [documentation-guideline.md](docs/documentation-guideline.md) が持ちます。

新しい文書を足したら、上の表にも 1 行足してください。表と実ファイルが一致しているかは
次で確かめられます。

```bash
diff <(grep -o 'docs/[a-z-]*\.md' README.md | sort -u) <(ls -1 docs/ | sed 's|^|docs/|' | sort)
```

## License

ドキュメントの著作権は作成者に帰属します。
利用条件を明確にする必要がある場合は、個別のライセンスファイルまたは各ドキュメント内に記載してください。
