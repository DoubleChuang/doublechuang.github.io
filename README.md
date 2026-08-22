# Double Chuang's Blog

使用 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) Jekyll 主題的部落格,部署於 GitHub Pages。

## 本地開發

```bash
bundle install
bundle exec jekyll serve
```

## 部署

推送到 `master` 分支會觸發 GitHub Actions (`pages-deploy.yml`) 自動建置並部署。

## 內容

- 部落格文章:`_posts/`(包含 [智慧家庭 Wiki](/wiki/) 章節)
- About 頁:`_tabs/about.md`
- 設定:`_config.yml`