# Connect Claude to your Adobe apps

**For people:** open the Claude desktop app, start a new session in the **Code** tab, and paste this:

> Read this page and set up the Adobe connections on my computer: https://github.com/Blaze-Media/claude-setup/blob/main/adobe.md

Claude will ask which apps you use, do the setup, and tell you the few things you need to click yourself. Plan on about 15 minutes. Save and close your work in Premiere, After Effects and Photoshop before you start.

---

## Instructions for the agent

You are setting up MCP servers so the Claude desktop app can operate the user's own copies of Adobe Premiere Pro, After Effects and Photoshop. The user is a video editor or designer, not a developer. Talk to them in plain language, one step at a time, and say what you are doing and why before each change.

### Ground rules

- **Mac only.** This page covers macOS. On Windows, stop and tell the user. Each project's README has Windows steps, and someone can help them set it up.
- **Never ask for a password in chat.** If something needs an administrator password (for example the Node.js installer), tell the user to run it themselves and type the password into the macOS prompt.
- **Don't use `sudo`.** Everything below installs into the user's home folder.
- **Back up before editing a config file:** `cp file file.bak-adobe-$(date +%H%M%S)`. Merge into the existing `mcpServers`. Never replace the file or remove entries already in it. Validate the JSON after each edit (`python3 -m json.tool file > /dev/null`).
- **Install only what this page lists.** All three servers are community projects (MIT licensed), not made by Adobe. Turn their usage reporting off as shown.
- **Don't open, change or save any of the user's projects.** All tests below only read.

### Step 1: Find out what to set up

1. List the installed apps: `ls -d /Applications/Adobe\ Premiere\ Pro* /Applications/Adobe\ After\ Effects* /Applications/Adobe\ Photoshop* 2>/dev/null`
2. Tell the user what you found and ask which ones they want connected. Set up only those.

### Step 2: Node.js (version 20 or newer)

1. Find Node: `command -v node && node -v`. If that finds nothing, also check `~/.local/share/fnm`, `~/.nvm`, `/opt/homebrew/bin` and `/usr/local/bin`.
2. If there is no Node 20+, ask the user to download the **LTS** macOS installer from https://nodejs.org, run it, and tell you when it's done. Then check again.
3. Write down three absolute paths. You'll need them because the desktop app does not load the user's shell setup, so bare `node` or `npx` may not be found:
   - `NODE` = the real path of `node` (resolve symlinks, e.g. `python3 -c "import os,shutil;print(os.path.realpath(shutil.which('node')))"`)
   - `NODE_BIN` = the folder containing it
   - `NPX` = `$NODE_BIN/npx`
4. Make global npm installs land in the home folder so no admin rights are needed. Run `npm config get prefix`. If that folder isn't writable by the user, run `npm config set prefix ~/.npm-global` and use `~/.npm-global/bin` and `~/.npm-global/lib/node_modules` below.

### Step 3: Where the connections are registered

The desktop app keeps **two** lists. Add each server to both, so it works in Chat/Cowork and in the Code tab:

- **Chat and Cowork:** `~/Library/Application Support/Claude/claude_desktop_config.json`, under the top-level `"mcpServers"` key. Create the file as `{"mcpServers": {}}` if it doesn't exist, and keep any other keys already there.
- **Code tab:** the user-level Claude Code config. If the `claude` command exists, use `claude mcp add-json --scope user <name> '<json>'`. Otherwise edit `~/.claude.json` (or `$CLAUDE_CONFIG_DIR/.claude.json` if that variable is set), top-level `"mcpServers"` key.

The server entries below use the same JSON in both places.

### Step 4: Premiere Pro

Project: https://github.com/hetpatel-11/Adobe_Premiere_Pro_MCP (npm `adobe-premiere-pro-mcp`). It adds a small panel inside Premiere that passes commands from Claude to Premiere.

