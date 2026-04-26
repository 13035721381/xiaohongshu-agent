# Xiaohongshu Agent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local TypeScript Xiaohongshu AI agent that searches AI-related notes through `xpzouying/xiaohongshu-mcp`, ranks Top 5 per keyword, generates AI rewritten drafts and multi-page image cards, and lets the user review/export drafts without auto-publishing.

**Architecture:** Use a local Next.js app as the Web console and API host. Keep domain logic in focused server modules under `src/server`, persist app data in SQLite plus local asset files under `data/`, and isolate Xiaohongshu access behind `XhsMcpClient` so HTTP and stdio transport can be swapped without changing collection logic.

**Tech Stack:** Next.js App Router, TypeScript, Vitest, SQLite via `better-sqlite3`, Zod, OpenAI-compatible HTTP client, `sharp` for deterministic SVG-to-PNG card rendering.

---

## File Structure

- Create: `package.json` - scripts and dependencies for the local app.
- Create: `tsconfig.json` - strict TypeScript config.
- Create: `next.config.mjs` - Next.js config.
- Create: `vitest.config.ts` - test config.
- Create: `.gitignore` - ignores env files, local data, and generated assets.
- Create: `.env.example` - non-secret environment template.
- Create: `config/keywords.json` - default keyword config.
- Create: `config/card-templates.json` - default card template config.
- Create: `src/server/config/env.ts` - validates environment variables.
- Create: `src/server/config/keywords.ts` - loads default keyword config.
- Create: `src/server/db/schema.ts` - SQLite schema creation SQL.
- Create: `src/server/db/client.ts` - SQLite connection and migration runner.
- Create: `src/server/types.ts` - shared domain types.
- Create: `src/server/mcp/xhs-client.ts` - `xiaohongshu-mcp` adapter.
- Create: `src/server/collection/normalize.ts` - normalizes MCP search/detail payloads.
- Create: `src/server/collection/ranking.ts` - engagement parsing and Top N ranking.
- Create: `src/server/collection/service.ts` - orchestrates keyword collection.
- Create: `src/server/ai/json.ts` - extracts and validates AI JSON.
- Create: `src/server/ai/client.ts` - OpenAI-compatible chat client.
- Create: `src/server/ai/pipeline.ts` - classify, rewrite, and card-script calls.
- Create: `src/server/cards/templates.ts` - deterministic card template definitions.
- Create: `src/server/cards/render.ts` - SVG-to-PNG renderer.
- Create: `src/server/drafts/export.ts` - approved draft package export.
- Create: `src/app/layout.tsx` - app shell.
- Create: `src/app/page.tsx` - dashboard.
- Create: `src/app/settings/page.tsx` - keyword/config page.
- Create: `src/app/runs/page.tsx` - collection run monitor.
- Create: `src/app/notes/page.tsx` - candidate notes list.
- Create: `src/app/drafts/page.tsx` - draft review list.
- Create: `src/app/approved/page.tsx` - approved draft export page.
- Create: `src/app/api/health/route.ts` - config/MCP health endpoint.
- Create: `src/app/api/keywords/route.ts` - keyword read/update endpoint.
- Create: `src/app/api/runs/route.ts` - start/list collection runs.
- Create: `src/app/api/notes/route.ts` - list candidate notes.
- Create: `src/app/api/drafts/route.ts` - list drafts.
- Create: `src/app/api/drafts/[id]/actions/route.ts` - approve/reject/regenerate/export actions.
- Test: `tests/server/config.test.ts`
- Test: `tests/server/ranking.test.ts`
- Test: `tests/server/normalize.test.ts`
- Test: `tests/server/ai-json.test.ts`
- Test: `tests/server/cards.test.ts`
- Test: `tests/server/export.test.ts`

## Task 1: Project Skeleton

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `next.config.mjs`
- Create: `vitest.config.ts`
- Create: `.gitignore`
- Create: `.env.example`
- Create: `config/keywords.json`
- Create: `config/card-templates.json`

- [ ] **Step 1: Create `package.json`**

```json
{
  "name": "xiaohongshu-agent",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "@types/better-sqlite3": "latest",
    "better-sqlite3": "latest",
    "next": "latest",
    "openai": "latest",
    "react": "latest",
    "react-dom": "latest",
    "sharp": "latest",
    "zod": "latest"
  },
  "devDependencies": {
    "@types/node": "latest",
    "@types/react": "latest",
    "@types/react-dom": "latest",
    "typescript": "latest",
    "vitest": "latest"
  }
}
```

- [ ] **Step 2: Create TypeScript and Next config**

Write `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "ES2022"],
    "allowJs": false,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

Write `next.config.mjs`:

```js
/** @type {import('next').NextConfig} */
const nextConfig = {};

export default nextConfig;
```

Write `vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    include: ["tests/**/*.test.ts"]
  },
  resolve: {
    alias: {
      "@": new URL("./src", import.meta.url).pathname
    }
  }
});
```

- [ ] **Step 3: Create local config files**

Write `.gitignore`:

```text
node_modules/
.next/
out/
.env*
data/
cookies/
.superpowers/
```

Write `.env.example`:

```text
OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
OPENAI_MODEL=glm-5.1
OPENAI_API_KEY=local-test-key-not-for-commit
XHS_MCP_MODE=http
XHS_MCP_HTTP_URL=http://localhost:8000
```

Write `config/keywords.json`:

```json
{
  "defaultTopN": 5,
  "keywords": [
    { "keyword": "AI工具", "topN": 5, "enabled": true },
    { "keyword": "AI资讯", "topN": 5, "enabled": true }
  ]
}
```

Write `config/card-templates.json`:

```json
{
  "defaultTemplate": "clean-tech",
  "templates": ["clean-tech", "bold-rednote"]
}
```

- [ ] **Step 4: Install dependencies**

Run: `npm install`

Expected: `node_modules` and `package-lock.json` are created without dependency resolution errors.

- [ ] **Step 5: Verify skeleton**

Run: `npm test`

Expected: Vitest exits successfully with no tests found or an empty test suite message.

- [ ] **Step 6: Commit**

```bash
git add package.json package-lock.json tsconfig.json next.config.mjs vitest.config.ts .gitignore .env.example config/keywords.json config/card-templates.json
git commit -m "chore: scaffold xiaohongshu agent project"
```

## Task 2: Config And Domain Types

**Files:**
- Create: `src/server/types.ts`
- Create: `src/server/config/env.ts`
- Create: `src/server/config/keywords.ts`
- Test: `tests/server/config.test.ts`

- [ ] **Step 1: Write failing config tests**

Write `tests/server/config.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { parseEnv } from "../../src/server/config/env";
import { parseKeywordConfig } from "../../src/server/config/keywords";

