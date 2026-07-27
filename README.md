# hassaansaleem.github.io

Personal site + blog. Static HTML/CSS — **no build step, no dependencies**. GitHub Pages serves it as-is.

## Structure

```
index.html                         landing / portfolio (bio, projects, writing list)
writing/<slug>.html                one file per post
posts/<slug>.md                    markdown source for each post (for editing/archival)
assets/style.css                   shared styles (dark theme)
assets/*.png                       post images
```

## Preview locally

Just open `index.html` in a browser — everything is relative, so it works from the filesystem. No server needed.

## Publish to GitHub Pages

1. Create a **public** repo named exactly `HassaanSaleem.github.io` under the `HassaanSaleem` account.
2. Push this directory to its `main` branch.
3. On GitHub: **Settings → Pages**. For a `user.github.io` repo, Pages auto-serves the default branch root — no config needed.
4. Live within a minute or two at **https://hassaansaleem.github.io/**.

### Custom domain (optional, later)

Add a `CNAME` file containing your domain (e.g. `hassaansaleem.com`), point a DNS `CNAME`/`A` record at GitHub Pages, then set the domain under Settings → Pages.

## Add a new post

1. Copy `posts/agentic-coding-the-delta-the-loop-and-the-learning.md` to `posts/<new-slug>.md` and write it.
2. Copy `writing/agentic-coding-the-delta-the-loop-and-the-learning.html` to `writing/<new-slug>.html`, swap in the new title, meta, and body.
3. Add a `<a class="post-link">…</a>` block to the **Writing** section of `index.html` (newest first).

## TODO

- Fill in your real LinkedIn URL in `index.html` (currently a placeholder `https://www.linkedin.com/`).
