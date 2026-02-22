---
name: zopack
description: Export and import zo.space route setups as shareable .zopack.md files. Use when the user wants to package their zo.space routes for sharing, or when they receive a .zopack.md file to deploy. Triggers on "export my zo.space", "create a zopack", "import this zopack", or any mention of .zopack.md files.
compatibility: Created for Zo Computer
metadata:
  author: skeletorjs.zo.computer
---

# zopack

Package zo.space routes into a single shareable `.zopack.md` file that anyone can give to their Zo to deploy instantly.

## Export

To export routes from the user's zo.space into a `.zopack.md`:

1. Ask the user which routes to export (or use "all"). List their current routes with `list_space_routes`.
2. Fetch each route's code using `get_space_route(path)`.
3. Build a JSON array of route objects:
   ```json
   [
     { "path": "/api/foo", "route_type": "api", "public": true, "code": "..." },
     { "path": "/bar", "route_type": "page", "public": false, "code": "..." }
   ]
   ```
4. Pipe the JSON array to the export script:
   ```bash
   echo '<json array>' | bun /home/workspace/Skills/zopack/scripts/export.ts \
     --name "pack-name" \
     --description "What this does" \
     --output "Inbox/pack-name.zopack.md"
   ```
5. The script outputs the file path. Show the user the generated file.

The export script automatically:
- Detects the user's handle and replaces it with `{{HANDLE}}`
- Identifies npm dependencies not in the base zo.space install
- Detects shadcn/AnimateUI component imports and maps to registry URLs
- Finds filesystem paths referenced in code (directories, JSON files)
- Identifies required environment variable secrets

## Import

To import a `.zopack.md` file and deploy its routes:

1. Parse the pack file to get the deployment plan:
   ```bash
   bun /home/workspace/Skills/zopack/scripts/import.ts \
     --file "/path/to/pack.zopack.md" \
     --handle "<user-handle>" \
     --preview
   ```
   Use `--preview` first to show the user what will be deployed.

2. If the user confirms, run without `--preview` to get the full JSON plan:
   ```bash
   bun /home/workspace/Skills/zopack/scripts/import.ts \
     --file "/path/to/pack.zopack.md" \
     --handle "<user-handle>"
   ```

3. Parse the JSON output and execute the deployment in order:

   **a. Install dependencies:**
   - npm packages: `cd /__substrate/space && bun add <package>` for each
   - shadcn components: `cd /__substrate/space && npx shadcn@latest add <url> --yes` for each
   - For `shadcn:<name>` entries, use: `npx shadcn@latest add <name> --yes`

   **b. Create filesystem structure:**
   - Directories: `mkdir -p /home/workspace/<dir>` for each
   - Files: write each file with the specified initial content

   **c. Check for secret requirements:**
   - If secrets are listed, warn the user to configure them in Settings > Advanced before the routes will work

   **d. Deploy routes:**
   - Use `update_space_route(path, route_type, code, public)` for each route in the plan
   - Deploy API routes before page routes (pages may depend on APIs)

   **e. Verify:**
   - Run `get_space_errors()` to check for build/runtime errors
   - Report success or any issues

## Format Reference

The `.zopack.md` format:

```
---
format: zopack
version: "1.0"
name: pack-name
description: "What this does"
author: handle.zo.computer
routes: 3
exported: 2026-02-21
---

# pack-name

Description text.

## Routes

### `/path` (api|page, public|private)

\```typescript|tsx
// route code here
\```

## Dependencies

**npm packages** (not in default zo.space):
- `package-name`

**Components** (install via shadcn CLI):
- `https://animate-ui.com/r/component-name.json`
- `shadcn:component-name`

## Setup

**Directories to create:**
- `Data/my-dir`

**Files to initialize:**
- `Data/state.json` with content: `[]`

**Secrets required** (configure in Settings > Advanced):
- `API_KEY_NAME`

## Variables

| Placeholder | Description |
|---|---|
| `{{HANDLE}}` | Your zo.space handle |
```
