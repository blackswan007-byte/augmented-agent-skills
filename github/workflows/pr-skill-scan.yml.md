name: PR Skill Scan

on:  
  pull\_request:  
    paths:  
      \- "skills/\*\*"  
      \- "scan\_skills.py"  
      \- "scan\_pr\_skills.py"  
      \- "pyproject.toml"  
      \- "uv.lock"  
      \- ".github/workflows/pr-skill-scan.yml"

permissions:  
  contents: read  
  pull-requests: write

concurrency:  
  group: pr-skill-scan-${{ github.event.pull\_request.number }}  
  cancel-in-progress: true

jobs:  
  scan:  
    name: Scan changed skills  
    runs-on: ubuntu-latest  
    timeout-minutes: 60

    steps:  
      \- name: Checkout PR  
        uses: actions/checkout@v6  
        with:  
          fetch-depth: 0  
          ref: ${{ github.event.pull\_request.head.sha }}

      \- name: Detect changed skills  
        id: changed  
        run: |  
          set \-euo pipefail  
          BASE\_SHA="${{ github.event.pull\_request.base.sha }}"  
          HEAD\_SHA="${{ github.event.pull\_request.head.sha }}"  
          echo "Base: $BASE\_SHA"  
          echo "Head: $HEAD\_SHA"

          \# Files added/copied/modified/renamed under skills/\<skill\>/...  
          CHANGED\_FILES=$(git diff \--name-only \--diff-filter=ACMR "$BASE\_SHA" "$HEAD\_SHA" \-- 'skills/\*\*' || true)  
          echo "Changed files under skills/:"  
          echo "$CHANGED\_FILES"

          \# Derive unique top-level skill directories and keep only those that still exist with a SKILL.md  
          SKILL\_DIRS=$(echo "$CHANGED\_FILES" \\  
            | awk \-F/ 'NF\>=2 && $1=="skills" {print $1 "/" $2}' \\  
            | sort \-u)

          EXISTING=""  
          for d in $SKILL\_DIRS; do  
            if \[ \-f "$d/SKILL.md" \]; then  
              EXISTING="$EXISTING $d"  
            fi  
          done  
          EXISTING=$(echo "$EXISTING" | xargs || true)

          echo "Skill dirs to scan: '$EXISTING'"  
          echo "skill\_dirs=$EXISTING" \>\> "$GITHUB\_OUTPUT"

      \- name: Set up uv  
        if: steps.changed.outputs.skill\_dirs \!= ''  
        uses: astral-sh/setup-uv@v8.0.0  
        with:  
          enable-cache: true  
          cache-dependency-glob: uv.lock  
          python-version: "3.13"

      \- name: Install dependencies  
        if: steps.changed.outputs.skill\_dirs \!= ''  
        run: uv sync \--python 3.13

      \# Fork PRs do not receive SKILL\_SCANNER\_LLM\_API\_KEY. scan\_pr\_skills.py  
      \# detects the missing key, writes an explanatory sticky comment, and exits 0\.  
      \- name: Run scanner on changed skills  
        if: steps.changed.outputs.skill\_dirs \!= ''  
        id: scan  
        env:  
          SKILL\_SCANNER\_LLM\_API\_KEY: ${{ secrets.SKILL\_SCANNER\_LLM\_API\_KEY }}  
          SKILL\_SCANNER\_LLM\_MODEL: ${{ vars.SKILL\_SCANNER\_LLM\_MODEL || 'claude-opus-5' }}  
        run: |  
          uv run python scan\_pr\_skills.py \\  
            \--output pr\_scan\_comment.md \\  
            \--fail-on HIGH \\  
            ${{ steps.changed.outputs.skill\_dirs }}

      \- name: Prepare no-op comment  
        if: steps.changed.outputs.skill\_dirs \== ''  
        run: |  
          cat \> pr\_scan\_comment.md \<\<'EOF'  
          \<\!-- skill-security-scan \--\>  
          \#\# 🛡️ Skill Security Scan

          No skill directories (with a \`SKILL.md\`) were changed in this PR — nothing to scan.  
          EOF

      \- name: Upload scan comment as artifact  
        if: always()  
        uses: actions/upload-artifact@v4  
        with:  
          name: pr-skill-scan-comment  
          path: pr\_scan\_comment.md  
          if-no-files-found: ignore

      \- name: Post or update PR comment  
        if: always() && hashFiles('pr\_scan\_comment.md') \!= ''  
        uses: marocchino/sticky-pull-request-comment@v2  
        with:  
          header: skill-security-scan  
          path: pr\_scan\_comment.md

