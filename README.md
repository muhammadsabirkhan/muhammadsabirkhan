name: Update Profile Dashboard

on:
  schedule:
    - cron: "0 */6 * * *"   # every 6 hours
  push:
    branches: [main]
    paths-ignore:
      - "assets/**"
      - "README.md"
  workflow_dispatch:

permissions:
  contents: write

concurrency:
  group: profile-update
  cancel-in-progress: true

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Generate dashboard
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GH_USER: muhammadsabirkhan
        run: python scripts/generate.py

      - name: Commit changes
        run: |
          git config user.name  "profile-bot"
          git config user.email "profile-bot@users.noreply.github.com"
          git add -A
          git diff --cached --quiet || git commit -m "chore: refresh live profile dashboard"
          git push
