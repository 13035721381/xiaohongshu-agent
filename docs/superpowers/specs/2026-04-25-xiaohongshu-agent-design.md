# Xiaohongshu AI Agent Design

## Goal

Build a local Xiaohongshu AI agent for discovering high-engagement AI-related posts, rewriting them into original draft content, generating multi-page image cards, and letting the user manually review which drafts are worth publishing.

Version 1 does not auto-publish. It prepares approved publish packages and keeps the publishing interface reserved for later.

## Confirmed Decisions

- Xiaohongshu integration uses `xpzouying/xiaohongshu-mcp`.
- Collection is based on keyword search from Xiaohongshu.
- Ranking uses Top N per keyword, default Top 5.
- Publishing is not automated in version 1.
- Draft output is cover plus multi-page infographic cards.
- Review happens in a local Web console.
- Tech stack is TypeScript full stack.
- AI model access uses an OpenAI-compatible API.
- Model configuration:
  - `OPENAI_MODEL=glm-5.1`
  - `OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1`
  - `OPENAI_API_KEY` stored only in local env files.
- Keywords support both default config files and Web console edits.

## Scope

### In Scope

- Manage keywords and Top N settings.
- Check `xiaohongshu-mcp` connection and login status.
- Search Xiaohongshu notes through `xiaohongshu-mcp`.
- Fetch note details and engagement data where available.
- Deduplicate collected notes by note ID.
- Use AI to classify whether a note is about AI news or AI tools.
- Use AI to summarize and rewrite relevant notes into original Xiaohongshu drafts.
- Generate title, body, tags, and 6-8 infographic card pages.
- Render card pages to local PNG files.
- Let the user approve, reject, or regenerate drafts in the Web console.
- Export approved draft packages for manual publishing.

### Out of Scope For Version 1

- Automatic publishing to Xiaohongshu.
- Bypassing login, captcha, risk controls, or real-name verification.
- Automated commenting, liking, favoriting, or account-growth actions.
- AI image generation through diffusion models or external image models.
- Cloud deployment and multi-user account support.

## Architecture

The system is a local Next.js application with server-side modules for collection, AI processing, storage, and card rendering.

Data flow:

```text
Keyword config
  -> xiaohongshu-mcp search
  -> note details and engagement data
  -> per-keyword Top 5 selection
  -> AI relevance classification
  -> AI summary and rewrite
  -> card script generation
  -> PNG card rendering
  -> local Web review
  -> approved publish package
```

Main components:

- `Web Console`: Local UI for configuration, task control, candidate review, draft review, and approved packages.
- `XhsMcpClient`: Adapter around `xpzouying/xiaohongshu-mcp`. Version 1 exposes `checkLogin()`, `searchNotes()`, and `getNoteDetail()`. It reserves `publishImageNote()` for later.
- `Collection Service`: Runs keyword searches, normalizes MCP responses, deduplicates notes, and stores collection results.
- `AI Pipeline`: Classifies relevance, rewrites content, generates publish copy, and creates card scripts.
- `Card Generator`: Renders structured card scripts into local PNG image files using deterministic templates.
- `Storage`: Persists keywords, collection runs, source notes, AI analyses, drafts, and generated assets.
- `Config`: Loads `.env.local`, default keyword config, and card template config.

## Web Console

### Configuration Page

Shows and edits:

- Keywords.
- Per-keyword Top N, default 5.
- Enabled or disabled state.
- AI model configuration status.
- `xiaohongshu-mcp` endpoint.
- Login status check.

### Collection Runs Page

Starts and monitors collection tasks. A run can move through:

- `pending`
- `searching`
- `fetching_details`
- `analyzing`
- `drafting`
- `rendering`
- `completed`
- `failed`

Users can retry failed keywords or failed notes.

### Candidate Notes Page

Displays collected source notes with:

- Title.
- Author.
- Like, favorite, comment, and share counts where available.
- Published time where available.
- Image previews.
- Original summary.
- Source keyword.
- Relevance result.
- Failure state if detail fetching failed.

Filtering supports keyword, engagement ranking, relevance, and status.

### Draft Review Page

Displays AI output:

- Relevance decision.
- Category: AI news, AI tool, both, or irrelevant.
- Confidence.
- Reasoning summary.
- Key facts.
- Rewritten title.
- Body copy.
- Tags.
- Cover copy.
- Card page previews.

