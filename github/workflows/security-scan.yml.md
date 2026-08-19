name: Weekly Security Scan

on:  
  schedule:  
    \- cron: "0 9 \* \* 1" \# Every Monday at 09:00 UTC  
  workflow\_dispatch: \# Allow manual trigger  
    inputs:  
      full\_scan:  
        description: "Rescan every skill, ignoring cached findings"  
        type: boolean  
        default: false

permissions:  
  contents: write

jobs:  
  scan:  
    runs-on: ubuntu-latest  
    \# Scans run concurrently and reuse findings for unchanged skills, so a  
    \# typical incremental run is minutes. The headroom is for a full rescan  
    \# (triggered by a scanner/model change or the 30-day backstop) plus the  
    \# scanner's own rate-limit retries.  
    timeout-minutes: 60

    steps:  
      \- uses: actions/checkout@v6

      \- uses: astral-sh/setup-uv@v8.0.0  
        with:  
          enable-cache: true  
          cache-dependency-glob: uv.lock  
          python-version: "3.13"

      \- name: Install dependencies  
        run: uv sync \--python 3.13

      \- name: Run security scan  
        env:  
          SKILL\_SCANNER\_LLM\_API\_KEY: ${{ secrets.SKILL\_SCANNER\_LLM\_API\_KEY }}  
          SKILL\_SCANNER\_LLM\_MODEL: ${{ vars.SKILL\_SCANNER\_LLM\_MODEL || 'claude-opus-5' }}  
          \# Each skill scan is blocked on LLM network I/O, so concurrency is  
          \# bounded by API rate limits rather than by the runner. Lower this if  
          \# runs start hitting sustained 429s.  
          SKILL\_SCAN\_WORKERS: ${{ vars.SKILL\_SCAN\_WORKERS || '8' }}  
          SKILL\_SCAN\_FULL: ${{ inputs.full\_scan && '1' || '' }}  
        run: uv run python scan\_skills.py

      \- name: Upload report artifact  
        if: always()  
        uses: actions/upload-artifact@v4  
        with:  
          name: security-report  
          path: |  
            docs/security-report.md  
            docs/security-report.json  
          if-no-files-found: warn

      \- name: Commit updated security report  
        run: |  
          if \[ \-z "$(git status \--porcelain docs/security-report.md docs/security-report.json)" \]; then  
            echo "Report unchanged; nothing to commit."  
            exit 0  
          fi  
          git config user.name "github-actions\[bot\]"  
          git config user.email "41898282+github-actions\[bot\]@users.noreply.github.com"  
          git stash \--include-untracked  
          git pull \--rebase  
          git stash pop || true  
          git add docs/security-report.md docs/security-report.json  
          git commit \-m "chore: update security scan report \[skip ci\]"  
          git push

