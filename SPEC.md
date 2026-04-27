# SPEC — Garu Docs MCP Surface: Snippet Inlining + Page Enrichment

> **Goal:** Make `docs.garu.com.br/guias/mcp-server` and the eight `/guias/integracoes/*` pages a complete, self-contained, crawler-friendly install reference. A developer (or an LLM agent) reading any one of these pages must see the exact install command without resolving an unrendered Mintlify `<Snippet>` reference.

**Owner:** bissuh
**Status:** Planning
**Last updated:** 2026-04-27
**Companion spec:** `/Users/bissuh/Projects/garu-projects/gateway/SPEC.md` (marketing `/mcp` page + SEO infrastructure)

---

## 1. Why

A test-drive simulation of "fresh user setting up Garu MCP" uncovered two concrete problems on the docs site:

1. **Snippet rendering is unreliable from the LLM/crawler side.** When `https://docs.garu.com.br/guias/integracoes/claude-code` is fetched, the `<Snippet file="snippets/mcp-install-claude-code.mdx" />` reference is not surfaced in the rendered output. The actual install command (`claude mcp add garu -- npx -y @garuhq/mcp`) is invisible to anything that doesn't execute Mintlify's client renderer end-to-end. This breaks the very audience the MCP page is meant for: AI agents reading docs.

