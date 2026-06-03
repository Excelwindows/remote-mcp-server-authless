# Salesforce File-Read — Engineering Spec

**Lane:** Engineering / PAPI · **Target:** Excel-MCP Worker (Cloudflare, API v62, JWT Bearer Flow → improveit360-2900) · **Date:** 2026-06-02

## 0. Why this exists
The door-ordering system (and "look at the pictures") needs the **measure sheet**, the **signed contract PDF**, and **measure photos** that live as Salesforce *Files/Attachments* on the Prospect / Project / Sale. The current MCP surface (`get_sale`, `get_project`, `get_prospect`) returns **zero** file references — there is no read path for SF binaries. This spec adds it. It also closes the long-open Engineering carry-forward `create_prospect_attachment` / **"photo-lake."**

Downstream consumers:
- **Auto-draft** (entryLINK `createorder`) — extract door spec from contract/measure → property bag.
- **Watch-outs email to Chris** — reconcile measure ↔ contract; flag capping/hardware/handing.
- **Sold-vs-ordered audit** + a real order-error rate (today only inferable from the 47% rework signal).

## 1. Salesforce file model (support BOTH storage modes)
i360 orgs mix two mechanisms; we must handle both because the measure (newer, photo uploads) and the scanned contract (older workflow) may live in different ones.

**A. Modern Files (Content*)**
- `ContentDocument` — file header (`Id`, `Title`, `FileExtension`, `FileType`, `ContentSize`, `LatestPublishedVersionId`).
- `ContentVersion` — the versioned binary (`Id`, `ContentDocumentId`, `VersionData` = the bytes, `Title`, `FileExtension`, `ContentSize`, `CreatedDate`).
- `ContentDocumentLink` (**CDL**) — junction: `ContentDocumentId`, `LinkedEntityId` (= the Prospect/Project/Sale/Activity Id), `ShareType`, `Visibility`.

**B. Legacy Attachments**
- `Attachment` — `Id`, `ParentId` (= the record), `Name`, `ContentType`, `BodyLength`, `Body` (the bytes), `CreatedDate`.

```
Prospect/Project/Sale/Activity (LinkedEntityId / ParentId)
        │
        ├─ ContentDocumentLink ─→ ContentDocument ─→ ContentVersion.VersionData  (Files)
        └─ Attachment.Body                                                        (legacy)
```

## 2. Where door files attach (DISCOVERY — do first)
Unknown until probed; the `Scanned Contract` + `Measure` Project Activities imply files hang off the **Project** and/or **Prospect** (and possibly the Activity record). One-time discovery on a known clean deal:

- **McMurray** — Project `a0XPk000005O0onMAC` (PRJ31334), Prospect `a0ZPk00000IzlFlMAJ`, Sale `a0fPk00000TuAs1IAF`.

Run (Cowork SOQL now, or via the new tool once built):
```sql
SELECT LinkedEntityId, ContentDocument.Title, ContentDocument.FileExtension,
       ContentDocument.ContentSize, ContentDocument.LatestPublishedVersionId
FROM ContentDocumentLink
WHERE LinkedEntityId IN ('a0XPk000005O0onMAC','a0ZPk00000IzlFlMAJ','a0fPk00000TuAs1IAF');

SELECT Id, ParentId, Name, ContentType, BodyLength
FROM Attachment
WHERE ParentId IN ('a0XPk000005O0onMAC','a0ZPk00000IzlFlMAJ','a0fPk00000TuAs1IAF');
```
Deliverable of discovery: which object holds the contract vs the measure vs photos, and the title conventions (so `get_door_packet` can classify).

## 3. Permissions — the gotcha (Alex/Cowork action)
`ContentDocumentLink` can only be **queried for documents shared with the running user** unless that user has the **`QueryAllFiles`** system permission (or `View All Data`). The integration user **`mcpapi`** (`005Pk00000Gemv2IAR`) will likely **not** see files uploaded/shared by other staff without it.

- **Grant `Query All Files`** to `mcpapi` via a permission set (preferred over View All Data — least privilege). Read-only.
- Legacy `Attachment` follows **parent-record sharing**; `mcpapi` already reads Prospect/Project/Sale, so Attachments should be reachable — confirm.
- **Test:** as `mcpapi`, query CDL for a record it does **not** own; expect rows only after the grant.
- Per lane history: do not assume a perms wall; verify with a throwaway query first (the "Insufficient Privileges" class here has been environmental before).

## 4. New MCP tools

### 4.1 `list_record_files(recordId, includeFiles=true, includeAttachments=true)`
Lists everything attached to a record. No binaries.
- Files: query CDL → ContentDocument fields above.
- Attachments: query `Attachment` by `ParentId`.
- **Output** — unified descriptors:
```json
{ "files": [
  { "source":"File", "fileId":"<ContentVersionId>", "docId":"<ContentDocumentId>",
    "title":"Measure - McMurray", "ext":"pdf", "mime":"application/pdf",
    "sizeBytes":824133, "createdDate":"2026-03-27", "linkedTo":"a0XPk000005O0onMAC" },
  { "source":"Attachment", "fileId":"<AttachmentId>", "title":"Signed Contract",
    "ext":"pdf", "mime":"application/pdf", "sizeBytes":1240050, "createdDate":"2026-03-03",
    "linkedTo":"a0XPk000005O0onMAC" }
]}
```

