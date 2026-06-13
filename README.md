# public-docs

公開・共有用ドキュメントを管理するためのリポジトリです。

設計資料、技術メモ、外部共有向けの PDF など、プロジェクトや技術思想を説明するドキュメントをここに集約します。

## Contents

```text
public-docs/
├── README.md
└── docs/
    └── AI_Fortress_Architecture.pdf
```

## Documents

| Path | Description |
| --- | --- |
| `docs/AI_Fortress_Architecture.pdf` | AI Fortress Architecture に関する設計資料 |

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
