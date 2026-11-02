# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Personal Hugo blog (繁體中文 notes) on the [toha](https://github.com/hugo-themes/toha) theme, published at https://holmeslin.github.io. There are no tests or linters; a clean `hugo` build is the check.

## Commands

Tool versions (hugo-extended, go, node) are pinned in `mise.toml`, which CI also reads via `jdx/mise-action`. Change versions there only.

```sh
mise run install   # hugo mod tidy + hugo mod npm pack + npm install
mise run dev       # hugo server -D at http://localhost:1313
mise run build     # hugo --gc --minify --cleanDestinationDir (identical to CI)
hugo new posts/<slug>/index.md
```

The `WARN deprecated: css.Sass: libsass` line comes from the theme itself; it is expected.

## How the theme is wired

- toha v4 is a **Hugo Module** (`github.com/hugo-toha/toha/v4`, pinned in `go.mod`), not a submodule. Its JS/CSS dependencies come from npm: `hugo mod npm pack` generates `package.json` and `packages/hugoautogen/`, which are committed together with `package-lock.json`. Upgrade the theme with `hugo mod get github.com/hugo-toha/toha/v4@<tag>`, then `mise run install`.
- `hugo.yaml` `module.mounts` replaces Hugo's default static mount, so it must keep mounting `static` alongside the `node_modules` font/flag mounts.
- The homepage is built from data files, not content: `data/zh-tw/author.yaml`, `site.yaml` (footer copyright, license disclaimer, OpenGraph), and `sections/*.yaml`. Only `sections/about.yaml` is enabled; adding a section means adding its YAML file.
- `data/` files are not templated, so the footer copyright end year in `site.yaml` is static.
- Theme images (favicon, logos, avatar, post hero) are loaded with `resources.Get`, so they must live in `assets/`, not `static/`.
- Overrides live in `layouts/` (`robots.txt`, `partials/navigators/sidebar.html`) and `assets/styles/override.scss`. The sidebar override makes a group name toggle its post list (toha's own version links it to the group's `url`), so `menus.sidebar` groups carry no `url`. The theme's sitemap template lists regular pages only.
- KaTeX runs site-wide with only `$$…$$` and `\(…\)` delimiters; single `$` is deliberately excluded so prose like "$100" is left alone. Mermaid is via the `mermaid` shortcode.

## Posts

- Each post is a page bundle: `content/posts/<slug>/index.md`, URL `/posts/<slug>/`.
- The sidebar shows only posts with `menu.sidebar`. Set `parent` to one of the groups defined under `menus.sidebar` in `hugo.yaml` (`go`, `javascript-web`, `nginx-server`, `clean-code`, `site`); `archetypes/default.md` scaffolds this.
- Posts migrated from the old `even` theme carry `aliases:` for their old `/post/YYYY/MM/<slug>/` URLs (and `content/_index.md` aliases `/about/` to the homepage). Keep these when renaming or moving posts.
- Put `<!--more-->` after the intro: post cards show `.Summary`, not `description`.
- toha renders `tags` but not `categories` (they only feed the search index).
- To override the post author, set `author` front matter as a map (`{name, image}`); otherwise omit it and `data/zh-tw/author.yaml` is used.

## Deploy

Pushing to `develop` runs `.github/workflows/deploy.yml`, which builds and publishes through the official GitHub Pages actions (repo Settings → Pages → Source must be "GitHub Actions"). The old `master` branch holds output from the previous theme and is no longer deployed.