2. **The MCP page is information-thin compared to industry references.** [resend.com/mcp](https://resend.com/mcp) ships hero + 8 prompts + tool catalog grouped by category + per-client install snippets + testimonials + FAQ-equivalent content. Our `guias/mcp-server.mdx` has the catalog and 5 prompts but no inline install (only Mintlify Tabs that hide commands behind clicks), no FAQ/troubleshooting, no transport deep-dive, no error-handling guidance.

The marketing-side fix (companion spec) addresses discovery and conversion. This spec addresses **comprehension and execution** once a developer (or agent) is on the docs.

---

## 2. Scope

### In scope

- **Inline install commands** in the 4 surviving `/guias/integracoes/*.mdx` pages (Claude Code, Cursor, Codex, Windsurf), replacing the `<Snippet file="...">` references with literal fenced code blocks.
- **Delete `bolt.mdx`, `lovable.mdx`, `replit.mdx`, `v0.mdx`** entirely. These are cloud/web-based AI builders that don't support local-stdio MCP today, so the existing pages teach a non-working setup. Remove from `docs.json` nav and from `llms.txt`.
- **Inline install commands** in `guias/mcp-server.mdx` Tabs section.
- **Delete the now-orphaned snippets** (`mcp-install-*.mdx`) once references are gone. Keep `cli-install.mdx`, `auth-header.mdx`, etc. — they're untouched.
- **Enrich `guias/mcp-server.mdx`** with: (a) FAQ/Troubleshooting section, (b) error-handling section ("o que acontece quando o agente erra"), (c) example prompts expanded from 5 → 8 to match the marketing page, (d) "Próximos Passos" pointing to changelog tutorial article (companion spec ships this).
- **Update `llms.txt`** to expose every page including the new sections, with one-sentence descriptions tuned for LLM consumption.
- **Update `docs.json` `metadata`** with per-page OG/Twitter overrides where Mintlify supports them; add `og:image` pointing to a Garu-branded MCP card.
- **Verify `<Snippet>` removal** doesn't break other pages. Audit all `<Snippet>` usage; some may be legitimate (e.g., `auth-header.mdx` is a small reusable block).

### Out of scope

- Migrating off Mintlify. Mintlify works; the bug is our usage pattern, not the platform.
- Adding English versions. Marketing chose pt-BR-only for v1; docs site already pt-BR.
- Schema.org / structured data injection (Mintlify's surface is limited; revisit if it becomes possible).
- New tutorials or recipes beyond the FAQ/Troubleshooting addition.
- Changes to `api-reference/*` pages.

---

## 3. Architecture decisions

### 3.1 Source-of-truth for install commands

**Decision: README in `Garu-Pagamentos/garu-mcp` repo is the canonical source.**

When a command changes, the release engineer:
1. Updates the `garu-mcp` README.
2. Updates `docs/guias/mcp-server.mdx` (1 place).
3. Updates `docs/guias/integracoes/<client>.mdx` (1 place per client).
4. Updates `gateway/frontend/src/screens/marketing/McpInstallTabs.tsx` (1 place).

Total: 1 + 1 + 8 + 1 = 11 places **in the worst case** (every client command changed). In practice, commands change roughly once per major version (~yearly), and the per-client commands are highly stable (the package name is stable; the per-tool CLI subcommand is stable). The cost of duplication is low; the benefit (each page is self-contained, crawler-readable, AI-ingestable) is high.

### 3.2 Why drop `<Snippet>` instead of fixing it

Two rendering failure modes exist:
- **Mode A**: Mintlify's static export does include the snippet content, but our specific `<Snippet>` invocations have a path or syntax issue. Test: open a snippet in a browser incognito + view source. If the install command is in the rendered HTML, Mode A.
- **Mode B**: Mintlify renders snippets client-side via JS hydration. Crawlers without JS execution don't see them.

**We don't need to know which mode.** Inlining solves both. Empirically (per the test-drive), the content was missing — fixing the symptom directly is faster than diagnosing Mintlify internals.

The cost: 8 small files inlined into 9 pages. Each inline is 5–14 lines of code. Total ~80 lines of duplicated content that updates ~yearly.

### 3.3 Snippets that stay

Audit (verified 2026-04-27):

| Snippet | Status | Reason |
|---|---|---|
| `snippets/mcp-install-claude-code.mdx` | **Delete** | Inlined. |
| `snippets/mcp-install-cursor.mdx` | **Delete** | Inlined. |
| `snippets/mcp-install-codex.mdx` | **Delete** | Inlined. |
| `snippets/mcp-install-windsurf.mdx` | **Delete** | Inlined. |
| `snippets/cli-install.mdx` | **Keep** | Used by CLI page; install command stable. Re-audit later — may also need inlining for the same reason. |
| `snippets/skills-install.mdx` | **Keep for now** | Same as cli-install. Track for follow-up. |
| `snippets/auth-header.mdx` | **Keep** | Tiny prose block, not a code command. Lower crawler-impact. |
| `snippets/api-url.mdx` | **Keep** | Same as auth-header. |
| `snippets/code-setup-{js,php,python}.mdx` | **Keep** | Code blocks, but reused across many api-reference pages. **Re-audit:** if these also fail to render in fetched HTML, they need the same treatment. Track as a follow-up issue. |
| `snippets/integration-resources-{mcp,sdk}.mdx` | **Keep** | Card groups; less critical than install commands. |
| `snippets/payment-methods.mdx` | **Keep** | Same. |
| `snippets/webhook-validation.mdx` | **Keep** | Same. |

**Open question Q1:** Should we re-audit `cli-install.mdx`, `skills-install.mdx`, and the `code-setup-*.mdx` snippets in the same pass? Recommendation: **yes** — verify each in fetched HTML; inline any that fail. Cheap to do while we're already in this part of the codebase.

---

## 4. File-by-file changes

### 4.1 Files to modify

#### `guias/mcp-server.mdx`

Current state: 161 lines. Has 4 Mintlify Tabs each containing a `<Snippet>` reference.

Changes:
- **Lines 29–55** (the `<Tabs>` block): replace each `<Snippet file="..." />` with the inline content of that snippet. Result: every Tab self-contains its install command.
- **After line 132** (after "Exemplos de Prompts"): add new section **"Resolução de Problemas"** with 6 troubleshooting entries:
  - Erro `command not found: claude` (Claude Code não instalado)
  - Erro `Invalid API key` (verifique `GARU_API_KEY`)
  - Erro `network timeout` (firewall / proxy corporativo)
  - O agente não vê as ferramentas (cliente MCP não recarregou)
  - Erro `permission denied` ao executar `npx` (ambiente Node desatualizado)
  - O agente cria cobranças no ambiente errado (chave `sk_test_` vs `sk_live_`)
- **Expand example prompts** from 5 to 8, mirroring the marketing page (so search results are coherent). New prompts: per-installment card charge, customer update by ID, monthly revenue summary.
- **Add a "Boas práticas" callout** right after the install Steps: "Use sempre `sk_test_` em desenvolvimento; o MCP não distingue ambientes — quem distingue é a chave."
- Update **"Próximos Passos"** to add a card linking to the long-form changelog article (slug TBD by companion spec): `Tutorial completo: Claude Code + Garu`.

#### `guias/integracoes/claude-code.mdx`

Current line 14: `<Snippet file="snippets/mcp-install-claude-code.mdx" />`.

Replace with the literal content from `snippets/mcp-install-claude-code.mdx`:
```bash
claude mcp add garu -- npx -y @garuhq/mcp
```

Same pattern, same one-time edit.

#### `guias/integracoes/cursor.mdx`, `codex.mdx`, `windsurf.mdx`

Same treatment. Replace each `<Snippet file="snippets/mcp-install-<tool>.mdx" />` with the inlined fenced code block. The inlined content is whatever currently lives in the snippet file — a JSON config for Cursor/Windsurf, a CLI command for Codex.

#### `guias/integracoes/bolt.mdx`, `lovable.mdx`, `replit.mdx`, `v0.mdx` — DELETED

These four are cloud/web-based AI builders that don't support local-stdio MCP today (a browser-based IDE can't spawn `npx @garuhq/mcp` on the user's machine). Keeping the pages would teach a setup that doesn't work and pollute search results.

Action:
- `git rm guias/integracoes/{bolt,lovable,replit,v0}.mdx`
- Remove the four entries from `docs.json` navigation (`Integrações` group).
- Remove the four URLs from `llms.txt`.
- If any `<Snippet>` was referenced only by these four pages, those snippets become orphaned and are deleted with the rest.

If any of these tools later add remote-MCP support (HTTP transport), we'll add a single page "MCP em IDEs na nuvem" pointing to the HTTP transport instructions in `mcp-server.mdx` rather than per-tool pages.

#### `llms.txt`

Currently structured as topic groups. Update to:
- Add the new "Resolução de Problemas" subsection of mcp-server.md to the index.
- Tighten descriptions to be LLM-actionable: "Para conectar Claude Code à API Garu via MCP, leia: https://docs.garu.com.br/guias/integracoes/claude-code.md (contém o comando exato de instalação inline)."
- Make the file's first lines a 3-sentence overview that an LLM can quote when asked "what is Garu MCP".

#### `docs.json`

- Add per-page `og:image` overrides where the Mintlify schema allows. At minimum, set `og:title` and `og:description` overrides for `/guias/mcp-server`.
- Set `metadata.og:locale = "pt_BR"` (already set, verify).
- Verify `seo.indexHiddenPages = false` is correct (it is — keeps API playground out of search).

### 4.2 Files to delete

Snippets (orphaned after inlining):
- `snippets/mcp-install-claude-code.mdx`
- `snippets/mcp-install-cursor.mdx`
- `snippets/mcp-install-codex.mdx`
- `snippets/mcp-install-windsurf.mdx`

Integration pages (cloud IDEs without MCP support):
- `guias/integracoes/bolt.mdx`
- `guias/integracoes/lovable.mdx`
- `guias/integracoes/replit.mdx`
- `guias/integracoes/v0.mdx`

Delete in the same commit so reviewers see the full trade clearly. Update `docs.json` and `llms.txt` in the same commit.

### 4.3 Files to add

- `guias/mcp-server-troubleshooting.mdx` — **alternative**: instead of growing `mcp-server.mdx` past 200 lines, factor the troubleshooting block into its own page added to the `Ferramentas para Agentes` group. Recommendation: **keep inline in mcp-server.mdx** for v1 (one-page tells one story); split later if the page exceeds 250 lines.

---

## 5. Build sequence

1. **Phase 0 — Audit (1 hour).**
   - Open each `<Snippet>` reference in the live docs. View source. Confirm content is missing in static HTML.
   - Verify `bolt/lovable/replit/v0` MCP support actually exists (Q2).
   - Decision points resolved before code.

2. **Phase 1 — Inline (2 hours).**
   - Edit the 9 `.mdx` files (`mcp-server.mdx` + 8 `integracoes/*.mdx`).
   - Delete the 4 orphaned snippets.
   - `prettier --check` (Mintlify projects use prettier per CLAUDE.md G-1).

3. **Phase 2 — Enrich `mcp-server.mdx` (2 hours).**
   - Add troubleshooting section.
   - Expand prompts.
   - Add boas-práticas callout.
   - Update Próximos Passos.

4. **Phase 3 — `llms.txt` + `docs.json` (1 hour).**
   - Tighten llms.txt descriptions.
   - Add metadata overrides.

5. **Phase 4 — Verify (30 min).**
   - Deploy to Mintlify preview.
   - `curl -sL https://<preview>.docs.garu.com.br/guias/integracoes/claude-code | grep "claude mcp add"` returns the command.
   - Spot-check each integration page in the preview UI.

**Total estimate:** ~6.5 hours of focused work. One day end-to-end including review.

---

## 6. Acceptance criteria

- [ ] `curl https://docs.garu.com.br/guias/integracoes/claude-code` returns HTML containing the literal string `claude mcp add garu`.
- [ ] Same check passes for `cursor`, `codex`, `windsurf` integration pages with their respective commands.
- [ ] No `<Snippet>` reference remains in any `guias/integracoes/*.mdx` or in the install Tabs of `guias/mcp-server.mdx`.
- [ ] The four orphaned snippet files are deleted.
- [ ] `guias/mcp-server.mdx` has a "Resolução de Problemas" section with ≥6 entries.
- [ ] Example prompt count on `guias/mcp-server.mdx` is 8 (matches marketing page).
- [ ] `llms.txt` description for `/guias/mcp-server` mentions troubleshooting.
- [ ] Prettier and Mintlify build pass in CI.
- [ ] A blind LLM fetch of any integration page surfaces the install command in the first 50 lines of returned content.

---

## 7. Risks & open questions

### 7.1 Risks

- **Future drift between marketing page, mcp-server.mdx, integration page, and garu-mcp README.** Mitigation: add a one-line "🔄 sync source: garu-mcp README" comment in each .mdx near the install block, naming the upstream. Reviewers know to check upstream when changing commands.
- **`mcp-server.mdx` grows too long with troubleshooting added.** Mitigation: hard cap at 250 lines; if exceeded, factor troubleshooting to its own page (§4.3).
- **Other snippets (`code-setup-*.mdx`, `cli-install.mdx`) likely have the same crawler problem.** This spec scopes only the MCP install snippets. A follow-up issue should re-audit and inline as needed.

### 7.2 Open questions

- **Q1 — Audit other snippets in this pass?** Recommendation: yes, while we're already touching the snippet system. ~1 extra hour.
- ~~**Q2 — Do bolt/lovable/replit/v0 actually support MCP?**~~ **Resolved 2026-04-27:** they don't (cloud IDEs, no local-stdio). All four pages deleted in the same commit (§4.2).
- **Q3 — Should mcp-server.mdx and the `/mcp` marketing page share the FAQ content?** Companion spec keeps FAQ marketing-only; docs has troubleshooting (operational, command-output focused). They serve different intents — FAQ converts skeptics, troubleshooting unblocks active users. Keep separate.

---

## 8. Companion spec

Marketing-side work (the `/mcp` page rebuild + SEO/prerender infrastructure) lives in `/Users/bissuh/Projects/garu-projects/gateway/SPEC.md`. Either spec can ship independently, but the discovery loop (search → marketing → docs → install) only closes cleanly when both are live.
