# claude-statusline → moved to [cc-plugins](https://github.com/anthonybaldwin/cc-plugins)

This repository is archived. The status line now lives in the **cc-plugins** marketplace as the
`statusline-dashboard` plugin, with its full history:

```
/plugin marketplace add anthonybaldwin/cc-plugins
/plugin install statusline-dashboard@cc-plugins
/statusline-dashboard:setup
```

Source: <https://github.com/anthonybaldwin/cc-plugins/tree/main/plugins/statusline-dashboard>

If `~/.claude/settings.json` points at a checkout of this repo, run `bun install.js --uninstall`
here first, then set it up from cc-plugins (`/statusline-dashboard:setup`), or run `bun install.js`
from `plugins/statusline-dashboard/` in a cc-plugins checkout.