describe("parseEnv", () => {
  it("accepts http MCP mode with OpenAI-compatible config", () => {
    const env = parseEnv({
      OPENAI_BASE_URL: "https://dashscope.aliyuncs.com/compatible-mode/v1",
      OPENAI_MODEL: "glm-5.1",
      OPENAI_API_KEY: "sk-test",
      XHS_MCP_MODE: "http",
      XHS_MCP_HTTP_URL: "http://localhost:8000"
    });

    expect(env.xhsMcp.mode).toBe("http");
    expect(env.openai.model).toBe("glm-5.1");
  });

  it("rejects http MCP mode without URL", () => {
    expect(() =>
      parseEnv({
        OPENAI_BASE_URL: "https://dashscope.aliyuncs.com/compatible-mode/v1",
        OPENAI_MODEL: "glm-5.1",
        OPENAI_API_KEY: "sk-test",
        XHS_MCP_MODE: "http"
      })
    ).toThrow(/XHS_MCP_HTTP_URL/);
  });
});

describe("parseKeywordConfig", () => {
  it("normalizes empty topN to defaultTopN", () => {
    const config = parseKeywordConfig({
      defaultTopN: 5,
      keywords: [{ keyword: "AI工具", enabled: true }]
    });

    expect(config.keywords[0]).toEqual({
      keyword: "AI工具",
      topN: 5,
      enabled: true
    });
  });
});
```

- [ ] **Step 2: Run tests to verify failure**

Run: `npm test -- tests/server/config.test.ts`

Expected: FAIL because `src/server/config/env.ts` and `src/server/config/keywords.ts` do not exist.

- [ ] **Step 3: Create shared domain types**

Write `src/server/types.ts`:

```ts
export type XhsMcpMode = "http" | "stdio";

export type AppEnv = {
  openai: {
    baseUrl: string;
    model: string;
    apiKey: string;
  };
  xhsMcp:
    | { mode: "http"; httpUrl: string }
    | { mode: "stdio"; command: string };
};

export type KeywordConfig = {
  defaultTopN: number;
  keywords: KeywordItem[];
};

export type KeywordItem = {
  keyword: string;
  topN: number;
  enabled: boolean;
};

export type SourceNote = {
  noteId: string;
  xsecToken?: string;
  sourceUrl?: string;
  sourceKeyword: string;
  title: string;
  description: string;
  authorName?: string;
  authorId?: string;
  likeCount: number;
  favoriteCount: number;
  commentCount: number;
  shareCount: number;
  publishedAt?: string;
  imageUrls: string[];
  rawPayload: unknown;
};

export type AiCategory = "ai_news" | "ai_tool" | "both" | "irrelevant";

export type AiAnalysis = {
  isRelevant: boolean;
  category: AiCategory;
  confidence: number;
  reason: string;
  keyFacts: string[];
  skipReason?: string;
  riskFlags: string[];
};

export type CardPage = {
  title: string;
  subtitle?: string;
  bullets: string[];
  emphasis?: string;
};

export type Draft = {
  id: string;
  sourceNoteId: string;
  status: "drafted" | "rendered" | "approved" | "rejected" | "needs_review" | "failed";
  title: string;
  body: string;
  tags: string[];
  coverTitle: string;
  coverSubtitle?: string;
  cardPages: CardPage[];
};
```

- [ ] **Step 4: Implement env parser**

Write `src/server/config/env.ts`:

```ts
import { z } from "zod";
import type { AppEnv } from "../types";

const rawEnvSchema = z.object({
  OPENAI_BASE_URL: z.string().url(),
  OPENAI_MODEL: z.string().min(1),
  OPENAI_API_KEY: z.string().min(1),
  XHS_MCP_MODE: z.enum(["http", "stdio"]).default("http"),
  XHS_MCP_HTTP_URL: z.string().url().optional(),
  XHS_MCP_COMMAND: z.string().min(1).optional()
});

export function parseEnv(input: Record<string, string | undefined>): AppEnv {
  const raw = rawEnvSchema.parse(input);

  if (raw.XHS_MCP_MODE === "http") {
    if (!raw.XHS_MCP_HTTP_URL) {
      throw new Error("XHS_MCP_HTTP_URL is required when XHS_MCP_MODE=http");
    }

    return {
      openai: {
        baseUrl: raw.OPENAI_BASE_URL,
        model: raw.OPENAI_MODEL,
        apiKey: raw.OPENAI_API_KEY
      },
      xhsMcp: { mode: "http", httpUrl: raw.XHS_MCP_HTTP_URL }
    };
  }

  if (!raw.XHS_MCP_COMMAND) {
    throw new Error("XHS_MCP_COMMAND is required when XHS_MCP_MODE=stdio");
  }

  return {
    openai: {
      baseUrl: raw.OPENAI_BASE_URL,
      model: raw.OPENAI_MODEL,
      apiKey: raw.OPENAI_API_KEY
    },
    xhsMcp: { mode: "stdio", command: raw.XHS_MCP_COMMAND }
  };
}

export function getEnv(): AppEnv {
  return parseEnv(process.env);
}
```

- [ ] **Step 5: Implement keyword parser**

Write `src/server/config/keywords.ts`:

```ts
import { readFileSync } from "node:fs";
import { z } from "zod";
import type { KeywordConfig } from "../types";

const keywordInputSchema = z.object({
  defaultTopN: z.number().int().positive().default(5),
  keywords: z.array(
    z.object({
      keyword: z.string().trim().min(1),
      topN: z.number().int().positive().optional(),
      enabled: z.boolean().default(true)
    })
  )
});

export function parseKeywordConfig(input: unknown): KeywordConfig {
  const parsed = keywordInputSchema.parse(input);
  return {
    defaultTopN: parsed.defaultTopN,
    keywords: parsed.keywords.map((item) => ({
      keyword: item.keyword,
      topN: item.topN ?? parsed.defaultTopN,
      enabled: item.enabled
    }))
  };
}

export function loadKeywordConfig(path = "config/keywords.json"): KeywordConfig {
  return parseKeywordConfig(JSON.parse(readFileSync(path, "utf8")));
}
```

- [ ] **Step 6: Verify tests pass**

Run: `npm test -- tests/server/config.test.ts`

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/server/types.ts src/server/config/env.ts src/server/config/keywords.ts tests/server/config.test.ts
git commit -m "feat: add local configuration parsing"
```

## Task 3: SQLite Storage Foundation

