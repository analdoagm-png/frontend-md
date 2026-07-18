# Frontend Integration Playbook

A reusable policy for frontend-to-backend handoff. It defines the contracts, state coverage, permissions, analytics, accessibility, observability, and acceptance criteria that make integrations predictable.

## Use it

1. Read [FRONTEND.md](FRONTEND.md) before starting backend-dependent frontend work.
2. Copy the feature documentation and checklist conventions into the consuming project.
3. Keep feature-specific details in that project; update this repository when the cross-feature policy itself improves.

## Add it to another project

Download the current playbook into the root of an existing project:

```bash
curl -fsSL -o FRONTEND.md https://raw.githubusercontent.com/analdoagm-png/frontend-md/main/FRONTEND.md
```

For a stable dependency, replace `main` in the URL with a release tag or commit SHA that your project has reviewed.

## Repository contents

- `FRONTEND.md` — the canonical playbook
- `.github/PULL_REQUEST_TEMPLATE.md` — a ready-to-use integration review template

## Publishing to GitHub

Create an empty GitHub repository, then connect and publish this local repository:

```bash
git add .
git commit -m "Add frontend integration playbook"
git remote add origin <your-repository-url>
git push -u origin main
```

Choose a license before publishing if you want others to reuse the document under explicit terms.
