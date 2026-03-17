# Ron's Tech Blog (Hugo + PaperMod)

Local development
- Clone with submodules or init them after cloning:
  - git clone --recurse-submodules git@github.com:RonVeen/ronsblog.git
  - Or: git submodule update --init --recursive
- Start the local server (includes drafts/future by default):
  - ./bin/dev
  - Env toggles:
    - BUILD_DRAFTS=0 ./bin/dev
    - BUILD_FUTURE=0 ./bin/dev
    - HUGO_BASEURL=http://localhost:1313/ ./bin/dev

Build (locally)
- hugo --gc --minify --baseURL "https://ronveen.com/"

Cloudflare Pages
- Build command:
  - hugo --gc --minify --baseURL "${HUGO_BASEURL:-$CF_PAGES_URL}"
- Environment (production):
  - HUGO_VERSION=0.153.3
  - HUGO_BASEURL=https://ronveen.com/
- Enable submodules for the PaperMod theme.

Generated output
- public/ is ignored in git. If you ever commit it accidentally:
  - git rm -r --cached public && git commit -m "Stop tracking generated public/ output"

Notes
- Theme: PaperMod via git submodule in themes/PaperMod
- Search: enabled with home JSON output; see hugo.toml [outputs]
- Series ordering: use series and series_order in front matter
