Issues Automation
=================

This repo can auto-create GitHub issues from a JSON file via GitHub Actions.

How it works
------------

- Edit `issues/issues.json` and push to the default branch.
- The workflow `.github/workflows/sync-issues.yml` reads the JSON and creates any missing issues (by title).
- Existing open issues with the same title are skipped to avoid duplicates.

JSON format
-----------

`issues/issues.json` must be an array of objects:

```
[
  {
    "title": "Short issue title",
    "body": "Longer Markdown description.",
    "labels": ["frontend", "a11y"],
    "assignees": ["octocat"],
    "milestone": 1
  }
]
```

Notes
-----

- Labels must already exist in the repository; if not, label assignment is skipped with a warning.
- To run on demand, trigger the workflow manually from the Actions tab ("Run workflow").
- Requires no personal access token; uses the built-in `GITHUB_TOKEN` with `issues: write` permissions.