### 4.2 `get_file_content(source, fileId, mode)`
Fetches one file. `source ∈ {File, Attachment}`.
- Binary endpoints (GET, auth bearer, octet-stream — do **not** JSON-parse):
  - File: `/services/data/v62.0/sobjects/ContentVersion/{fileId}/VersionData`
  - Attachment: `/services/data/v62.0/sobjects/Attachment/{fileId}/Body`
- **`mode`:**
  - `metadata` — descriptor only.
  - `base64` — inline base64. **Hard cap (≤ ~4 MB)**; reject larger (McMurray photos were 5 MB PNGs — inlining those would blow the MCP payload). For small PDFs/sheets only.
  - `r2` — Worker **streams** the bytes into Cloudflare **R2** (`excel-photo-lake/{prospectId}/{docId}.{ext}`), returns `{ key, url, sizeBytes, mime }`. Default for images/large files. This *is* the photo-lake.
  - `extract` *(Phase 2)* — server-side: PDF→text (pdf parser) or image→OCR/vision → returns structured text/fields. Feeds the auto-draft + watch-outs.

### 4.3 `get_door_packet(projectId)` (composite — what the ordering flow calls)
- Resolves the door Project + parent Prospect + Sale + the `Measure`/`Scanned Contract` activities; gathers all files across them.
- Classifies each by title/type heuristics → `measure_sheet | contract | photo | other` (heuristics finalized from §2 discovery).
- Pushes each to R2; returns the packet: refs + R2 URLs + classification. One call = everything the order needs.

## 5. Storage: the photo-lake (Cloudflare R2)
- New R2 bucket bound to the Worker (e.g. binding `PHOTO_LAKE`).
- Key: `excel-photo-lake/{prospectId}/{contentDocumentId}.{ext}`; dedup by ContentDocumentId/VersionId.
- Rationale: keep multi-MB binaries **out of** MCP responses; give stable URLs for (a) Chris's email, (b) vision/OCR extraction, (c) audit. Cheap, no egress fees.
- Optional later: lifecycle/expiry, virus-scan hook, thumbnailing.

## 6. Worker implementation notes
- **Reuse** existing auth: JWT Bearer Flow, ~50-min token cache, API v62, org base URL. No new auth.
- **New binding:** R2 `PHOTO_LAKE`.
- **Binary handling:** SF returns `application/octet-stream` for VersionData/Body — fetch with bearer, **stream `response.body` directly into `R2.put()`** (do not buffer whole file in Worker memory; matters for 5 MB+ images).
- **base64 path:** only after size check; `arrayBuffer()` → base64; enforce cap.
- **No phantom-field risk:** ContentDocument/ContentVersion/ContentDocumentLink/Attachment are **standard** objects — stable schema, but still run the describe/smoke gate before ship (the org has bitten us on field-existence before).
- **Error modes:** `INSUFFICIENT_ACCESS` on CDL (→ perms §3), file-too-large for base64, missing/var file, unsupported type.

## 7. Validation / smoke plan (mandatory per-tool functional smoke)
1. **Discovery** (§2) — confirm attachment points + perms for `mcpapi`.
2. `list_record_files(PRJ31334)` → returns contract + measure + photos.
3. `get_file_content(... , r2)` on the measure sheet → R2 URL renders the PDF/image.
4. `get_file_content(... , base64)` on a small file → caps enforced.
5. *(Phase 2)* `extract` on a measure sheet → door count / sizes / handing come back as text.
6. Throwaway perms test: CDL query as `mcpapi` for a non-owned record.

## 8. Phasing
- **P0** — `list_record_files` + `get_file_content(metadata|base64-small)`. Proves read + perms. Smallest shippable.
- **P1** — R2 streaming + `get_door_packet`. The photo-lake + the ordering-flow input.
- **P2** — `extract` (PDF text + vision OCR → structured door spec). Feeds entryLINK property bag + watch-outs.

## 9. Dependencies / asks
- **Alex/Cowork:** grant `mcpapi` the **Query All Files** permission set (read-only). One admin action; gates everything.
- **Engineering (Code):** provision R2 bucket + binding; build P0 tools; wrangler schema/smoke gate.
- **Discovery:** §2 SOQL to locate contract vs measure vs photos + title conventions.
- **Ties to entryLINK spec:** the `extract` output (door spec) becomes the entryLINK property-bag input; sequence P2 alongside the entryLINK mapping (tomorrow's ProVia access).

## 10. Open questions for Alex
1. OK to grant `mcpapi` **Query All Files** (read-only)? (vs. case-by-case sharing — not scalable.)
2. R2 as the photo-lake store — confirmed? (alternative: SharePoint/OneDrive, but R2 is native to the Worker and cheapest.)
3. Retention: keep pulled measure/contract files in R2 indefinitely, or TTL after the order is submitted?
