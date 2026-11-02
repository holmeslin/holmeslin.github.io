# holmeslin.github.io

Feng-Chi 的個人筆記站，使用 [Hugo](https://gohugo.io/) 與 [toha](https://github.com/hugo-themes/toha) 主題，網址：https://holmeslin.github.io

## 本機開發

工具版本（Hugo extended、Go、Node）統一由 [`mise.toml`](mise.toml) 管理：

```sh
mise install        # 安裝鎖定版本的 hugo / go / node
mise run dev        # 安裝主題與 npm 相依套件，並啟動 http://localhost:1313（含草稿）
mise run build      # 正式建置，輸出到 public/
```

## 發表文章

```sh
hugo new posts/<slug>/index.md
```

1. 每篇文章是一個資料夾（page bundle），圖片可以直接放在同一個資料夾。
2. 在 front matter 的 `categories` 填一個主分類：`backend`（後端）、`frontend`（前端）、`devops`（維運與雲原生）、`ai`、`security`（資安）、`software-engineering`（軟體工程）。側欄會依分類自動列出文章。`tags` 用英文小寫加連字號，1～3 個，不重複分類名稱。新增分類時，在 `content/categories/<代號>/_index.md` 設定中文名稱（`title`）和排序（`weight`）。
3. 寫完把 `draft: true` 改成 `false`。
4. 在摘要段落後面放 `<!--more-->`，首頁卡片只會顯示這段之前的內容。

## 部署

push 到 `develop` 後，GitHub Actions（`.github/workflows/deploy.yml`）會建置網站並透過 GitHub Pages 官方流程發佈。repo 的 Settings → Pages → Source 必須設為「GitHub Actions」。
