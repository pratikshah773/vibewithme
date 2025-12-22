# Copilot / AI agent instructions — Repo snapshot (auto-generated)

Purpose
- Help AI coding agents become productive quickly in this repository.
- Be conservative: only change files you can verify or that the user explicitly asks you to edit.

What I discovered
- This repository currently contains a single agent definition at `.github/agents/coding frontend expert with next js.agent.md` which describes a Next.js frontend-focused assistant.
- No application source files (e.g. `package.json`, `next.config.js`, `src/` or `app/`) were present at scan time. Before making broad changes, search for these files.

How to start (actionable steps)
1. Run a targeted file search for typical web app entry points: `package.json`, `next.config.js`, `tsconfig.json`, `src/`, `app/`, `pages/`, `components/`.
2. If `package.json` exists, run `npm ci` (or `npm install`) then `npm run dev` to verify the dev server. If no scripts, ask the user.
3. Run linters/tests only after confirming the repo has the necessary tooling. Typical scripts: `npm run lint`, `npm test`, `npm run build`.

Repository-specific conventions and notes
- Agent files live under `.github/agents/`. Example: `.github/agents/coding frontend expert with next js.agent.md` — use these to infer intended assistant roles and constraints.
- Naming and spacing: some agent filenames include spaces. When referencing, use the exact filename or the agent content rather than a derived slug.

Editing and PR guidance for agents
- When modifying source files, prefer minimal, focused changes with tests or a local build to validate.
- If you change CI, workflow, or repo metadata, include a short explanation in the commit message and call out any required secrets or runner expectations.

Patterns and integration points to look for (search for these)
- API boundaries: `pages/api/*` or `app/api/*` for Next.js API routes.
- Server components vs client components: look for `use client` directives or `app/` (Next.js App Router).
- Environment/config: `.env`, `.env.local`, `next.config.js`, and platform config (Vercel, Netlify) in workflow files.

Behavior rules for the AI agent (practical constraints)
- Do not invent missing files. If expected files (e.g., `package.json`) are absent, ask the user before scaffolding a project.
- Avoid large refactors without user approval. Offer a plan and run small, verifiable steps.
- Prefer to run diagnostics locally (build, lint, tests) and report outputs rather than editing to fix unknown failures.

If you add or change an agent file
- Preserve the `description` and `tools` keys in the existing agent files unless the user requests otherwise.
- When merging content into `.github/copilot-instructions.md`, keep original agent guidance and add a brief changelog header describing what was consolidated.

Examples (from this repo)
- Agent definition: `.github/agents/coding frontend expert with next js.agent.md` — use its `description` field to adopt a Next.js-focused review lens.

When in doubt
- Ask a single clarifying question about missing context (build system, package manager, deploy target) before making non-trivial changes.

---
If you'd like, I can now: (1) run a wider file search for common Next.js files, (2) scaffold a minimal `package.json` and `README.md`, or (3) open a draft PR with this `copilot-instructions.md`.
