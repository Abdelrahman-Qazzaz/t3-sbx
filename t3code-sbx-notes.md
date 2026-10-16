# T3 Code + Docker Sandboxes (sbx): notes

*October 2026*

## Status

- T3 Code has **no native Docker Sandboxes support**. It treats a sandbox as an ordinary SSH host.
- A one-click "Docker Sandbox" option has been proposed by someone on Docker's sbx team ([t3code discussion #7841](https://github.com/pingdotgg/t3code/discussions/7841)). It is not built; there was no maintainer reply when last checked.
- Docker documents the SSH approach: [Connect T3 Code to a sandbox](https://docs.docker.com/ai/sandboxes/integrations/t3-code/). It needs sbx 0.37.0+.
- **Gotcha:** T3's server needs `node-pty`, which has no prebuilt binaries for Linux, and fresh sandboxes have no compiler. The **t3code kit** solves this. It has been published as `sbx/t3code-kit`, `sbx/t3code` and `docker/sbx-kit-t3code`.

## How it works

```
Mac                                  Linux microVM (<name>.sbx)
T3 Code app (window)  ──SSH──▶  T3 server → agents → your project
```

- The microVM is just another machine to T3. The SSH environment is set up on the Mac, pointing at the VM.
- With **Run on** set to the sandbox environment, everything T3 does (commands, edits, agents) runs inside the VM.
- **SSH is not isolation in itself.** The isolation comes from the sandbox: a throwaway VM that only has your project mounted, plus Docker's network rules. SSH to a regular server gives the agent full access to that machine.

## Workflow

1. Install sbx and log in: `brew install docker/tap/sbx`, then `sbx login`.
2. From the project folder, run the script from the `t3-sbx` repo:
   ```bash
   bash t3-sbx.sh [name]
   ```
   It creates the sandbox with the T3 kit, runs `sbx setup ssh`, tests the connection, checks that the T3 server can run, and prints the connection details.
3. In T3 Code, go to **Settings → Connections → Add environment → SSH**. Host: `<name>.sbx`, User: `agent`, Port: leave empty.
4. **Add project**, pick the sandbox environment, type the project's **Mac path** (e.g. `/Users/you/.../my-app`), then **Add**.
5. Start threads **from that project**, not from "No project".
6. For your own terminal in the same VM:
   ```bash
   sbx exec -it <name> bash   # or: ssh <name>.sbx
   npm run dev
   ```
7. Cleanup: `sbx rm <name>`, then remove the environment in T3 Code.

## Findings

- **Projects show where they run.** One labelled **Local** runs on the Mac with no sandbox. One labelled with the sandbox's name runs in the VM.
- **"No project"** on the sandbox runs in `/home/agent/.t3/scratch`, not in your code.
- **The project is mounted at its Mac path inside the VM** (where `sbx exec` lands you). `/home/agent/workspace` also exists but is **empty**, so a T3 project pointed there finds no code.
- **The project folder is shared, not copied.** Changes the agent makes, including deletions, hit the real files on the Mac. Commit to git often.
- **`node_modules`:** run `npm install` only inside the VM. Mac and Linux builds of native packages don't mix.
- **Dev server ports:** if the Mac's browser can't reach the server, tunnel it with `ssh -L 3000:localhost:3000 <name>.sbx`.
- "Full access" is reasonable inside the sandbox. On Local projects it means real access to your Mac.
