# Sync & Contribution Rules

## Source of Truth

This repository syncs **ONLY via Git/GitHub**. No Google Drive, iCloud, or any other cloud storage sync.

## Related Repositories

| Repo | Visibility | Purpose | Sync |
|------|-----------|---------|------|
| [learning-eks](https://github.com/smustafa75/learning-eks) | Public | Published EKS curriculum | Git → GitHub only |
| [eks-learn](https://github.com/smustafa75/eks-learn) | Private | Working/development copy | Git → GitHub only |
| [real-world-eks](https://github.com/smustafa75/real-world-eks) | Private | Capstone project (Flask + RDS + ArgoCD) | Git → GitHub only |

## Rules

- **No Google Drive sync** — these folders are excluded from Drive
- **Git is the single source of truth** — always `git pull` before starting work
- **Never commit secrets** — use environment variables and `.gitignore`
- **Check untracked files** before pushing (`git status`, then `git add -A` if needed)

## Authors

Sabir Mustafa & Kiro (AI pair)