1. Install it and its Premiere panel:
   ```bash
   npm install -g adobe-premiere-pro-mcp
   premiere-pro-mcp --install-cep
   premiere-pro-mcp --doctor
   ```
   (If `premiere-pro-mcp` isn't found, run it as `"$NODE" "$(npm root -g)/adobe-premiere-pro-mcp/dist/cli.js" --install-cep`.) The installer may add its own entry to `claude_desktop_config.json`. Replace that entry with the one below so both lists match.
2. Register it as `premiere-pro`:
   ```json
   {
     "command": "<NODE>",
     "args": ["<npm root -g>/adobe-premiere-pro-mcp/dist/index.js"],
     "env": {
       "PREMIERE_TEMP_DIR": "/tmp/premiere-mcp-bridge",
       "PREMIERE_MCP_TELEMETRY": "0",
       "PREMIERE_MCP_UPDATE_CHECK": "0"
     }
   }
   ```
   In the **Code tab** entry only, also add `"PREMIERE_MCP_TOOLSET": "full"` to `env`.
3. **The user's part:** quit and reopen Premiere, then open **Window > Extensions > MCP Bridge (CEP)**. The bridge folder must read `/tmp/premiere-mcp-bridge`, and the bridge should say it's running (click **Start Bridge** if not). Leave the panel open; it can sit docked anywhere.

### Step 5: After Effects

Project: https://github.com/Dakkshin/after-effects-mcp. It runs a small bridge window inside After Effects that checks for commands from Claude.

1. Download and build it in the home folder. The zip avoids needing git:
   ```bash
   mkdir -p ~/claude-adobe && cd ~/claude-adobe
   curl -L -o ae-mcp.zip https://github.com/Dakkshin/after-effects-mcp/archive/refs/heads/main.zip
   unzip -q -o ae-mcp.zip && rm ae-mcp.zip && mv -f after-effects-mcp-main after-effects-mcp
   cd after-effects-mcp && npm install
   ```
   `npm install` also builds it. Confirm `build/index.js` and `build/scripts/mcp-bridge-auto.jsx` exist.
2. **Skip `npm run install-bridge`.** It copies into `/Applications` and needs admin rights. Instead, have After Effects open the bridge by itself at startup: find the version folder with `ls ~/Library/Preferences/Adobe/After\ Effects/`. If it isn't there, ask the user to open and quit After Effects once. Then create `~/Library/Preferences/Adobe/After Effects/<version>/Scripts/Startup/mcp-bridge-start.jsx` containing:
   ```javascript
   // Opens the Claude MCP bridge when After Effects starts.
   app.scheduleTask('$.evalFile(new File("~/claude-adobe/after-effects-mcp/build/scripts/mcp-bridge-auto.jsx"))', 4000, false);
   ```
3. Register it as `after-effects`:
   ```json
   { "command": "<NODE>", "args": ["<HOME>/claude-adobe/after-effects-mcp/build/index.js"] }
   ```
4. **The user's part:** open After Effects, go to **Settings > Scripting & Expressions** (on older versions, **Preferences > Scripting & Expressions**), tick **Allow Scripts to Write Files and Access Network**, then quit and reopen After Effects. A small "MCP Bridge Auto" window should appear a few seconds after launch. Make sure its **Auto-run commands** box is ticked. If macOS asks whether After Effects may access the Documents folder, click **Allow** (the bridge passes messages through `~/Documents/ae-mcp-bridge`).

### Step 6: Photoshop

Project: https://github.com/alisaitteke/photoshop-mcp (npm `@alisaitteke/photoshop-mcp`). It drives Photoshop through macOS scripting, so nothing gets installed inside Photoshop.

1. Register it as `photoshop`:
   ```json
   {
     "command": "<NPX>",
     "args": ["-y", "@alisaitteke/photoshop-mcp"],
     "env": {
       "PATH": "<NODE_BIN>:/usr/bin:/bin:/usr/sbin:/sbin",
       "ANALYTICS_DISABLED": "1",
       "PSMCP_UPDATE_CHECK": "0",
       "PSMCP_FEEDBACK": "0"
     }
   }
   ```
2. Pre-download it so the first launch is quick: `"$NPX" -y @alisaitteke/photoshop-mcp < /dev/null`. It starts, logs "Photoshop MCP Server is running" and exits.
3. **The user's part:** Photoshop must be open, with no dialog box showing, whenever Claude uses it. The first time Claude uses it, macOS asks whether Claude may control Adobe Photoshop; click **OK**. If that was denied earlier, turn it on under **System Settings > Privacy & Security > Automation > Claude > Adobe Photoshop**.

### Step 7: Restart and check

1. Tell the user to **quit the Claude desktop app completely (Cmd+Q) and reopen it**. Servers only load at startup. Before they do, write a short note to `~/claude-adobe/SETUP-NOTES.md`: what you installed, the paths, which config files you changed and their backups, and the checks below. That way a new session can pick up where you left off.
2. In a **new** session, run one read-only check per app, with the app open:
   - Premiere: `verify_premiere_connection`. It should return the Premiere version and the open project.
   - After Effects: `get-help`, then `run-script` with `getProjectInfo`, then `get-results`.
   - Photoshop: `photoshop_ping`, then `photoshop_get_version`.
3. If a check fails:
   - **Premiere says the bridge isn't running:** the MCP Bridge (CEP) panel isn't open or started, or its folder isn't `/tmp/premiere-mcp-bridge`. Run `premiere-pro-mcp --doctor`.
   - **After Effects returns nothing or stale results:** the bridge window isn't open, Auto-run is off, or the scripting setting from Step 5 isn't ticked.
   - **Photoshop times out:** a dialog is open in Photoshop (a missing-font warning, for example). Close it and retry `photoshop_ping`. Error `-1743` means the Automation permission from Step 6 is off.
   - **A server doesn't appear at all:** usually a wrong absolute path or a JSON error in the config. Check the paths exist and the file validates, then restart the app.
4. Finish by telling the user, in plain words: which apps are connected, what they need to have open, and three things to try, for example:
   - "In Premiere, list the sequences in my open project and tell me how long each one is."
   - "In After Effects, make a 1920x1080 comp called Lower third with my name animating in from the left."
   - "In Photoshop, make a 1080x1080 document with a dark background and the words Episode 57 in the middle."

### Good habits to pass on

- Save before asking Claude to change a project. Undo works in all three apps.
- Ask it to describe what it's about to do before it does something big.
- The Adobe panels and bridge windows need to stay open while Claude works.

### Also available: Adobe's own connector

Adobe publishes an official connector for Claude covering Photoshop, Illustrator, Premiere, Lightroom, InDesign, Express, Firefly, Acrobat and Adobe Stock. It works through Adobe's online tools rather than the apps installed on this computer. To add it: in the desktop app choose **Customize** in the left sidebar, then **Plugins > Discover**, search **Adobe**, add it, then click **Connect** next to Adobe on the **Connectors** tab. Details: https://www.adobe.com/adobe-connectors/adobe-for-claude.html. It's worth mentioning to the user, but it works separately from the steps above.
