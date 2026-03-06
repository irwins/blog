# Blog Instructions

Quick reminder for publishing this Hugo blog safely.

Reference: [Hugo Quick Start](https://gohugo.io/getting-started/quick-start/)

## Create a New Post

- `hugo new content posts/my-new-post.md`
- Use TOML front matter (`+++`)
- Keep `draft = true` while writing

## Write and Preview

- Preview drafts locally: `hugo server -D`
- Local preview URL includes `/blog/` path
- Do not commit output generated only for local preview checks

## Images and Assets (Important)

- Source images belong in root `images/` (not in `docs/images/`)
- Keep paths in posts like `../../images/<post-slug>/<file>`
- Root `images/` is mounted to `static/images/` by Hugo config
- If source image files are missing, older published posts will render with missing images

## Publish Checklist

1. Set `draft = false` in the post front matter
2. Run clean production build:
	- `hugo --cleanDestinationDir --baseURL "https://irwins.github.io/blog/"`
3. Verify generated files do not contain `localhost` or `livereload`:
	- `rg -n "localhost|livereload" docs/index.html docs/posts/index.html docs/index.xml docs/posts/index.xml`
4. Review `git status` and stage only intended publish files
5. Commit and push to `main`

## Useful Commands

- Local preview: `hugo server -D`
- Production build: `hugo --cleanDestinationDir --baseURL "https://irwins.github.io/blog/"`
- Find draft posts: `rg -n "^draft\s*=\s*true" content/posts/*.md`