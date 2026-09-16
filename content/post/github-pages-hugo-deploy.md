---
title: "HugoサイトをGitHub Pagesへ自動デプロイする"
date: 2026-09-16
tags:
  - Hugo
  - GitHub Pages
  - GitHub Actions
summary: "HugoサイトをGitHub Actionsでビルドし、GitHub Pagesへ自動デプロイする設定"
---

## ご相談

Hugoで作成したサイトを、GitHub Pagesへ静的サイトとしてデプロイできるようにしたいという相談がありました。

## 解決

`.github/workflows/hugo.yml` を作成し、`main` ブランチへの push をきっかけに、HugoでサイトをビルドしてGitHub Pagesへデプロイするようにしました。

```yaml
name: Build and deploy Hugo site

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.123.7

    steps:
      - name: Checkout
        uses: actions/checkout@v7
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v6

      - name: Install Hugo
        run: |
          curl -fsSL -o /tmp/hugo.tar.gz \
            "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          tar -xzf /tmp/hugo.tar.gz -C /tmp hugo
          sudo install /tmp/hugo /usr/local/bin/hugo

      - name: Build
        run: hugo --gc --minify --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v5
        with:
          path: ./public

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

GitHubリポジトリの `Settings → Pages → Source` は `GitHub Actions` に設定します。

これで、`main` に push すると、HugoのビルドからGitHub Pagesへの公開まで自動で実行されます。