**Files:**
- Create: `src/server/db/schema.ts`
- Create: `src/server/db/client.ts`
- Modify: `src/server/types.ts`

- [ ] **Step 1: Create schema module**

Write `src/server/db/schema.ts`:

```ts
export const schemaSql = `
CREATE TABLE IF NOT EXISTS keywords (
  id TEXT PRIMARY KEY,
  keyword TEXT NOT NULL UNIQUE,
  topN INTEGER NOT NULL,
  enabled INTEGER NOT NULL,
  createdAt TEXT NOT NULL,
  updatedAt TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS collection_runs (
  id TEXT PRIMARY KEY,
  status TEXT NOT NULL,
  startedAt TEXT NOT NULL,
  finishedAt TEXT,
  error TEXT,
  settingsSnapshot TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS source_notes (
  id TEXT PRIMARY KEY,
  noteId TEXT NOT NULL UNIQUE,
  xsecToken TEXT,
  sourceUrl TEXT,
  sourceKeyword TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  authorName TEXT,
  authorId TEXT,
  likeCount INTEGER NOT NULL,
  favoriteCount INTEGER NOT NULL,
  commentCount INTEGER NOT NULL,
  shareCount INTEGER NOT NULL,
  publishedAt TEXT,
  imageUrls TEXT NOT NULL,
  rawPayload TEXT NOT NULL,
  collectedAt TEXT NOT NULL,
  detailStatus TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS ai_analyses (
  id TEXT PRIMARY KEY,
  sourceNoteId TEXT NOT NULL,
  isRelevant INTEGER NOT NULL,
  category TEXT NOT NULL,
  confidence REAL NOT NULL,
  reason TEXT NOT NULL,
  keyFacts TEXT NOT NULL,
  skipReason TEXT,
  riskFlags TEXT NOT NULL,
  rawModelOutput TEXT NOT NULL,
  createdAt TEXT NOT NULL,
  FOREIGN KEY(sourceNoteId) REFERENCES source_notes(id)
);

CREATE TABLE IF NOT EXISTS drafts (
  id TEXT PRIMARY KEY,
  sourceNoteId TEXT NOT NULL,
  analysisId TEXT NOT NULL,
  status TEXT NOT NULL,
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  tags TEXT NOT NULL,
  coverTitle TEXT NOT NULL,
  coverSubtitle TEXT,
  cardPages TEXT NOT NULL,
  reviewNotes TEXT,
  createdAt TEXT NOT NULL,
  updatedAt TEXT NOT NULL,
  FOREIGN KEY(sourceNoteId) REFERENCES source_notes(id),
  FOREIGN KEY(analysisId) REFERENCES ai_analyses(id)
);

CREATE TABLE IF NOT EXISTS draft_assets (
  id TEXT PRIMARY KEY,
  draftId TEXT NOT NULL,
  pageIndex INTEGER NOT NULL,
  filePath TEXT NOT NULL,
  width INTEGER NOT NULL,
  height INTEGER NOT NULL,
  templateName TEXT NOT NULL,
  createdAt TEXT NOT NULL,
  FOREIGN KEY(draftId) REFERENCES drafts(id)
);
`;
```

- [ ] **Step 2: Create database client**

Write `src/server/db/client.ts`:

```ts
import Database from "better-sqlite3";
import { mkdirSync } from "node:fs";
import { dirname } from "node:path";
import { schemaSql } from "./schema";

export type Db = Database.Database;

let singleton: Db | undefined;

export function createDb(path = "data/app.db"): Db {
  mkdirSync(dirname(path), { recursive: true });
  const db = new Database(path);
  db.pragma("journal_mode = WAL");
  db.exec(schemaSql);
  return db;
}

export function getDb(): Db {
  singleton ??= createDb();
  return singleton;
}
```

- [ ] **Step 3: Verify TypeScript**

Run: `npx tsc --noEmit`

Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add src/server/db/schema.ts src/server/db/client.ts
git commit -m "feat: add sqlite storage foundation"
```

## Task 4: Ranking And Normalization

**Files:**
- Create: `src/server/collection/ranking.ts`
- Create: `src/server/collection/normalize.ts`
- Test: `tests/server/ranking.test.ts`
- Test: `tests/server/normalize.test.ts`

- [ ] **Step 1: Write failing ranking tests**

Write `tests/server/ranking.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { parseEngagementCount, rankTopNotes } from "../../src/server/collection/ranking";
import type { SourceNote } from "../../src/server/types";

const baseNote = (noteId: string, likeCount: number): SourceNote => ({
  noteId,
  sourceKeyword: "AI工具",
  title: noteId,
  description: "",
  likeCount,
  favoriteCount: 0,
  commentCount: 0,
  shareCount: 0,
  imageUrls: [],
  rawPayload: {}
});

describe("parseEngagementCount", () => {
  it("parses Chinese ten-thousand units", () => {
    expect(parseEngagementCount("1.2万")).toBe(12000);
    expect(parseEngagementCount("987")).toBe(987);
  });
});

describe("rankTopNotes", () => {
  it("orders notes by weighted engagement and returns top N", () => {
    const result = rankTopNotes(
      [
        { ...baseNote("a", 10), favoriteCount: 100 },
        { ...baseNote("b", 100), favoriteCount: 0 },
        { ...baseNote("c", 80), commentCount: 50 }
      ],
      2
    );

    expect(result.map((note) => note.noteId)).toEqual(["c", "a"]);
  });
});
```

- [ ] **Step 2: Write failing normalization tests**

Write `tests/server/normalize.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { normalizeSearchResult } from "../../src/server/collection/normalize";

describe("normalizeSearchResult", () => {
  it("maps flexible MCP payload fields into SourceNote", () => {
    const note = normalizeSearchResult(
      {
        id: "note-1",
        xsec_token: "token-1",
        title: "AI工具爆火",
        desc: "一个新的AI工具",
        user: { nickname: "作者", user_id: "user-1" },
        interact_info: { liked_count: "1.2万", collected_count: "300", comment_count: "20" },
        image_list: [{ url: "https://img.example/a.png" }]
      },
      "AI工具"
    );

    expect(note.noteId).toBe("note-1");
    expect(note.likeCount).toBe(12000);
    expect(note.imageUrls).toEqual(["https://img.example/a.png"]);
  });
});
```

- [ ] **Step 3: Run tests to verify failure**

Run: `npm test -- tests/server/ranking.test.ts tests/server/normalize.test.ts`

Expected: FAIL because modules do not exist.

- [ ] **Step 4: Implement ranking**

Write `src/server/collection/ranking.ts`:

```ts
import type { SourceNote } from "../types";

export function parseEngagementCount(value: unknown): number {
  if (typeof value === "number") return Math.max(0, Math.round(value));
  if (typeof value !== "string") return 0;

  const normalized = value.trim().replace(/,/g, "");
  if (!normalized) return 0;

  if (normalized.endsWith("万")) {
    const number = Number.parseFloat(normalized.slice(0, -1));
    return Number.isFinite(number) ? Math.round(number * 10000) : 0;
  }

  const number = Number.parseInt(normalized, 10);
  return Number.isFinite(number) ? number : 0;
}

export function engagementScore(note: SourceNote): number {
  return note.likeCount + note.favoriteCount * 1.2 + note.commentCount * 2 + note.shareCount * 1.5;
}

export function rankTopNotes(notes: SourceNote[], topN: number): SourceNote[] {
  return [...notes]
    .sort((a, b) => engagementScore(b) - engagementScore(a))
    .slice(0, topN);
}
```

- [ ] **Step 5: Implement normalization**

Write `src/server/collection/normalize.ts`:

```ts
import type { SourceNote } from "../types";
import { parseEngagementCount } from "./ranking";

function readString(value: unknown): string | undefined {
  return typeof value === "string" && value.trim() ? value.trim() : undefined;
}

function readRecord(value: unknown): Record<string, unknown> {
  return value && typeof value === "object" && !Array.isArray(value) ? (value as Record<string, unknown>) : {};
}

function readImages(value: unknown): string[] {
  if (!Array.isArray(value)) return [];
  return value
    .map((item) => {
      if (typeof item === "string") return item;
      const record = readRecord(item);
      return readString(record.url) ?? readString(record.url_default) ?? readString(record.src);
    })
    .filter((url): url is string => Boolean(url));
}

export function normalizeSearchResult(payload: unknown, sourceKeyword: string): SourceNote {
  const root = readRecord(payload);
  const user = readRecord(root.user ?? root.author);
  const interact = readRecord(root.interact_info ?? root.interactInfo ?? root.stats);

  const noteId = readString(root.note_id) ?? readString(root.noteId) ?? readString(root.id);
  if (!noteId) {
    throw new Error("MCP search result is missing note id");
  }

  return {
    noteId,
    xsecToken: readString(root.xsec_token) ?? readString(root.xsecToken),
    sourceUrl: readString(root.url) ?? readString(root.note_url),
    sourceKeyword,
    title: readString(root.title) ?? "",
    description: readString(root.desc) ?? readString(root.description) ?? "",
    authorName: readString(user.nickname) ?? readString(user.name),
    authorId: readString(user.user_id) ?? readString(user.userId) ?? readString(user.id),
    likeCount: parseEngagementCount(interact.liked_count ?? interact.likeCount ?? root.like_count),
    favoriteCount: parseEngagementCount(interact.collected_count ?? interact.favoriteCount ?? root.favorite_count),
    commentCount: parseEngagementCount(interact.comment_count ?? interact.commentCount ?? root.comment_count),
    shareCount: parseEngagementCount(interact.share_count ?? interact.shareCount ?? root.share_count),
    publishedAt: readString(root.time) ?? readString(root.publishedAt),
    imageUrls: readImages(root.image_list ?? root.images ?? root.imageUrls),
    rawPayload: payload
  };
}
```

- [ ] **Step 6: Verify tests pass**

Run: `npm test -- tests/server/ranking.test.ts tests/server/normalize.test.ts`

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/server/collection/ranking.ts src/server/collection/normalize.ts tests/server/ranking.test.ts tests/server/normalize.test.ts
git commit -m "feat: normalize and rank xiaohongshu notes"
```

## Task 5: MCP Client And Collection Service

**Files:**
- Create: `src/server/mcp/xhs-client.ts`
- Create: `src/server/collection/service.ts`

- [ ] **Step 1: Implement MCP client interface**

Write `src/server/mcp/xhs-client.ts`:

```ts
import type { AppEnv } from "../types";

export type LoginStatus = {
  loggedIn: boolean;
  message: string;
};

export type SearchOptions = {
  limit: number;
};

export interface XhsMcpClient {
  checkLogin(): Promise<LoginStatus>;
  searchNotes(keyword: string, options: SearchOptions): Promise<unknown[]>;
  getNoteDetail(noteId: string, xsecToken: string): Promise<unknown>;
}

export class HttpXhsMcpClient implements XhsMcpClient {
  constructor(private readonly baseUrl: string) {}

  async checkLogin(): Promise<LoginStatus> {
    const response = await fetch(`${this.baseUrl}/api/check_login`);
    if (!response.ok) return { loggedIn: false, message: `MCP HTTP ${response.status}` };
    const body = await response.json();
    return {
      loggedIn: Boolean(body.loggedIn ?? body.is_logged_in ?? body.success),
      message: String(body.message ?? "")
    };
  }

  async searchNotes(keyword: string, options: SearchOptions): Promise<unknown[]> {
    const response = await fetch(`${this.baseUrl}/api/search`, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ keyword, limit: options.limit })
    });
    if (!response.ok) throw new Error(`MCP search failed with HTTP ${response.status}`);
    const body = await response.json();
    return Array.isArray(body) ? body : Array.isArray(body.data) ? body.data : Array.isArray(body.items) ? body.items : [];
  }

  async getNoteDetail(noteId: string, xsecToken: string): Promise<unknown> {
    const response = await fetch(`${this.baseUrl}/api/note_detail`, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ note_id: noteId, xsec_token: xsecToken })
    });
    if (!response.ok) throw new Error(`MCP detail failed with HTTP ${response.status}`);
    return response.json();
  }
}

export function createXhsMcpClient(env: AppEnv): XhsMcpClient {
  if (env.xhsMcp.mode === "http") return new HttpXhsMcpClient(env.xhsMcp.httpUrl);
  throw new Error("stdio MCP mode is not wired yet; keep the XhsMcpClient interface unchanged when adding it");
}
```

- [ ] **Step 2: Implement collection service**

Write `src/server/collection/service.ts`:

```ts
import { randomUUID } from "node:crypto";
import type { Db } from "../db/client";
import type { KeywordItem, SourceNote } from "../types";
import type { XhsMcpClient } from "../mcp/xhs-client";
import { normalizeSearchResult } from "./normalize";
import { rankTopNotes } from "./ranking";

export async function collectKeyword(params: {
  db: Db;
  client: XhsMcpClient;
  keyword: KeywordItem;
}): Promise<SourceNote[]> {
  const rawResults = await params.client.searchNotes(params.keyword.keyword, {
    limit: Math.max(params.keyword.topN * 3, params.keyword.topN)
  });

  const normalized = rawResults.map((item) => normalizeSearchResult(item, params.keyword.keyword));
  const topNotes = rankTopNotes(normalized, params.keyword.topN);

  const insert = params.db.prepare(`
    INSERT INTO source_notes (
      id, noteId, xsecToken, sourceUrl, sourceKeyword, title, description,
      authorName, authorId, likeCount, favoriteCount, commentCount, shareCount,
      publishedAt, imageUrls, rawPayload, collectedAt, detailStatus
    ) VALUES (
      @id, @noteId, @xsecToken, @sourceUrl, @sourceKeyword, @title, @description,
      @authorName, @authorId, @likeCount, @favoriteCount, @commentCount, @shareCount,
      @publishedAt, @imageUrls, @rawPayload, @collectedAt, @detailStatus
    )
    ON CONFLICT(noteId) DO UPDATE SET
      sourceKeyword = source_notes.sourceKeyword || ',' || excluded.sourceKeyword,
      likeCount = excluded.likeCount,
      favoriteCount = excluded.favoriteCount,
      commentCount = excluded.commentCount,
      shareCount = excluded.shareCount,
      rawPayload = excluded.rawPayload
  `);

  const now = new Date().toISOString();
  for (const note of topNotes) {
    insert.run({
      ...note,
      id: randomUUID(),
      imageUrls: JSON.stringify(note.imageUrls),
      rawPayload: JSON.stringify(note.rawPayload),
      collectedAt: now,
      detailStatus: note.xsecToken ? "collected" : "failed_detail"
    });
  }

  return topNotes;
}
```

- [ ] **Step 3: Verify TypeScript**

Run: `npx tsc --noEmit`

Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add src/server/mcp/xhs-client.ts src/server/collection/service.ts
git commit -m "feat: add xiaohongshu mcp collection service"
```

## Task 6: AI JSON And Pipeline

**Files:**
- Create: `src/server/ai/json.ts`
- Create: `src/server/ai/client.ts`
- Create: `src/server/ai/pipeline.ts`
- Test: `tests/server/ai-json.test.ts`

- [ ] **Step 1: Write failing JSON parser tests**

Write `tests/server/ai-json.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { parseAiJson } from "../../src/server/ai/json";

describe("parseAiJson", () => {
  it("extracts JSON from fenced model output", () => {
    const value = parseAiJson<{ ok: boolean }>("```json\n{\"ok\":true}\n```");
    expect(value.ok).toBe(true);
  });

  it("throws a readable error for invalid JSON", () => {
    expect(() => parseAiJson("not json")).toThrow(/AI response was not valid JSON/);
  });
});
```

- [ ] **Step 2: Run test to verify failure**

Run: `npm test -- tests/server/ai-json.test.ts`

Expected: FAIL because `src/server/ai/json.ts` does not exist.

- [ ] **Step 3: Implement JSON parser**

Write `src/server/ai/json.ts`:

```ts
export function parseAiJson<T>(content: string): T {
  const trimmed = content.trim();
  const fenced = trimmed.match(/^```(?:json)?\s*([\s\S]*?)\s*```$/i);
  const candidate = fenced ? fenced[1] : trimmed;

  try {
    return JSON.parse(candidate) as T;
  } catch {
    const firstBrace = candidate.indexOf("{");
    const lastBrace = candidate.lastIndexOf("}");
    if (firstBrace >= 0 && lastBrace > firstBrace) {
      try {
        return JSON.parse(candidate.slice(firstBrace, lastBrace + 1)) as T;
      } catch {
        throw new Error("AI response was not valid JSON");
      }
    }
    throw new Error("AI response was not valid JSON");
  }
}
```

- [ ] **Step 4: Implement OpenAI-compatible client**

Write `src/server/ai/client.ts`:

```ts
import OpenAI from "openai";
import type { AppEnv } from "../types";

export type ChatMessage = {
  role: "system" | "user";
  content: string;
};

export class AiClient {
  private readonly client: OpenAI;

  constructor(private readonly env: AppEnv["openai"]) {
    this.client = new OpenAI({
      apiKey: env.apiKey,
      baseURL: env.baseUrl
    });
  }

  async completeJson(messages: ChatMessage[]): Promise<string> {
    const response = await this.client.chat.completions.create({
      model: this.env.model,
      messages,
      temperature: 0.4,
      response_format: { type: "json_object" }
    });

    const content = response.choices[0]?.message?.content;
    if (!content) throw new Error("AI response was empty");
    return content;
  }
}
```

- [ ] **Step 5: Implement AI pipeline prompts**

Write `src/server/ai/pipeline.ts`:

```ts
import type { AiAnalysis, CardPage, SourceNote } from "../types";
import type { AiClient } from "./client";
import { parseAiJson } from "./json";

export type RewriteResult = {
  title: string;
  body: string;
  tags: string[];
  coverTitle: string;
  coverSubtitle?: string;
  reviewNotes: string;
};

export async function classifyNote(ai: AiClient, note: SourceNote): Promise<AiAnalysis> {
  const content = await ai.completeJson([
    {
      role: "system",
      content: "你是小红书AI内容审核助手。只输出JSON，不要输出Markdown。"
    },
    {
      role: "user",
      content: JSON.stringify({
        task: "判断原帖是否属于AI资讯或AI工具内容",
        outputShape: {
          isRelevant: "boolean",
          category: "ai_news | ai_tool | both | irrelevant",
          confidence: "0-1 number",
          reason: "string",
          keyFacts: "string[]",
          skipReason: "string optional",
          riskFlags: "string[]"
        },
        note
      })
    }
  ]);

  return parseAiJson<AiAnalysis>(content);
}

export async function rewriteNote(ai: AiClient, note: SourceNote, analysis: AiAnalysis): Promise<RewriteResult> {
  const content = await ai.completeJson([
    {
      role: "system",
      content: "你是小红书AI科技内容编辑。改写必须原创表达，不得照搬原文，不得编造来源中没有的信息。只输出JSON。"
    },
    {
      role: "user",
      content: JSON.stringify({
        task: "基于原帖和事实要点生成小红书草稿",
        constraints: ["标题尽量不超过20个中文字符", "正文不超过1000中文字符", "生成5到8个标签", "避免夸大承诺和引流话术"],
        note,
        analysis
      })
    }
  ]);

  return parseAiJson<RewriteResult>(content);
}

export async function generateCardPages(ai: AiClient, rewrite: RewriteResult): Promise<CardPage[]> {
  const content = await ai.completeJson([
    {
      role: "system",
      content: "你是小红书图文卡片策划。把文案拆成6到8页信息图卡片。只输出JSON。"
    },
    {
      role: "user",
      content: JSON.stringify({
        task: "生成卡片分页脚本",
        outputShape: { pages: [{ title: "string", subtitle: "string optional", bullets: ["string"], emphasis: "string optional" }] },
        rewrite
      })
    }
  ]);

  return parseAiJson<{ pages: CardPage[] }>(content).pages;
}
```

- [ ] **Step 6: Verify tests and types**

Run: `npm test -- tests/server/ai-json.test.ts && npx tsc --noEmit`

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/server/ai/json.ts src/server/ai/client.ts src/server/ai/pipeline.ts tests/server/ai-json.test.ts
git commit -m "feat: add ai drafting pipeline"
```

## Task 7: Card Rendering

**Files:**
- Create: `src/server/cards/templates.ts`
- Create: `src/server/cards/render.ts`
- Test: `tests/server/cards.test.ts`

- [ ] **Step 1: Write failing card renderer test**

Write `tests/server/cards.test.ts`:

```ts
import { existsSync, rmSync } from "node:fs";
import { describe, expect, it } from "vitest";
import { renderDraftCards } from "../../src/server/cards/render";

describe("renderDraftCards", () => {
  it("renders card pages into png files", async () => {
    const outDir = "data/test-assets/card-render";
    rmSync(outDir, { recursive: true, force: true });

    const assets = await renderDraftCards({
      draftId: "draft-test",
      templateName: "clean-tech",
      outDir,
      pages: [{ title: "AI工具更新", subtitle: "3个重点", bullets: ["更快", "更便宜"] }]
    });

    expect(assets).toHaveLength(1);
    expect(existsSync(assets[0].filePath)).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify failure**

Run: `npm test -- tests/server/cards.test.ts`

Expected: FAIL because card renderer modules do not exist.

- [ ] **Step 3: Implement templates**

Write `src/server/cards/templates.ts`:

```ts
export type CardTemplateName = "clean-tech" | "bold-rednote";

export type CardTemplate = {
  name: CardTemplateName;
  background: string;
  foreground: string;
  accent: string;
};

export const templates: Record<CardTemplateName, CardTemplate> = {
  "clean-tech": {
    name: "clean-tech",
    background: "#F7FAFC",
    foreground: "#111827",
    accent: "#2563EB"
  },
  "bold-rednote": {
    name: "bold-rednote",
    background: "#FFF1F2",
    foreground: "#111827",
    accent: "#E11D48"
  }
};

export function getTemplate(name: string): CardTemplate {
  return templates[name as CardTemplateName] ?? templates["clean-tech"];
}
```

- [ ] **Step 4: Implement renderer**

Write `src/server/cards/render.ts`:

```ts
import { mkdirSync } from "node:fs";
import { join } from "node:path";
import sharp from "sharp";
import type { CardPage } from "../types";
import { getTemplate } from "./templates";

export type RenderedAsset = {
  pageIndex: number;
  filePath: string;
  width: number;
  height: number;
  templateName: string;
};

function escapeXml(value: string): string {
  return value.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;");
}

function pageToSvg(page: CardPage, templateName: string): string {
  const template = getTemplate(templateName);
  const bullets = page.bullets
    .slice(0, 5)
    .map((bullet, index) => `<text x="120" y="${760 + index * 92}" font-size="44" fill="${template.foreground}">• ${escapeXml(bullet)}</text>`)
    .join("");

  return `<svg width="1242" height="1660" viewBox="0 0 1242 1660" xmlns="http://www.w3.org/2000/svg">
    <rect width="1242" height="1660" rx="0" fill="${template.background}"/>
    <rect x="72" y="72" width="1098" height="1516" rx="56" fill="#FFFFFF"/>
    <rect x="112" y="128" width="160" height="16" rx="8" fill="${template.accent}"/>
    <text x="112" y="360" font-size="82" font-weight="700" fill="${template.foreground}">${escapeXml(page.title)}</text>
    <text x="112" y="480" font-size="46" fill="${template.accent}">${escapeXml(page.subtitle ?? page.emphasis ?? "")}</text>
    ${bullets}
  </svg>`;
}

export async function renderDraftCards(input: {
  draftId: string;
  templateName: string;
  outDir: string;
  pages: CardPage[];
}): Promise<RenderedAsset[]> {
  mkdirSync(input.outDir, { recursive: true });

  const assets: RenderedAsset[] = [];
  for (const [index, page] of input.pages.entries()) {
    const filePath = join(input.outDir, `page-${String(index + 1).padStart(2, "0")}.png`);
    await sharp(Buffer.from(pageToSvg(page, input.templateName))).png().toFile(filePath);
    assets.push({ pageIndex: index + 1, filePath, width: 1242, height: 1660, templateName: input.templateName });
  }
  return assets;
}
```

- [ ] **Step 5: Verify tests pass**

Run: `npm test -- tests/server/cards.test.ts`

Expected: PASS and a PNG file exists under `data/test-assets/card-render`.

- [ ] **Step 6: Commit**

```bash
git add src/server/cards/templates.ts src/server/cards/render.ts tests/server/cards.test.ts
git commit -m "feat: render draft card images"
```

## Task 8: Draft Export

**Files:**
- Create: `src/server/drafts/export.ts`
- Test: `tests/server/export.test.ts`

- [ ] **Step 1: Write failing export test**

Write `tests/server/export.test.ts`:

```ts
import { existsSync, readFileSync, rmSync } from "node:fs";
import { describe, expect, it } from "vitest";
import { exportDraftPackage } from "../../src/server/drafts/export";

describe("exportDraftPackage", () => {
  it("writes a publish package with markdown and metadata", () => {
    const outDir = "data/test-export";
    rmSync(outDir, { recursive: true, force: true });

    const result = exportDraftPackage({
      outDir,
      draft: {
        id: "draft-1",
        sourceNoteId: "note-1",
        status: "approved",
        title: "AI工具更新",
        body: "这是一篇测试正文",
        tags: ["AI工具", "效率"],
        coverTitle: "AI工具更新",
        cardPages: []
      },
      assets: [{ pageIndex: 1, filePath: "data/assets/draft-1/page-01.png", width: 1242, height: 1660, templateName: "clean-tech" }]
    });

    expect(existsSync(result.markdownPath)).toBe(true);
    expect(readFileSync(result.markdownPath, "utf8")).toContain("# AI工具更新");
  });
});
```

- [ ] **Step 2: Run test to verify failure**

Run: `npm test -- tests/server/export.test.ts`

Expected: FAIL because export module does not exist.

- [ ] **Step 3: Implement exporter**

Write `src/server/drafts/export.ts`:

```ts
import { mkdirSync, writeFileSync } from "node:fs";
import { join } from "node:path";
import type { Draft } from "../types";
import type { RenderedAsset } from "../cards/render";

export function exportDraftPackage(input: {
  outDir: string;
  draft: Draft;
  assets: RenderedAsset[];
}): { packageDir: string; markdownPath: string; metadataPath: string } {
  const packageDir = join(input.outDir, input.draft.id);
  mkdirSync(packageDir, { recursive: true });

  const markdownPath = join(packageDir, "content.md");
  const metadataPath = join(packageDir, "metadata.json");

  writeFileSync(
    markdownPath,
    [`# ${input.draft.title}`, "", input.draft.body, "", input.draft.tags.map((tag) => `#${tag}`).join(" "), ""].join("\n"),
    "utf8"
  );

  writeFileSync(
    metadataPath,
    JSON.stringify(
      {
        draftId: input.draft.id,
        sourceNoteId: input.draft.sourceNoteId,
        title: input.draft.title,
        tags: input.draft.tags,
        assets: input.assets
      },
      null,
      2
    ),
    "utf8"
  );

  return { packageDir, markdownPath, metadataPath };
}
```

- [ ] **Step 4: Verify tests pass**

Run: `npm test -- tests/server/export.test.ts`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/server/drafts/export.ts tests/server/export.test.ts
git commit -m "feat: export approved draft packages"
```

## Task 9: API Routes

**Files:**
- Create: `src/app/api/health/route.ts`
- Create: `src/app/api/keywords/route.ts`
- Create: `src/app/api/runs/route.ts`
- Create: `src/app/api/notes/route.ts`
- Create: `src/app/api/drafts/route.ts`
- Create: `src/app/api/drafts/[id]/actions/route.ts`

- [ ] **Step 1: Implement health endpoint**

Write `src/app/api/health/route.ts`:

```ts
import { NextResponse } from "next/server";
import { getEnv } from "@/server/config/env";
import { createXhsMcpClient } from "@/server/mcp/xhs-client";

export async function GET() {
  const env = getEnv();
  const client = createXhsMcpClient(env);
  const login = await client.checkLogin().catch((error: unknown) => ({
    loggedIn: false,
    message: error instanceof Error ? error.message : "Unknown MCP error"
  }));

  return NextResponse.json({
    ok: true,
    model: env.openai.model,
    mcp: login
  });
}
```

- [ ] **Step 2: Implement keyword endpoint**

Write `src/app/api/keywords/route.ts`:

```ts
import { NextRequest, NextResponse } from "next/server";
import { loadKeywordConfig, parseKeywordConfig } from "@/server/config/keywords";

let memoryConfig = loadKeywordConfig();

export async function GET() {
  return NextResponse.json(memoryConfig);
}

export async function PUT(request: NextRequest) {
  memoryConfig = parseKeywordConfig(await request.json());
  return NextResponse.json(memoryConfig);
}
```

- [ ] **Step 3: Implement list endpoints**

Write `src/app/api/notes/route.ts`:

```ts
import { NextResponse } from "next/server";
import { getDb } from "@/server/db/client";

export async function GET() {
  const rows = getDb().prepare("SELECT * FROM source_notes ORDER BY collectedAt DESC LIMIT 200").all();
  return NextResponse.json({ notes: rows });
}
```

Write `src/app/api/drafts/route.ts`:

```ts
import { NextResponse } from "next/server";
import { getDb } from "@/server/db/client";

export async function GET() {
  const rows = getDb().prepare("SELECT * FROM drafts ORDER BY updatedAt DESC LIMIT 200").all();
  return NextResponse.json({ drafts: rows });
}
```

- [ ] **Step 4: Implement run endpoint**

Write `src/app/api/runs/route.ts`:

```ts
import { randomUUID } from "node:crypto";
import { NextResponse } from "next/server";
import { loadKeywordConfig } from "@/server/config/keywords";
import { getEnv } from "@/server/config/env";
import { getDb } from "@/server/db/client";
import { createXhsMcpClient } from "@/server/mcp/xhs-client";
import { collectKeyword } from "@/server/collection/service";

export async function GET() {
  const rows = getDb().prepare("SELECT * FROM collection_runs ORDER BY startedAt DESC LIMIT 50").all();
  return NextResponse.json({ runs: rows });
}

export async function POST() {
  const db = getDb();
  const config = loadKeywordConfig();
  const runId = randomUUID();
  const now = new Date().toISOString();

  db.prepare("INSERT INTO collection_runs (id, status, startedAt, settingsSnapshot) VALUES (?, ?, ?, ?)").run(
    runId,
    "searching",
    now,
    JSON.stringify(config)
  );

  const client = createXhsMcpClient(getEnv());
  const enabled = config.keywords.filter((keyword) => keyword.enabled);

  for (const keyword of enabled) {
    await collectKeyword({ db, client, keyword });
  }

  db.prepare("UPDATE collection_runs SET status = ?, finishedAt = ? WHERE id = ?").run("completed", new Date().toISOString(), runId);
  return NextResponse.json({ runId, status: "completed" });
}
```

- [ ] **Step 5: Implement draft action endpoint**

Write `src/app/api/drafts/[id]/actions/route.ts`:

```ts
import { NextRequest, NextResponse } from "next/server";
import { getDb } from "@/server/db/client";

export async function POST(request: NextRequest, context: { params: Promise<{ id: string }> }) {
  const { id } = await context.params;
  const body = await request.json();
  const action = String(body.action ?? "");

  if (!["approve", "reject"].includes(action)) {
    return NextResponse.json({ error: "Unsupported draft action" }, { status: 400 });
  }

  const status = action === "approve" ? "approved" : "rejected";
  getDb().prepare("UPDATE drafts SET status = ?, updatedAt = ? WHERE id = ?").run(status, new Date().toISOString(), id);
  return NextResponse.json({ id, status });
}
```

- [ ] **Step 6: Verify TypeScript**

Run: `npx tsc --noEmit`

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/app/api/health/route.ts src/app/api/keywords/route.ts src/app/api/runs/route.ts src/app/api/notes/route.ts src/app/api/drafts/route.ts src/app/api/drafts/[id]/actions/route.ts
git commit -m "feat: add local agent api routes"
```

## Task 10: Web Console Pages

**Files:**
- Create: `src/app/layout.tsx`
- Create: `src/app/page.tsx`
- Create: `src/app/settings/page.tsx`
- Create: `src/app/runs/page.tsx`
- Create: `src/app/notes/page.tsx`
- Create: `src/app/drafts/page.tsx`
- Create: `src/app/approved/page.tsx`

- [ ] **Step 1: Create app layout**

Write `src/app/layout.tsx`:

```tsx
import type { ReactNode } from "react";

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="zh-CN">
      <body style={{ margin: 0, fontFamily: "Arial, sans-serif", background: "#f8fafc", color: "#111827" }}>
        <nav style={{ display: "flex", gap: 16, padding: 20, background: "#111827" }}>
          {[
            ["/", "概览"],
            ["/settings", "配置"],
            ["/runs", "采集任务"],
            ["/notes", "候选帖子"],
            ["/drafts", "AI草稿"],
            ["/approved", "已通过"]
          ].map(([href, label]) => (
            <a key={href} href={href} style={{ color: "white", textDecoration: "none" }}>
              {label}
            </a>
          ))}
        </nav>
        <main style={{ padding: 24 }}>{children}</main>
      </body>
    </html>
  );
}
```

- [ ] **Step 2: Create dashboard page**

Write `src/app/page.tsx`:

```tsx
export default function HomePage() {
  return (
    <section>
      <h1>小红书 AI 选题与图文草稿 Agent</h1>
      <p>第一版支持关键词采集、AI筛选、改写、图文卡片生成和人工审核，不自动发布。</p>
      <div style={{ display: "grid", gridTemplateColumns: "repeat(4, minmax(0, 1fr))", gap: 16 }}>
        {["配置关键词", "启动采集", "审核草稿", "导出发布包"].map((title) => (
          <div key={title} style={{ padding: 20, borderRadius: 12, background: "white" }}>
            <h2>{title}</h2>
          </div>
        ))}
      </div>
    </section>
  );
}
```

- [ ] **Step 3: Create simple pages**

Write each page with server-side fetch from its API:

`src/app/settings/page.tsx`:

```tsx
export default async function SettingsPage() {
  const config = await fetch("http://localhost:3000/api/keywords", { cache: "no-store" }).then((res) => res.json()).catch(() => null);
  return (
    <section>
      <h1>配置</h1>
      <pre>{JSON.stringify(config, null, 2)}</pre>
    </section>
  );
}
```

`src/app/runs/page.tsx`:

```tsx
export default async function RunsPage() {
  const data = await fetch("http://localhost:3000/api/runs", { cache: "no-store" }).then((res) => res.json()).catch(() => ({ runs: [] }));
  return (
    <section>
      <h1>采集任务</h1>
      <pre>{JSON.stringify(data.runs, null, 2)}</pre>
    </section>
  );
}
```

`src/app/notes/page.tsx`:

```tsx
export default async function NotesPage() {
  const data = await fetch("http://localhost:3000/api/notes", { cache: "no-store" }).then((res) => res.json()).catch(() => ({ notes: [] }));
  return (
    <section>
      <h1>候选帖子</h1>
      <pre>{JSON.stringify(data.notes, null, 2)}</pre>
    </section>
  );
}
```

`src/app/drafts/page.tsx`:

```tsx
export default async function DraftsPage() {
  const data = await fetch("http://localhost:3000/api/drafts", { cache: "no-store" }).then((res) => res.json()).catch(() => ({ drafts: [] }));
  return (
    <section>
      <h1>AI 草稿</h1>
      <pre>{JSON.stringify(data.drafts, null, 2)}</pre>
    </section>
  );
}
```

`src/app/approved/page.tsx`:

```tsx
export default async function ApprovedPage() {
  const data = await fetch("http://localhost:3000/api/drafts", { cache: "no-store" }).then((res) => res.json()).catch(() => ({ drafts: [] }));
  const approved = data.drafts.filter((draft: { status: string }) => draft.status === "approved");
  return (
    <section>
      <h1>已通过草稿</h1>
      <pre>{JSON.stringify(approved, null, 2)}</pre>
    </section>
  );
}
```

- [ ] **Step 4: Verify app builds**

Run: `npm run build`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/app/layout.tsx src/app/page.tsx src/app/settings/page.tsx src/app/runs/page.tsx src/app/notes/page.tsx src/app/drafts/page.tsx src/app/approved/page.tsx
git commit -m "feat: add local review console pages"
```

## Task 11: End-To-End Verification

**Files:**
- Modify only files required by failures found in this task.

- [ ] **Step 1: Run unit tests**

Run: `npm test`

Expected: PASS for config, ranking, normalization, AI JSON, card rendering, and export tests.

- [ ] **Step 2: Run type check**

Run: `npx tsc --noEmit`

Expected: PASS.

- [ ] **Step 3: Run production build**

Run: `npm run build`

Expected: PASS.

- [ ] **Step 4: Create local env file**

Create `.env.local` with the user's local test values:

```text
OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
OPENAI_MODEL=glm-5.1
OPENAI_API_KEY=local-private-test-key
XHS_MCP_MODE=http
XHS_MCP_HTTP_URL=http://localhost:8000
```

Expected: `.env.local` is ignored by git.

- [ ] **Step 5: Manual smoke test**

Run: `npm run dev`

Open: `http://localhost:3000`

Check:

- Dashboard loads.
- `/api/health` returns model name and MCP status.
- `/settings` shows default keywords.
- `/runs`, `/notes`, `/drafts`, and `/approved` render without crashing.

- [ ] **Step 6: Commit final fixes**

```bash
git status --short
git add src package.json package-lock.json config .gitignore .env.example next.config.mjs tsconfig.json vitest.config.ts tests
git commit -m "fix: stabilize local agent verification"
```

Only run the final commit if verification required code fixes. Keep `.env.local`, `data/`, and generated images untracked.

## Self-Review

- Spec coverage: The plan covers project setup, keyword config, MCP health/search adapter, Top N ranking, SQLite storage, AI classification/rewrite/card scripts, deterministic card rendering, Web console pages, approved export packages, and verification. Automatic publishing remains out of scope and is only reserved through the adapter interface.
- Placeholder scan: The plan contains concrete file paths, code snippets, commands, and expected outcomes. The local API key is intentionally represented as a local-only secret value in `.env.local`.
- Type consistency: Shared names are consistent across tasks: `SourceNote`, `AiAnalysis`, `CardPage`, `Draft`, `XhsMcpClient`, `collectKeyword`, `parseAiJson`, and `renderDraftCards`.
