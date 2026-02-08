# Migrating a Repository to the Astro Starlight Pipeline

This runbook documents how to onboard a downstream repo into the f5xc-template governance model and Astro Starlight documentation pipeline.

## Prerequisites

- The repo is owned by `robinmordasiewicz` on GitHub
- You have the `REPO_ADMIN_TOKEN` secret configured in the downstream repo
- The f5xc-docs-builder Docker image (`ghcr.io/robinmordasiewicz/f5xc-docs-builder:latest`) supports Astro Starlight builds
- GitHub Pages is enabled (or will be enabled by enforcement)

## Step-by-Step Process

### 1. Add Per-Repo Config

Create `.github/config/repo-settings.json`:

```json
{
  "repository": {
    "description": "Your repo description",
    "homepage": "https://robinmordasiewicz.github.io/REPO_NAME/"
  },
  "topics": ["f5xc"]
}
```

### 2. Add Caller Workflows

Copy these from the template's `workflows/` directory into the downstream repo's `.github/workflows/`:

| File | Source in Template | Purpose |
|------|-------------------|---------|
| `enforce-repo-settings.yml` | `workflows/enforce-repo-settings.yml` | Calls reusable enforcement workflow |
| `auto-merge.yml` | `workflows/auto-merge.yml` | Auto-merges PRs once checks pass |
| `github-pages-deploy.yml` | `workflows/github-pages-deploy.yml` | Builds and deploys docs via Astro |
| `require-linked-issue.yml` | `.github/workflows/require-linked-issue.yml` | Blocks PRs without linked issues |

### 3. Add Governance Files

Copy from the template root:

- `CONTRIBUTING.md` -- contributor workflow rules
- `CLAUDE.md` -- AI assistant instructions
- `.editorconfig` -- editor style settings
- `LICENSE` -- MIT license
- `.github/PULL_REQUEST_TEMPLATE.md`
- `.github/ISSUE_TEMPLATE/bug_report.md`
- `.github/ISSUE_TEMPLATE/feature_request.md`
- `.github/ISSUE_TEMPLATE/documentation.md`
- `.github/ISSUE_TEMPLATE/config.yml`
- `.github/CODEOWNERS` (set `* @robinmordasiewicz`)

### 4. Convert Docs to Starlight MDX

Replace any existing `docs/` content with Starlight-compatible `.mdx` files.

**Landing page** (`docs/index.mdx`):

```yaml
---
title: Site Title
description: Site description
template: splash
sidebar:
  hidden: true
hero:
  tagline: Your tagline
  actions:
    - text: Get Started
      link: 01-getting-started/
      icon: right-arrow
      variant: primary
---
```

**Content pages** (`docs/01-page-name.mdx`):

```yaml
---
title: Page Title
description: Page description
---

Content here in standard Markdown.
```

### 5. Delete Legacy Files

Remove any previous docs tooling:

- `mkdocs.yml`
- `requirements-docs.txt`
- `docs/overrides/`
- `docs/stylesheets/`
- `docs/javascripts/`
- Any MkDocs-specific Markdown (`.md` files with MkDocs admonitions)

### 6. Add to Downstream Registry

In f5xc-template, add the repo to `.github/config/downstream-repos.json`.

### 7. Verify

```bash
# Trigger enforcement
gh workflow run enforce-repo-settings.yml --repo robinmordasiewicz/REPO_NAME

# Check Pages config
gh api repos/robinmordasiewicz/REPO_NAME/pages

# Delete legacy gh-pages branch if it exists
gh api repos/robinmordasiewicz/REPO_NAME/git/refs/heads/gh-pages --method DELETE
```

## Starlight MDX Format Reference

### Admonitions

MkDocs Material uses `!!! type "Title"` with indented content. Starlight uses fenced directives:

| MkDocs Material | Starlight MDX |
|---|---|
| `!!! note "Title"` + indented content | `:::note[Title]` + content + `:::` |
| `!!! success "Title"` + indented content | `:::tip[Title]` + content + `:::` |
| `!!! warning "Title"` + indented content | `:::caution[Title]` + content + `:::` |
| `!!! danger "Title"` + indented content | `:::danger[Title]` + content + `:::` |
| `??? info "Title"` (collapsible) | `<details><summary>Title</summary>` + content + `</details>` |

### Icons

Replace Material icons with emoji or text:

| Material Icon | Replacement |
|---|---|
| `:material-arrow-up-bold-circle:` | Up arrow or text badge |
| `:material-arrow-down-bold-circle:` | Down arrow or text badge |
| `:material-plus-circle:` | `+` or text badge |
| `:material-minus-circle:` | `-` or text badge |
| `:material-refresh:` | Refresh symbol or text badge |

## Dynamic Doc Generation Pattern

For repos where docs are generated from pipeline artifacts (e.g., validation reports):

1. The pipeline workflow generates MDX files and commits them back to `docs/` via an auto-merged PR
2. The commit to `docs/` on main triggers the standard `github-pages-deploy.yml` workflow
3. This keeps the template's reusable workflow standard while supporting dynamic content

Example update-docs job to add to the pipeline workflow:

```yaml
  update-docs:
    name: Update Documentation
    needs: [validate]
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install -e .
      - uses: actions/download-artifact@v4
        with: { name: validation-reports, path: reports/ }
      - run: python -m scripts.generate_docs --output docs/01-validation-report.mdx
      - name: Commit and PR if changed
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add docs/
          if git diff --cached --quiet; then
            echo "No docs changes"
            exit 0
          fi
          BRANCH="docs/update-validation-report-$(date +%s)"
          git checkout -b "$BRANCH"
          git commit -m "docs: update validation report"
          git push origin "$BRANCH"
          gh pr create --title "docs: update validation report" \
            --body "Auto-generated from validation pipeline run."
          gh pr merge --auto --squash
```

## Troubleshooting

### Chicken-and-egg problem
If the enforcement workflow itself is broken in downstream repos, it can't self-heal. Use the GitHub Contents API to manually push fixes:

1. Get file SHA via `gh api repos/OWNER/REPO/contents/PATH`
2. Create branch from main SHA
3. PUT file with base64 content + old SHA + branch
4. Create PR
5. Auto-merge

### Pages not building
- Verify `build_type: workflow` via `gh api repos/OWNER/REPO/pages`
- Check that `docs/` directory exists with at least `index.mdx`
- Ensure the docs-builder image is accessible

### Auto-merge not working
- Check that the `auto-merge.yml` caller workflow is present
- Verify `GITHUB_TOKEN` permissions include `contents: write` and `pull-requests: write`
- Ensure branch protection allows auto-merge
