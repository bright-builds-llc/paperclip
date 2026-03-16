---
phase: 05-secrets-storage-data-exposure
plan: "02"
subsystem: security-docs
tags: [assets, attachments, documents, storage, xss]
requires: []
provides:
  - Asset, document, and storage review
  - Confirmed stored-XSS finding for HTML uploads
affects: [phase-05, phase-07]
tech-stack:
  added: []
  patterns: [provider-vs-route review, active-content analysis]
key-files:
  created:
    - .planning/phases/05-secrets-storage-data-exposure/05-ASSET-DOCUMENT-STORAGE-REVIEW.md
  modified: []
key-decisions:
  - "Treat the provider path checks as a separate question from browser-facing content safety so path traversal and stored XSS are not conflated."
  - "Use the V1 same-company visibility contract to avoid mislabeling broad issue reads as an auth bug when the real issue is active HTML content."
patterns-established:
  - "Compare object-store safety with HTTP content policy before assigning storage severity."
  - "Check UI upload affordances and content links, not only backend routes, when evaluating stored active content."
requirements-completed: [DATA-02]
duration: 17min
completed: 2026-03-16
---

# Phase 5: Asset, Document, And Storage Summary

**Confirmed the main storage vulnerability in this phase: Paperclip accepts HTML uploads and then serves them inline from the app origin, creating a stored XSS path through ordinary attachment and asset links.**

## Performance

- **Duration:** 17 min
- **Started:** 2026-03-16T17:40:00Z
- **Completed:** 2026-03-16T17:57:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Traced object-key construction and provider-level path protections for both local-disk and S3-backed storage.
- Reviewed asset and attachment uploads, default MIME-type policy, same-origin content serving, and the actual board UI links that open uploaded content.
- Confirmed that default `text/html` uploads plus `inline` content serving produce a stored same-origin HTML execution path.
- Distinguished inline Postgres-backed issue documents from uploaded files and confirmed that current document rendering does not execute raw HTML.

## Task Commits

This plan is included in the consolidated Phase 5 completion commit.

## Files Created/Modified

- `.planning/phases/05-secrets-storage-data-exposure/05-ASSET-DOCUMENT-STORAGE-REVIEW.md` - Storage provider, upload, document, and content-read review

## Decisions Made

- Assigned the primary confirmed finding to same-origin HTML content handling rather than to object-store or filesystem path safety.
- Kept broad same-company issue visibility out of the finding list because the current V1 contract explicitly grants that visibility.

## Deviations from Plan

- None

## Issues Encountered

- The route named `/assets/images` reuses the broader attachment MIME policy, so the artifact had to call out the naming mismatch as part of the stored-content analysis.

## User Setup Required

- None
