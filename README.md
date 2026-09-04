# public-docs

公開・共有用ドキュメントを管理するためのリポジトリです。

設計資料、技術メモ、外部共有向けの PDF など、プロジェクトや技術思想を説明するドキュメントをここに集約します。

## Contents

```text
public-docs/
├── README.md
└── docs/
```

## Documents

| Path | Description |
| --- | --- |
| [docs/documentation-guideline.md](docs/documentation-guideline.md) | このリポジトリに置くドキュメントの書き方（構成、決定の残し方、公開前チェック） |
| [docs/libraries.md](docs/libraries.md) | 公開 Go ライブラリ 19 本のリファレンス（担当範囲と、その境界をそう決めた理由） |
| [docs/applications.md](docs/applications.md) | アプリケーション 4 本と MCP サーバー 2 本のリファレンス |
| [docs/url-naming-convention.md](docs/url-naming-convention.md) | Web アプリケーションの URL 命名規約（リソースの骨格、採らなかった案、既存アプリの寄せ方） |
| [docs/worker-convention.md](docs/worker-convention.md) | Cloud Tasks ワーカーの規約（ジョブの一生の順序、ペイロード、タスク名、採らなかった案） |
| [docs/readme-convention.md](docs/readme-convention.md) | アプリ README の規約（README と CLAUDE.md の役割分担、固定の骨格、書かないもの） |

## Update Workflow

1. `docs/` 配下に公開・共有したいドキュメントを配置する
2. 必要に応じて、この README の `Documents` を更新する
3. Git で変更履歴を残す

```bash
git status
git add README.md docs/
git commit -m "Update public docs"
```

## Repository Policy

- 公開前提の資料のみを配置する
- 機密情報、認証情報、個人情報を含めない
- 差し替え時はファイル名または README の説明で変更内容が追えるようにする

## License

ドキュメントの著作権は作成者に帰属します。
利用条件を明確にする必要がある場合は、個別のライセンスファイルまたは各ドキュメント内に記載してください。
