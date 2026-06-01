# kj6lnh-content

Source content for **kj6lnh.org** — a static [Hugo](https://gohugo.io/) site
(amateur radio, electronics, computer projects). Replaces the previous WordPress site.

## How it works

Pushing to `main` triggers an AWS CodePipeline (via a CodeConnections GitHub app)
that runs `buildspec.yml` in CodeBuild: it installs Hugo, renders the site, syncs
the output to S3, and invalidates CloudFront. The infrastructure lives in a separate
private repo (`kj6lnh-infra`, Terraform).

## Writing a post

```bash
hugo new posts/my-new-post/index.md   # creates a page bundle (put images alongside)
```

Edit the front matter (`title`, `date`, `categories`, `slug`), write Markdown, drop any
images in the same folder, then commit and push. Categories in use: `Electronics`,
`Ham Radio`, `Computers`. URLs are clean (`/my-new-post/`); add an `aliases` entry if you
need to preserve an old path.

## Local preview

```bash
git submodule update --init   # fetch the PaperMod theme
hugo server -D                # http://localhost:1313
```

## Layout

- `content/posts/<slug>/index.md` — posts (page bundles with their images)
- `content/about.md`, `content/privacy-policy.md` — standalone pages
- `hugo.toml` — site config (apex baseURL, `/:slug/` permalinks, menu, taxonomies)
- `themes/PaperMod` — theme (git submodule; pinned commit re-fetched in `buildspec.yml`)
- `buildspec.yml` — CodeBuild build + deploy steps
