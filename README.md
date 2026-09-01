# ROB 474/574 Docs

> Documentation and lab instructions for ROB 474/574 at the University of Michigan.
> Published at <https://rob474-docs.github.io>.

Built with [Jekyll](https://jekyllrb.com/) and the
[Just the Docs](https://just-the-docs.com/) theme, deployed by the GitHub Actions
workflow in `.github/workflows/pages.yml` on every push to `main`.

## How to modify the files on GitHub

For quick typo fixes, use GitHub's web editor. Once you commit, the GitHub Action
builds and deploys automatically. For larger edits, work locally so you can preview.

## How to modify the site locally

### 1. Set up the environment and host locally

1. Install Jekyll and its prerequisites: [Jekyll docs](https://jekyllrb.com/docs/).
2. Install the bundler and jekyll gems: `gem install jekyll bundler`
3. From this folder:
   ```
   bundle install
   bundle exec jekyll serve
   ```

The site is then served at <http://localhost:4000>.

### 2. Add or edit articles

Articles live under `/docs`, images under `/assets/images`, PDFs under `/assets/pdfs`.

```
.
├── index.md            # site home page
└── docs
    └── labs
        ├── index.md    # section landing page
        ├── lab1.md
        ├── lab2.md
        ├── lab3.md
        └── lab4.md
```

- Every directory level needs an `index.md` declaring where it sits in the hierarchy.
- Each `.md` file starts with front matter that controls its place in the nav:

  ```
  ---
  layout: default
  title: Lab 1
  nav_order: 1
  parent: Labs
  last_modified_at: 2026-09-01 12:00:00 -0400
  ---
  ```

- `parent` / `grand_parent` name the enclosing sections by their `title`. A page with
  sub-pages needs `has_children: true`.
- See the [Just the Docs](https://just-the-docs.com/) docs for the full set of options.

### 3. Adding a new top-level section

Create `docs/<section>/index.md` with `has_children: true` and a `nav_order`, then add
child pages with `parent: <that section's title>`.

## Callouts

Custom callout styles are configured in `_config.yml`:
`highlight`, `important`, `new`, `note`, `warning`, `sanity_check`, `required_for_report`.

```markdown
{: .warning }
> Don't skip this step.
```

## Notes

Most content is Markdown, but some image pop-ups use raw HTML because of the
`magnific-popup` plugin (see `assets/js/`).
