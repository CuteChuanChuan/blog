# blog

> Write to remember, publish to be remembered.

Personal blog — notes on data engineering, distributed systems, and Rust.

Live at <https://raymondhung.dev>.

Built with [Zola](https://www.getzola.org/) and the [tabi](https://github.com/welpo/tabi) theme, deployed via Cloudflare Pages.

## Local dev

```bash
git clone --recursive git@github.com:CuteChuanChuan/blog.git
cd blog
zola serve
# → http://127.0.0.1:1111
```

`--recursive` matters — the tabi theme is a git submodule.

Requires Zola ≥ 0.22.1 (`brew install zola`).

## Writing posts

Posts live in `content/posts/YYYY-MM-<slug>.md`. See
`content/posts/2026-05-first-post.md` for a reference covering code blocks,
KaTeX, and Mermaid.

## Images

Two paths, by size and intent:

| Type | Location | Reference as |
|---|---|---|
| Decorative, < 100 KB | `static/img/` (in repo) | `/img/<file>.<ext>` |
| Post screenshots, charts, anything else | Cloudflare R2 (`blog-assets`) | `https://assets.raymondhung.dev/posts/YYYY-MM/<slug>/<file>.<ext>` |

R2 keeps the git repo small and offloads bandwidth from the Pages quota
(R2 has no egress fees).

Upload via rclone:

```bash
rclone copy ./screenshot.png r2:blog-assets/posts/2026-05/first-post/
```

The rclone profile lives in `~/.config/rclone/rclone.conf` (never in the
repo). Credentials come from a Cloudflare R2 API token scoped to the
`blog-assets` bucket.

The `blog-assets` bucket is for public assets only — never upload anything
private (drafts, internal screenshots, files with PII or secrets). If
private storage is needed later, create a separate bucket with public
access **disabled**.

## Daily workflow

Writing a new post end-to-end:

1. **Draft the post.** Create `content/posts/YYYY-MM-<slug>.md` and copy
   the frontmatter shape from `2026-05-first-post.md`. Set `date`,
   `taxonomies.tags`, `description`.

2. **Upload images** (if any) to a path that mirrors the post slug:

   ```bash
   rclone copy ./screenshot.png r2:blog-assets/posts/YYYY-MM/<slug>/
   ```

3. **Reference them** with absolute URLs:

   ```markdown
   ![alt text](https://assets.raymondhung.dev/posts/YYYY-MM/<slug>/screenshot.png)
   ```

4. **Preview locally:**

   ```bash
   zola serve   # → http://127.0.0.1:1111
   ```

5. **Ship it:**

   ```bash
   git add content/posts/<file>.md
   git commit -m "post: <title>"
   git push
   ```

   Cloudflare Pages rebuilds and deploys to <https://raymondhung.dev> in ~30s.

## Deploy

Push to `main` → Cloudflare Pages rebuilds and deploys in ~30s.
CI runs `zola check` + `zola build` on every PR.

## Repo layout

```
content/        Markdown posts + section indexes
static/         Small inline assets (favicon, robots, decorative img)
sass/           Custom SCSS overrides (currently empty — tabi defaults)
templates/      Override tabi templates here if needed
themes/tabi/    Theme (git submodule)
config.toml     Zola + tabi configuration
.github/        CI workflow
```
