# Phase 5: Asset, Document, And Storage Review

## Scope

This artifact reviews where tenant content actually lives, how Paperclip names and reads stored objects, and whether uploaded files or inline documents can become a data-exposure or active-content problem.

- Primary question: whether storage-provider safety, upload validation, and content-read routes prevent tenant-data leakage and unsafe file execution
- Boundary lens: separate provider-level path safety from route-level authorization and browser-facing content handling
- Principal focus: same-company board users and agents who can upload or read assets, attachments, and issue documents

## Storage Provider And Object-Key Model

The underlying storage layer is structurally sound.

`server/src/storage/service.ts` creates a clear company-prefixed namespace.

- `buildObjectKey(...)` prefixes every object key with `companyId/`
- namespace and filename segments are sanitized before they become part of the object key
- `ensureCompanyPrefix(...)` rejects cross-company object access and blocks `..` segments

The provider implementations preserve that boundary.

- `server/src/storage/local-disk-provider.ts` normalizes object keys, rejects absolute paths and `.` or `..` segments, and resolves files within the configured storage root
- `server/src/storage/s3-provider.ts` only adds an optional configured prefix before delegating to S3
- `server/src/storage/provider-registry.ts` switches between `local_disk` and `s3` without weakening the company-prefix model

This is a real positive control: straightforward path traversal or object-key escape is not the main storage risk here.

## Asset And Attachment Surfaces

The storage backends are not the weak point. The browser-facing content policy is.

Upload routes:

- `server/src/routes/assets.ts` exposes `POST /api/companies/:companyId/assets/images`
- `server/src/routes/issues.ts` exposes `POST /api/companies/:companyId/issues/:issueId/attachments`
- both routes only require `assertCompanyAccess(...)`
- both routes trust `file.mimetype` from the multipart upload and gate only on `isAllowedContentType(contentType)`

The default content-type allowlist is much broader than "images".

`server/src/attachment-types.ts` includes these defaults:

- image types
- `application/pdf`
- `text/markdown`
- `text/plain`
- `application/json`
- `text/csv`
- `text/html`

The UI reinforces that behavior for issue attachments.

- `ui/src/pages/IssueDetail.tsx` uses an `<input type="file">` with `accept="...text/html..."`
- `ui/src/components/NewIssueDialog.tsx` defines the same staged file accept string including `text/html`

Content reads are served as active same-origin content.

- `server/src/routes/assets.ts` returns `/api/assets/:assetId/content`
- `server/src/routes/issues.ts` returns `/api/attachments/:attachmentId/content`
- both routes set `Content-Type` from the stored MIME type
- both set `Content-Disposition: inline`
- both serve under the main app origin and only add `Cache-Control: private, max-age=60`
- no app-level CSP or `X-Content-Type-Options: nosniff` protection was found in the server startup or routing path

The UI links directly to attachment content.

- `ui/src/pages/IssueDetail.tsx` renders attachment names as `<a href={attachment.contentPath} target="_blank" rel="noreferrer">...`
- image attachments are previewed inline, while non-image attachments still get a direct same-origin open path

This means a same-company actor can upload HTML, receive a stable `contentPath`, and cause a board user to open active script content under the Paperclip origin.

## Document And Revision Persistence

Issue documents behave differently from attachments.

- `packages/db/src/schema/documents.ts` stores `latestBody` inline in Postgres
- `packages/db/src/schema/document_revisions.ts` stores every revision body inline as well
- `server/src/services/documents.ts` implements append-only revision history and optimistic concurrency via `baseRevisionId`
- `server/src/routes/issues.ts` exposes list, get, upsert, revision-list, and delete routes for issue documents under same-company access

That has two important implications.

First, document content is included in DB backups and seeded worktrees even when attachment bytes live in object storage.

Second, document bodies are not an immediate HTML execution sink in the current UI.

- `ui/src/components/IssueDocumentsSection.tsx` renders document bodies through `MarkdownBody`
- `ui/src/components/MarkdownBody.tsx` uses `react-markdown` with `remark-gfm`
- it does not enable `rehypeRaw`, so raw HTML in markdown is not interpreted as live DOM

So documents are a confidentiality and persistence concern, but not the same active-content problem as attachments and direct assets.

Broad same-company visibility for issue content is also currently intentional in the product contract.

- `doc/SPEC-implementation.md` states "Full visibility to board and all agents in same company"
- routes such as `/issues/:id`, `/issues/:id/documents`, and `/issues/:id/attachments` match that company-wide visibility model

That means broad same-company access to issue documents or attachment metadata is not, by itself, a Phase 5 authorization bug under the implemented V1 contract.

## Findings Or Review Leads

- **Confirmed vulnerability / High:** Paperclip allows stored same-origin HTML execution through asset and attachment content routes. The default allowlist includes `text/html`, upload routes trust the client-supplied MIME type, content is served from `/api/assets/:assetId/content` and `/api/attachments/:attachmentId/content` with `Content-Type` preserved and `Content-Disposition: inline`, and the UI exposes direct links to those URLs. A same-company actor can therefore upload HTML and induce a board user to open it, executing attacker-controlled script with the Paperclip origin and board session context.
- **No finding:** The storage-provider layer itself resists straightforward path traversal and cross-company object-key abuse. Company prefixes are enforced centrally, and the local-disk provider constrains resolved paths to the configured storage root.
- **No finding:** Issue documents are stored inline in Postgres and therefore widen backup scope, but the current UI renders them through `react-markdown` without raw HTML execution.
- **Accepted-risk sharp edge / Informational:** Broad same-company visibility for issue documents and attachments matches the stated V1 contract, so the main storage problem is unsafe active content, not hidden extra authorization drift on ordinary reads.

## Later Phase Questions

- Phase 7 should evaluate whether the cleanest fix is removing `text/html` from the default allowlist, forcing `attachment` downloads for non-image types, or serving user content from an isolated origin.
- Phase 7 should decide whether the direct asset route should enforce actual image content rather than reusing the broader attachment allowlist under an `/assets/images` path.
- Phase 6 may still want an operational note about object-store lifecycle, retention, and external bucket hardening, but the main code-level weakness is already in the HTTP content policy rather than provider path safety.
