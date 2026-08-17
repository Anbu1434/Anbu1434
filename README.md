#!/usr/bin/env bash
# Adds a gh-ascii ASCII profile card to the Anbu1434/Anbu1434 profile README.
# Run from anywhere; it clones into the current directory if needed.
set -euo pipefail

HANDLE="Anbu1434"
REPO="https://github.com/${HANDLE}/${HANDLE}.git"
DIR="${HANDLE}"

# --- 1. get the repo -------------------------------------------------------
if [ -d "${DIR}/.git" ]; then
  echo "==> Using existing clone: ${DIR}"
  git -C "${DIR}" pull --ff-only
else
  echo "==> Cloning ${REPO}"
  git clone "${REPO}" "${DIR}"
fi
cd "${DIR}"

# --- 2. download both themes ----------------------------------------------
# -f fails on HTTP errors, -L follows redirects. We additionally verify the
# payload is really an SVG: a 200 response carrying an HTML error page would
# otherwise sail straight through and commit a broken card.
for THEME in dark light; do
  OUT="${THEME}_mode.svg"
  echo "==> Fetching ${THEME} card -> ${OUT}"
  curl -fL --max-time 45 "https://gh.crafter.run/${HANDLE}?theme=${THEME}" -o "${OUT}"

  if ! head -c 1024 "${OUT}" | grep -qi "<svg"; then
    echo "ERROR: ${OUT} does not look like an SVG. First bytes:" >&2
    head -c 300 "${OUT}" >&2
    echo >&2
    exit 1
  fi
  echo "    ok: $(wc -c < "${OUT}") bytes"
done

# --- 3. insert the block at the top of README.md ---------------------------
touch README.md

if grep -q "dark_mode.svg" README.md; then
  echo "==> README already references the card; leaving markup untouched."
else
  echo "==> Inserting <picture> block at top of README.md"
  cp README.md README.md.bak            # safety net, deleted on success
  {
    cat <<'BLOCK'
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="dark_mode.svg" />
  <source media="(prefers-color-scheme: light)" srcset="light_mode.svg" />
  <img alt="Anbu1434's GitHub profile" src="dark_mode.svg" />
</picture>

BLOCK
    cat README.md.bak
  } > README.md

  # Confirm nothing was lost: new file must contain every old line.
  if [ "$(wc -l < README.md)" -lt "$(wc -l < README.md.bak)" ]; then
    echo "ERROR: README shrank. Restoring backup." >&2
    mv README.md.bak README.md
    exit 1
  fi
  rm README.md.bak
fi

# --- 4. review before committing ------------------------------------------
cat <<EOF

==> Files staged for review (NOT yet committed):
$(git status --short)

==> Look at both cards now, then commit. Fastest check is in a browser:
    https://gh.crafter.run/${HANDLE}?theme=dark
    https://gh.crafter.run/${HANDLE}?theme=light

    Or open the local files:
    xdg-open dark_mode.svg light_mode.svg   # Linux
    open dark_mode.svg light_mode.svg       # macOS
    start dark_mode.svg                     # Windows

==> If both read well, commit and push:
    git add dark_mode.svg light_mode.svg README.md
    git commit -m "feat: add gh-ascii profile card"
    git push

==> Then confirm it renders: https://github.com/${HANDLE}
EOF