Actions:

- Approve.
- Reject.
- Regenerate copy.
- Regenerate card script.
- Regenerate images.

### Approved Drafts Page

Shows ready-to-publish packages:

- Title.
- Body.
- Tags.
- Generated image paths.
- Source note reference for manual checking.

Version 1 exports local publish packages only. Later versions can call the reserved MCP publish adapter.

## Data Model

The first implementation can use SQLite plus local files under `data/`.

### `keywords`

- `id`
- `keyword`
- `topN`
- `enabled`
- `createdAt`
- `updatedAt`

### `collection_runs`

- `id`
- `status`
- `startedAt`
- `finishedAt`
- `error`
- `settingsSnapshot`

### `source_notes`

- `id`
- `noteId`
- `xsecToken`
- `sourceUrl`
- `sourceKeyword`
- `title`
- `description`
- `authorName`
- `authorId`
- `likeCount`
- `favoriteCount`
- `commentCount`
- `shareCount`
- `publishedAt`
- `imageUrls`
- `rawPayload`
- `collectedAt`
- `detailStatus`

### `ai_analyses`

- `id`
- `sourceNoteId`
- `isRelevant`
- `category`
- `confidence`
- `reason`
- `keyFacts`
- `skipReason`
- `riskFlags`
- `rawModelOutput`
- `createdAt`

### `drafts`

- `id`
- `sourceNoteId`
- `analysisId`
- `status`
- `title`
- `body`
- `tags`
- `coverTitle`
- `coverSubtitle`
- `cardPages`
- `reviewNotes`
- `createdAt`
- `updatedAt`

Draft statuses:

- `classified`
- `drafted`
- `rendered`
- `approved`
- `rejected`
- `needs_review`
- `failed`

### `draft_assets`

- `id`
- `draftId`
- `pageIndex`
- `filePath`
- `width`
- `height`
- `templateName`
- `createdAt`

## AI Pipeline

The AI pipeline is split into three structured calls. Each call should request JSON output and validate the result before storing it.

### 1. Relevance Classification

Input:

- Source title.
- Source description.
- OCR text if available in later versions.
- Engagement metadata.

Output:

- `isRelevant`
- `category`
- `confidence`
- `reason`
- `keyFacts`
- `skipReason`
- `riskFlags`

Rules:

- Relevant content must be about AI news, AI products, AI tools, AI workflows, or meaningful AI industry changes.
- Generic productivity content without a clear AI angle is not relevant.
- Low confidence results become `needs_review`.

### 2. Summary And Rewrite

Input:

- Source note.
- Classification result.
- Key facts.

Output:

- Xiaohongshu title under 20 Chinese characters where possible.
- Body under 1000 Chinese characters.
- 5-8 tags.
- Original angle.
- Source-check notes for the reviewer.

Rules:

- Do not copy original phrasing directly.
- Do not invent facts not present in the source.
- Prefer a new structure: hook, context, key points, user value, caution or recommendation.
- Avoid exaggerated claims, guaranteed outcomes, and hard selling.

### 3. Card Script Generation

Input:

- Rewritten draft copy.
- Selected template style.

Output:

- 6-8 card pages.
- Each page includes title, subtitle or short sentence, bullet points, and visual emphasis.

Default page structure:

1. Cover.
2. Background or pain point.
3. Core news or tool summary.
4. Key highlights.
5. Practical usage or impact.
6. Suitable audience.
7. Recommendation or conclusion.
8. Optional checklist or action step.

## Card Generation

Version 1 uses deterministic HTML/SVG/Canvas rendering to PNG. It does not require AI image generation.

Defaults:

- Size: `1242x1660`.
- Page count: 6-8.
- Output path: `data/assets/<draftId>/page-01.png`.
- Initial templates:
  - `clean-tech`
  - `bold-rednote`

Card constraints:

- Keep text density low.
- Use large title hierarchy.
- Keep each page focused on one idea.
- Avoid visual clutter and tiny text.
- Preserve consistent colors and typography within one draft.

## MCP Integration

`xpzouying/xiaohongshu-mcp` remains an external dependency. The project integrates through a small adapter to avoid coupling the application to MCP-specific response details.

Version 1 adapter methods:

- `checkLogin(): Promise<LoginStatus>`
- `searchNotes(keyword, options): Promise<SearchResult[]>`
- `getNoteDetail(noteId, xsecToken): Promise<NoteDetail>`

