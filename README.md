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
2. 在 front matter 的 `menu.sidebar.parent` 填入側欄群組：`go`、`javascript-web`、`nginx-server`、`clean-code`、`site`（定義在 `hugo.yaml` 的 `menus.sidebar`）。
3. 寫完把 `draft: true` 改成 `false`。
4. 在摘要段落後面放 `<!--more-->`，首頁卡片只會顯示這段之前的內容。

## 部署

push 到 `develop` 後，GitHub Actions（`.github/workflows/deploy.yml`）會建置網站並透過 GitHub Pages 官方流程發佈。repo 的 Settings → Pages → Source 必須設為「GitHub Actions」。
