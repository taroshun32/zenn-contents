# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Zenn CLI を使った技術記事・本の管理リポジトリ。mainブランチへのpushで Zenn (zenn.dev) に自動公開される。

## Commands

```bash
# ローカルプレビュー（ブラウザで記事を確認）
npx zenn preview

# 新しい記事を作成
npx zenn new:article

# 新しい本を作成
npx zenn new:book
```

## Repository Structure

- `articles/` - 技術記事（Markdown）。ファイル名がそのまま記事のスラッグになる
- `books/` - 本（現在未使用）
- `images/` - 記事で使用する画像。`images/<article-slug>/` のディレクトリ構成で格納

## Article Frontmatter Format

記事は以下のYAML frontmatterで始まる:

```yaml
---
title: "記事タイトル"
emoji: "絵文字1つ"
type: "tech"           # tech: 技術記事 / idea: アイデア
topics: ["topic1", "topic2"]  # 最大5つ
published: true        # true で公開
---
```

## Conventions

- 記事のファイル名はケバブケース（例: `aws-alb-error-cors.md`）
- 画像は `/images/<article-slug>/` に配置し、記事内では `![](/images/<article-slug>/filename.png)` で参照
- 言語: 記事は日本語で執筆
