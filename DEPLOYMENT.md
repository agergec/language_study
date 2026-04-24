# Deployment Guide

This project has two branches:

- **`beta`** — development/testing branch. Deployed to https://agergec.github.io/language_study_beta
- **`main`** — production branch. Deployed to the main GitHub Pages URL

## Branch Workflow

```
beta  →  test your changes  →  promote to main
```

## Switching Between Branches

```bash
git switch beta    # switch to beta (development)
git switch main    # switch to main (production)
```

## Push Changes to Beta

Work is done on the `beta` branch. Commit and push to trigger CI:

```bash
git switch beta
git add -A
git commit -m "describe your changes"
git push -u origin beta
```

## Promote Beta to Production

Before promoting, make sure beta is working correctly.

```bash
git switch main
cp ./beta/index.html ./index.html
git add -A
git commit -m "promote beta to production"
git push -u origin main
git switch beta    # switch back to beta to continue development
```
