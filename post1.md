---
title: AIコーディングを含むGitHub Workflow
---

# はじめに

単純作業だから自動化したいのに、人間らしい判断も必要で自動化できない……というとき、GitHubのワークフローにAIエージェントを組み込んだら解決するかもしれません。ここではboneのサーバーエンジニアの作業をワークフロー化した事例を紹介します。

# 課題
従来、クライアント用のマスターデータを追加するとき、クライアントエンジニアがサーバーエンジニアに仕様を伝え、DBマイグレーションや管理画面の修正を実装するという流れでした。これは単純な作業ですが、新しいコードを挿入する適切な場所を決めたり既存のコーディングスタイルに合わせたりするのはRubyのような言語では書きにくく、エンジニアの仕事になっていました。

# 解決方法
クライアントエンジニアが新しいテーブル/カラムの仕様をJSONで書き、それをパラメータとして GitHub Workflowを起動し、ActionとしてClaude Codeに改修させました。 TeamやProといったプランのClaude Codeの使用量は個人に紐づいてしまい、共有のworkflowに不向きなので、AWS Bedrockで提供されるClaude Sonnetを使うことで個人に紐づかない完全に従量制の運用が可能になりました。 また入力となるJSONに対してJSONスキーマを定義しておき、TypeScriptで検証するActionを最初に挟むことで無駄なトークン消費を防ぎました。

# 実装の要点

## workflow_dispatchトリガー

トリガーとして`workflow_dispatch`を設定しブラウザから実行可能にします。

## permissions

ワークフローによるコードの変更とPR作成を許可します。

## Claude Codeのインストール

公式の手順に従ってClaude Codeをインストールするステップを挿入します。所要時間は約10秒。

## 環境変数

以下の環境変数を設定します:

- AWS_REGION
- AWS_ACCESS_KEY_ID（secrets経由）
- AWS_SECRET_ACCESS_KEY（secrets経由）
- CLAUDE_CODE_USE_BEDROCK（1）

## スキルの用意

予め必要な手順を実行するスキルを作成し、リポジトリにコミットしておきます（今回は`.claude/skills/db-migration/SKILL.md`）。

## Claude Codeを非インタラクティブモードで実行

claudeの`-p`オプションによりインタラクティブモードに入らずに一つのプロンプトを処理させます。プロンプトでスキルと仕様JSONを渡します。

# サンプル

実際のワークフローから関連部分を抜粋して示します。

```yaml
name: create-migration2
on:
  workflow_dispatch:
    inputs:
      spec:
        description: Migration specification
        required: true
        type: string
jobs:
  migration:
    runs-on: ubuntu-latest
    services: （略）
    permissions:
      contents: write
      pull-requests: write
    env:
      MYSQL_HOST: 127.0.0.1
      REDIS_HOST: 127.0.0.1
      MIGRATION_SPEC: ${{ inputs.spec }}
      BUNDLE_PATH: vendor/bundle
    steps:
      - uses: actions/checkout@v6
      - name: （略: 仕様JSONの検証、ブランチ作成、Rails環境セットアップ）
      - name: Install Claude Code
        run: curl -fsSL https://claude.ai/install.sh | bash
      - name: Generate migration
        env:
          AWS_REGION: ap-northeast-1
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          CLAUDE_CODE_USE_BEDROCK: 1
        run: |
          ~/.local/bin/claude --allowedTools Read,Write,Edit,Bash,Skill --model sonnet -p "/db-migration specification:
          ${MIGRATION_SPEC}"
      - name: （略: コミット、プッシュ、PR作成）
```

# まとめ
このように、GitHub WorkflowにAIエージェントを組み込むことで、単純作業と人間らしい判断が必要なタスクの両方を自動化できます。これによりエンジニアはより価値の高い仕事に集中できるようになるでしょう。
