# Necro Notes

Simple Jekyll blog scaffolded with Bootstrap.

## Workflow

This project is Docker-first. Use Docker-backed rake tasks instead of running Jekyll or Bundler directly on the host.

## Start the blog

```bash
rake install
rake serve
```

Then open `http://127.0.0.1:4000`.

Stop it with:

```bash
rake stop
```

The container runs Jekyll with live reload enabled, so template and post changes will refresh automatically while it is running.

## Docker helpers

```bash
rake docker:build
rake docker:serve
rake docker:stop
rake docker:shell
```

## Write a post

Create a new post with:

```bash
rake post title="My New Post"
```

This will generate a Markdown file in `_posts/` using the `YYYY-MM-DD-title.md` format.

```text
2026-04-10-my-new-post.md
```
