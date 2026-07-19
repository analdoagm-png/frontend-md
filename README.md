# Frontend-to-Backend Health Audit Playbook

A reusable, evidence-based guide for auditing frontend-to-backend health. It helps product designers and engineering teams find integration risks, recommend fixes, and raise backend-owned gaps with enough context to resolve them.

## Use it

1. Place [FRONTEND.md](FRONTEND.md) in the repository you want to audit.
2. Follow its audit workflow to trace real user flows and assess each integration domain from available evidence.
3. Report findings with priority, owner, evidence, impact, recommendation, and verification method.
4. Use the Backend Guidance Register for missing capabilities or decisions that Product Design and Frontend cannot resolve alone.
5. Use the Product Design Response Plan to address P0–P3 findings in order and verify closure.
6. Keep project-specific audit reports in the audited project; update this repository when the cross-project audit policy improves.

## Add it to another project

Download the current playbook into the root of an existing project:

```bash
curl -fsSL -o FRONTEND.md https://raw.githubusercontent.com/analdoagm-png/frontend-md/main/FRONTEND.md
```

For a stable dependency, replace `main` in the URL with a release tag or commit SHA that your project has reviewed.

## Repository contents

- `FRONTEND.md` — the canonical audit playbook and Backend Guidance Register
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