Reserved method:

- `publishImageNote(payload): Promise<PublishResult>`

Important behavior:

- If search results include enough engagement data, Top N selection can run before detail fetching.
- If engagement data is incomplete, fetch details for a larger candidate set before sorting.
- Missing `xsec_token` marks a note as `failed_detail`.
- Duplicate notes are merged by `noteId`, with multiple source keywords recorded.

## Configuration

`.env.local`:

```text
OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
OPENAI_MODEL=glm-5.1
OPENAI_API_KEY=your-local-key
XHS_MCP_MODE=http
XHS_MCP_HTTP_URL=http://localhost:8000
# If stdio mode is selected later:
# XHS_MCP_MODE=stdio
# XHS_MCP_COMMAND=/absolute/path/to/xiaohongshu-mcp
```

`config/keywords.json`:

```json
{
  "defaultTopN": 5,
  "keywords": [
    { "keyword": "AI工具", "topN": 5, "enabled": true },
    { "keyword": "AI资讯", "topN": 5, "enabled": true }
  ]
}
```

`config/card-templates.json`:

```json
{
  "defaultTemplate": "clean-tech",
  "templates": ["clean-tech", "bold-rednote"]
}
```

`.gitignore` should include:

```text
.env*
data/
cookies/
.superpowers/
```

## Error Handling

- MCP not logged in: pause collection and ask the user to complete login.
- MCP connection failure: mark the run as failed and show setup guidance.
- Search failure: keep the run alive, mark that keyword failed, and allow retry.
- Missing `xsec_token`: mark note as `failed_detail`.
- Detail fetch failure: retry once, then store the error.
- AI non-JSON response: retry once with a stricter repair prompt.
- AI low confidence: mark draft as `needs_review`.
- Card rendering failure: allow image-only regeneration.
- Duplicate note: merge instead of creating duplicate drafts.

## Compliance And Content Safety

- The system does not bypass platform security mechanisms.
- The system does not auto-publish in version 1.
- The system keeps source references for manual verification.
- Generated content must be rewritten in an original structure.
- Basic checks flag:
  - exaggerated claims,
  - absolute promises,
  - direct traffic diversion language,
  - sensitive or risky words,
  - content that looks like pure copying.

## Testing Strategy

### Unit Tests

- Engagement count parsing.
- Top N sorting.
- Deduplication.
- AI JSON parsing and repair.
- Draft status transitions.
- Card page validation.

### Integration Tests

- Mock MCP search and detail responses.
- Run keyword collection through AI mock outputs.
- Generate draft assets from sample card scripts.
- Verify retry and failure states.

### Manual End-to-End Tests

- Start local console.
- Confirm MCP login status.
- Run one keyword with Top 5.
- Review collected notes.
- Generate one draft.
- Regenerate copy and images.
- Approve one draft.
- Confirm exported package contains title, body, tags, and images.

Real Xiaohongshu requests should not be part of automated CI because they depend on login state and platform behavior.

## Implementation Phases

### Phase 1: Local App Skeleton

- Create the Next.js TypeScript app.
- Add local config loading.
- Add SQLite schema.
- Add Web console shell pages.

### Phase 2: MCP Adapter And Collection

- Add `XhsMcpClient`.
- Implement login status check.
- Implement keyword search.
- Implement note detail normalization.
- Store collection runs and source notes.

### Phase 3: AI Pipeline

- Add OpenAI-compatible client.
- Implement classification prompt.
- Implement rewrite prompt.
- Implement card script prompt.
- Store structured AI outputs.

### Phase 4: Card Rendering

- Add deterministic card templates.
- Render PNG pages.
- Store generated assets.
- Preview cards in the console.

### Phase 5: Review Workflow

- Add approve, reject, and regenerate actions.
- Add approved draft export.
- Add error views and retry controls.

## Open Questions For Implementation Planning

- The first implementation should attempt HTTP mode first. If the selected `xpzouying/xiaohongshu-mcp` setup only exposes stdio in the user's local environment, switch the adapter to stdio mode behind the same `XhsMcpClient` interface.
- The first card renderer library should be selected during implementation planning based on Windows compatibility.
- If `xiaohongshu-mcp` search results do not include reliable engagement counts, the collection service should over-fetch details before selecting Top 5.
