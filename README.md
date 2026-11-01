# YOUSIVAR — UI

A cinematic, bilingual (Arabic / English) AI workspace: persistent conversations, multi-model
routing, side-by-side model comparison, private document attachments, user-controlled memory,
hashed API keys and an administrative audit trail.

Brand assets: `yousivar.png` (identity) and `grpwce.mp4` (hero film), referenced from the
welcome surface with autoplay / muted / playsinline / loop, a dark reading scrim, and a static
poster fallback when the video cannot load.

---

## 1. What actually runs

| Layer | Implementation |
|---|---|
| Frontend | React 18 + Vite, Tailwind, React Router, react-markdown |
| Auth | Platform authentication (email/password, Google, OTP, reset) — routes `/login`, `/register`, `/forgot-password`, `/reset-password` |
| Database | Platform entity store (7 entities, row-level security per record owner) |
| Server logic | Backend functions `chatCompletion`, `createApiKey`, `adminStats` |
| Model layer | `base44/shared/models.ts` — the single registry of routing keys + real capability flags |
| Private files | `UploadPrivateFile`, signed on demand inside the server function |
| Mobile | iOS / Android published from this same codebase by the platform |

### Data model (`base44/entities`)

`Conversation`, `Message`, `AIModel`, `Memory`, `Document`, `ApiKey`, `AuditLog`.

Every user-owned entity is locked to its creator (`created_by_id`); `AIModel` is publicly
readable but admin-writable; `AuditLog` is readable only by the `admin` role.

### Server functions

- **chatCompletion** — validates model key, message count/length, memory count and document
  URIs (storage URIs only, never caller URLs), signs at most 4 app-owned files, injects
  user-approved memory and document names, calls the platform LLM integration, writes an audit
  record. 401 / 400 / 500 boundaries, no secrets on the client.
- **createApiKey** — generates `ysv_<48 hex>`, stores only the SHA-256 hash plus an 11-char
  prefix, returns the full key exactly once, caps active keys at 10.
- **adminStats** — verifies `role === 'admin'` server-side before any service-role read.

---

## 2. Model registry

| Key | Files | Web grounding |
|---|---|---|
| `automatic` | yes | yes |
| `gemini_3_8_flash` | yes | yes |
| `gpt_6_luna` | yes | no |
| `gpt_6_sol` | yes | no |
| `claude-sonnet-5` | yes | no |
| `claude_opus_5_5` | yes | no |
| `base1_5_1` | yes | no |
| `glm_5_2` | **no** | no |

Capability flags are enforced, not decorative: asking for web search or documents on a model
without support returns a 400 instead of silently degrading. Admins can hide registry entries
from every picker; the registry itself is code, so a forged key cannot add a provider.

---

## 3. Local development

```bash
npm install
npm run dev      # Vite dev server
npm run build    # production bundle
```

Environment and secrets live in the platform project settings — never in the repository. There
are no provider API keys to add today: inference runs through the platform integration, so no
third-party secret is required for chat.

---

## 4. Deployment

Publish from the editor (Publish button) to get the hosted URL, custom domain, SEO settings and
the native iOS/Android builds. Database migrations are implicit: entity schemas in
`base44/entities/*.jsonc` are the migration.

### Cloudflare mapping (self-hosted target)

The project brief targets Cloudflare. This build runs on the platform's equivalent services; the
portable pieces are the data model, the function contracts and the model registry.

| Brief | Here | Port path |
|---|---|---|
| D1 | entity store | SQL schema from the entity files |
| R2 | private file storage | `UploadPrivateFile` → R2 bucket binding |
| Vectorize | not implemented (documents are passed to the model) | add `document_chunks` + embedding job |
| Workers AI / AI Gateway | platform LLM integration | same function body, different adapter |
| Queues | none needed (no async ingestion yet) | add for chunk + embed |
| Capacitor | platform native publishing | add Capacitor projects for self-hosted |

---

## 5. Security posture

- Authorization on every record via RLS (owner-anchored, admin-elevated where intended).
- Admin rollup is a server function, not a client-side role check.
- Document URIs are validated to app-owned storage and signed server-side for 15 minutes —
  arbitrary URLs cannot be submitted (SSRF guard).
- API keys are stored hashed; the plaintext exists only in the creation response.
- Model output renders through react-markdown without raw HTML, so untrusted markdown cannot
  inject script.
- Rate/abuse limits at the platform edge; per-call input bounds in the function (≤40 messages,
  ≤30 000 characters, ≤20 memories, ≤4 documents).

---

## 6. Known limitations (read before judging completeness)

1. **Responses are not token-streamed.** The model call returns one completion; the UI shows a
   thinking state and renders the answer once. True SSE streaming needs the self-hosted gateway.
2. **No vector retrieval.** Attached documents are sent with the prompt (signed URL) and cited by
   name. There is no chunking, embedding or Vectorize index.
3. **No public OpenAI-compatible endpoint.** `/v1/chat/completions` semantics, SSE framing and
   error shapes are a self-hosted deliverable; keys here are issued and hashed for your own
   integrations.
4. **No Capacitor project.** Native apps are built by the platform, not by a local Gradle/Xcode
   pipeline in this repository.
5. **No automated test suite in this repository.** The chat function is exercised against the live
   model integration during verification; there are no unit/e2e runners wired up.
6. Cloudflare bindings, D1 migrations, R2 buckets and AI Gateway config do not exist here by
   design — see the mapping table instead of expecting `wrangler.jsonc`.

---

## 7. License

MIT — see `LICENSE`.