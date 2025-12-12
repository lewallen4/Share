name: Update plugins1 from plugins2

on:
  push:
    branches:
      - "**"

permissions:
  contents: write

jobs:
  update-plugins:
    # Prevent infinite loop — skip if the commit already contains [skip ci]
    if: "!contains(github.event.head_commit.message, '[skip ci]')"
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Update plugins1.txt using awk
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          set -euo pipefail

          BRANCH="${GITHUB_REF#refs/heads/}"
          echo "Running on branch: $BRANCH"

          P1="plugins1.txt"
          P2="plugins2.txt"

          START="##merge_start_tag##"
          END="##merge_end_tag##"

          if [ ! -f "$P1" ]; then
            echo "::warning::${P1} not found — exiting."
            exit 0
          fi

          if [ ! -f "$P2" ]; then
            echo "::warning::${P2} not found — exiting."
            exit 0
          fi

          # --- AWK merge operation ---
          awk -v start="$START" -v end="$END" -v p2="$P2" '
            # When we hit the START tag:
            $0 == start {
              print $0          # print the start tag
              while ((getline line < p2) > 0) print line   # insert plugins2.txt
              inblock=1
              next
            }

            # When we hit the END tag:
            $0 == end {
              inblock=0
              print $0
              next
            }

            # Only print lines outside the merge block
            !inblock { print $0 }
          ' "$P1" > "${P1}.new"

          # If no change, bail out gracefully
          if cmp -s "$P1" "${P1}.new"; then
            echo "No changes needed for $P1."
            rm -f "${P1}.new"
            exit 0
          fi

          mv "${P1}.new" "$P1"

          # Git config
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

          # Commit with [skip ci] to avoid loop
          git add "$P1"
          git commit -m "Update ${P1} from ${P2} [skip ci]" || true

          git push origin "HEAD:${BRANCH}"

      - name: Done
        run: echo "Plugin merge completed."