---
name: setup
description: Use when a developer wants to set up the ddmr-developer-context symlink. Finds the common parent of their DDmR repos and creates the CLAUDE.md symlink there.
---

# Setup DDmR Developer Context

Create the CLAUDE.md symlink so the DDmR service map is automatically loaded in every DDmR repo session.

## How It Works

Claude Code reads `CLAUDE.md` from every parent directory. This skill finds where the developer's DDmR repos live, determines their common parent directory, and creates a symlink from that parent to this repo's `CLAUDE.md`.

## Steps

1. **Find this repo's location.** Run `pwd` or check `git rev-parse --show-toplevel` to get the absolute path of this repo.

2. **Find DDmR repos on this machine.** Search for known DDmR repos by looking for directories containing these marker repos (check siblings of this repo first, then broaden):
   - `scoping-engine`
   - `declaration-service`
   - `declaration-storage-service`
   - `micro-frontend-hub`
   - `blueprint-management-service`
   - `configuration-profile-service`
   
   Start by checking the parent directory of this repo. If not found there, ask the developer where their DDmR repos are.

3. **Determine the common parent.** Find the deepest directory that is a parent of both this repo and at least 2 of the DDmR repos found in step 2. This is where the symlink goes.

4. **Check for conflicts.** If `CLAUDE.md` already exists at the common parent:
   - If it's already a symlink to this repo's CLAUDE.md → tell the developer they're already set up.
   - If it's a symlink to something else → warn and ask before replacing.
   - If it's a real file → warn and ask before replacing. Suggest they review its contents first.

5. **Create the symlink.** Run:
   ```bash
   ln -s <this-repo>/CLAUDE.md <common-parent>/CLAUDE.md
   ```

6. **Verify.** Read the first line of the symlink target to confirm it resolves correctly:
   ```bash
   head -1 <common-parent>/CLAUDE.md
   ```
   Should show `# DDmR Platform — Service Map`.

7. **Report.** Tell the developer:
   - Where the symlink was created
   - That it will be auto-loaded in every DDmR repo under that directory
   - To run `git pull` in this repo periodically to get updates from the team
