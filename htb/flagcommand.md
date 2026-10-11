# HTB - Flag Command (Web)
**Difficulty:** Very Easy

## Tools Used
- Browser (View Page Source)
- `curl`

## Steps
1. Opened the challenge in browser — found a terminal-style web game
2. Viewed page source — found three JS files: `commands.js`, `main.js`, `game.js`
3. Fetched `main.js` — found the command validation logic:
   `availableOptions[currentStep].includes(currentCommand) || availableOptions['secret'].includes(currentCommand)`
4. Noticed commands are fetched from `/api/options` — fetched it with curl
5. Found a hidden `secret` key with command: `Blip-blop, in a pickle with a hiccup! Shmiggity-shmack`
6. Posted the secret command directly to `/api/monitor`:
   `curl -X POST http://<IP>:<PORT>/api/monitor -H "Content-Type: application/json" -d '{"command": "Blip-blop, in a pickle with a hiccup! Shmiggity-shmack"}'`
7. Response contained the flag

## What I Learned
- Always read JS source files — game logic and API endpoints are often exposed
- APIs may return more data than the UI shows (hidden keys, secret options)
- You can interact with API endpoints directly via curl, bypassing the UI entirely
- Developer tools and source review are often faster than playing the game
