# P5.js Drawing Optimization Loop - Implementation Plan

## Overview

A full-stack TypeScript system that uses Claude with vision capabilities in an iterative feedback loop to progressively improve p5.js drawings. The agent generates drawing code, views the rendered result, critiques it, and produces improved versions.

---

## Architecture & Data Flow

### Components

1. **Client** (Vite + React + TypeScript)
   - Orchestrates the optimization loop
   - Runs p5.js sketches in sandboxed iframe
   - Captures canvas as WebP images
   - Displays history with auto-scroll
   - Provides scoring/tagging interface

2. **Server** (Node + Express + TypeScript)
   - Calls Claude API (vision + text generation)
   - Persists sessions & attempts as JSON files
   - Stores images directly to disk
   - Exposes simple REST API
   - Serves static image files

3. **Storage** (Local disk - no database)
   - `./data/sessions/{sessionId}/` directories
   - `session.json` - session metadata
   - `attempts.json` - array of all attempts
   - `images/{attemptId}.webp` - captured canvases
   - Simple, inspectable, git-friendly

### Core Loop Data Flow

```
1. Client starts session → POST /api/sessions
2. Initial code generation → POST /api/agent/generate {sessionId, objective}
3. Client runs code in SketchRunner (iframe + p5 instance mode)
4. Capture canvas → Blob(WebP, q=0.85) → POST /api/images
5. Persist attempt → POST /api/attempts {sessionId, code, imageUrl, critique}
6. Improvement → POST /api/agent/improve {sessionId, code, imageUrl, objective}
7. Repeat steps 3-6 until stopped
8. UI queries last 5 attempts → GET /api/sessions/:id/history?limit=5
```

---

## Technology Stack

### Client
- **Build**: Vite
- **Framework**: React + TypeScript
- **Graphics**: p5.js (instance mode)
- **Sandbox**: iframe with strict CSP
- **Styling**: Minimal inline CSS
- **HTTP**: fetch

### Server
- **Runtime**: Node 20+
- **Framework**: Express (minimal setup)
- **Dependencies**: 
  - express
  - cors
  - multer (file uploads)
  - @anthropic-ai/sdk
- **Storage**: JSON files + images straight to disk

### Storage Structure
```
./data/
  sessions/
    {sessionId}/
      session.json
      attempts.json
      images/
        {attemptId}.webp
```

### Authentication
- None (single-user local prototype)
- Can add API key via .env if needed

---

## API Design

### Endpoints

#### Session Management
```typescript
POST /api/sessions
Body: { objective: string, size?: { w: number, h: number }, settings?: { maxIters?: number } }
Returns: { session: { id, objective, size, createdAt, active: true } }

POST /api/sessions/:id/stop
Returns: { ok: true }

GET /api/sessions/:id/history?limit=5
Returns: { attempts: Attempt[] }
```

#### Image Upload
```typescript
POST /api/images (multipart)
Form: { file: Blob, sessionId: string }
Returns: { imageUrl: string, width: number, height: number, sizeBytes: number }
```

#### Attempts
```typescript
POST /api/attempts
Body: { 
  sessionId: string, 
  index?: number, 
  code: string, 
  imageUrl?: string, 
  critique?: string, 
  error?: string, 
  metadata?: Record<string, any> 
}
Returns: Attempt

PATCH /api/attempts/:id
Body: { score?: number, tags?: string[] }
Returns: Attempt

GET /api/attempts/query?sessionId=...&minScore=...&maxScore=...&tags=tag1,tag2
Returns: { attempts: Attempt[] }
```

#### Agent (Claude)
```typescript
POST /api/agent/generate
Body: { sessionId: string, objective: string, size?: { w, h }, topKHistory?: number }
Returns: { code: string, critique: string, model: string, tokens?: { input, output } }

POST /api/agent/improve
Body: { sessionId: string, code: string, imageUrl: string, objective: string, topKHistory?: number }
Returns: { code: string, critique: string, model: string, tokens?: { input, output } }
```

### Data Types

```typescript
interface Attempt {
  id: string;
  index: number;
  code: string;
  critique: string | null;
  imageUrl: string | null;
  score: number | null;
  tags: string[];
  error: string | null;
  createdAt: Date;
  metadata: {
    durationMs?: number;
    model?: string;
    tokens?: { input: number; output: number };
    runtime?: { errors?: string[] };
    seed?: number;
  };
}
```

---

## Storage Format (JSON Files)

### `./data/sessions/{sessionId}/session.json`
```json
{
  "id": "sess_abc123",
  "objective": "Generate abstract geometric composition",
  "width": 512,
  "height": 512,
  "active": true,
  "model": "claude-3-5-sonnet-20241022",
  "settings": {},
  "createdAt": "2025-10-23T10:30:00Z"
}
```

### `./data/sessions/{sessionId}/attempts.json`
```json
[
  {
    "id": "att_xyz789",
    "index": 0,
    "code": "return (p) => { ... }",
    "critique": "Good composition but needs more contrast",
    "imageUrl": "/data/sessions/sess_abc123/images/att_xyz789.webp",
    "score": 75,
    "tags": ["geometric", "warm-colors"],
    "error": null,
    "metadata": {
      "durationMs": 450,
      "model": "claude-3-5-sonnet-20241022",
      "tokens": { "input": 1200, "output": 850 }
    },
    "createdAt": "2025-10-23T10:31:00Z"
  }
]
```

**Benefits**:
- Human-readable
- Git-friendly (can commit interesting sessions)
- Easy to inspect/debug
- No migration scripts
- Zero database overhead

---

## Client Implementation

### Components

#### 1. App
- Manages session state and loop (running/stopped)
- Inputs: objective, canvas size
- Buttons: Start, Stop
- Resumes if active session exists (GET on load)

#### 2. HistoryList
- Displays last 5 attempts
- Shows: image thumbnail, code (collapsible), critique, metadata
- Score/tag editor for each attempt
- Auto-scrolls to new attempts using `ref.scrollIntoView()`

#### 3. SketchRunner
- Runs p5.js code in sandboxed iframe
- Signals when rendered or errored
- Returns captured image as Blob

### SketchRunner Details (Safe Dynamic Execution)

**Sandbox Setup**:
```html
<iframe 
  sandbox="allow-scripts" 
  csp="default-src 'none'; script-src 'self'; img-src 'self' blob: data:; connect-src 'none';"
></iframe>
```

**Inside iframe**:
1. Preload p5.min.js
2. Listen for postMessage with `{ code, width, height }`
3. Instantiate p5 in instance mode:
   ```javascript
   const sketch = new Function('p', code);
   const inst = new p5((p) => {
     try {
       sketch(p);
     } catch (e) {
       parent.postMessage({ type: 'error', error: String(e) }, '*');
     }
   }, container);
   ```

**Guardrails**:
- Force canvas size: `p.createCanvas(width, height); p.pixelDensity(1);`
- Enforce single-frame: `p.noLoop()` unless code explicitly calls `loop()`
- Hard timeout: 800ms to capture and dispose `inst.remove()`

**Capture Flow**:
1. After two `requestAnimationFrame` ticks following setup/draw
2. Get `canvas.toDataURL('image/webp', 0.85)`
3. postMessage to parent `{ type: 'image', dataUrl }`
4. Parent converts to Blob → uploads to server

**Error Handling**:
- Surface uncaught errors in UI
- Persist attempt with error
- Send error text to `/api/agent/improve` for Claude to fix

### Loop Orchestration (Client-side)

```javascript
while (running) {
  // 1. Get code from Claude
  const { code, critique } = !history.length 
    ? await api.agent.generate({ sessionId, objective })
    : await api.agent.improve({ sessionId, code: lastAttempt.code, imageUrl: lastAttempt.imageUrl, objective });
  
  // 2. Run code and capture
  const imageBlob = await sketchRunner.render(code, { w: 512, h: 512 });
  
  // 3. Upload image
  const { imageUrl } = await api.images.upload(imageBlob, sessionId);
  
  // 4. Persist attempt
  await api.attempts.create({ sessionId, code, imageUrl, critique, metadata });
  
  // 5. Backoff to respect rate limits
  await delay(1500);
}
```

**Stop/Resume**:
- Stop button sets `running=false` and updates `session.active=false`
- On page refresh: GET last active session → fetch last attempt → continue if user presses Start

---

## Server Implementation

### Setup
```typescript
import express from 'express';
import cors from 'cors';
import multer from 'multer';
import Anthropic from '@anthropic-ai/sdk';
import fs from 'fs/promises';
import path from 'path';

const app = express();
const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

app.use(cors());
app.use(express.json());
app.use('/data', express.static('./data')); // Serve images

const upload = multer({ dest: 'temp/' }); // Temp storage, then move to session dir
```

### Storage Helpers
```typescript
async function loadSession(sessionId: string) {
  const sessionPath = path.join('./data/sessions', sessionId, 'session.json');
  return JSON.parse(await fs.readFile(sessionPath, 'utf-8'));
}

async function saveSession(session: Session) {
  const sessionDir = path.join('./data/sessions', session.id);
  await fs.mkdir(sessionDir, { recursive: true });
  await fs.writeFile(
    path.join(sessionDir, 'session.json'),
    JSON.stringify(session, null, 2)
  );
}

async function loadAttempts(sessionId: string) {
  const attemptsPath = path.join('./data/sessions', sessionId, 'attempts.json');
  try {
    return JSON.parse(await fs.readFile(attemptsPath, 'utf-8'));
  } catch {
    return [];
  }
}

async function saveAttempt(sessionId: string, attempt: Attempt) {
  const attempts = await loadAttempts(sessionId);
  attempts.push(attempt);
  const attemptsPath = path.join('./data/sessions', sessionId, 'attempts.json');
  await fs.writeFile(attemptsPath, JSON.stringify(attempts, null, 2));
}
```

### Claude Integration

**Keep calls server-side**:
- Environment variable: `ANTHROPIC_API_KEY`
- Model: `claude-3-5-sonnet-20241022` or latest vision-capable
- Use JSON-only responses (parse with robust fallback)

### Image Handling
1. Accept multipart file or base64 dataURL
2. Save directly to `./data/sessions/{sessionId}/images/{attemptId}.webp`
3. Return public URL: `/data/sessions/{sessionId}/images/{attemptId}.webp`
4. No re-encoding needed (client already sends WebP from canvas)

---

## Prompt Engineering Strategy

### System Prompt (Both Generate & Improve)

```
You are a p5.js code generator improving drawings iteratively.

CONSTRAINTS:
- Use p5.js instance mode ONLY
- Output a function body that accepts parameter 'p' (no global mode)
- Canvas size: {W}x{H} - call p.createCanvas(W, H) in setup
- Call p.pixelDensity(1) and p.noLoop() for single-frame rendering
- NO external assets, network calls, or async operations
- Prefer deterministic code; if random, use p.randomSeed() and p.noiseSeed()
- Keep draw time under 500ms
- Output JSON ONLY: { "critique": string, "code": string }

STYLE:
- Concise, well-structured code with minimal comments
```

### Initial Generation Message

```json
{
  "objective": "Generate a balanced abstract composition with 3-5 colors and geometric shapes",
  "size": { "w": 512, "h": 512 },
  "constraints": "Use warm color palette, ensure symmetry",
  "history_summary": "Last 3 attempts: [...]"
}
```

### Improvement Message

```json
{
  "objective": "Generate a balanced abstract composition...",
  "previous_code": "...",
  "image_url": "https://...",
  "request": "Provide a brief visual critique and ONE concrete improvement. If there were runtime errors, fix them first.",
  "runtime_error": "..."
}
```

### Expected Output Format

```json
{
  "critique": "The shapes overlap awkwardly; palette lacks contrast.",
  "code": "return (p) => {\n  p.setup = function() {\n    p.createCanvas(512, 512);\n    p.pixelDensity(1);\n    p.background(20);\n    // drawing code...\n    p.noLoop();\n  };\n};"
}
```

---

## Safety Considerations

### 1. Sandbox Execution
- **iframe**: `sandbox="allow-scripts"` (no same-origin, no top navigation)
- **CSP**: `default-src 'none'; script-src 'self'; img-src 'self' blob: data:; connect-src 'none'`
- **Timeouts**: Destroy p5 instance after capture or 1s max
- **No network**: Disable all network access in iframe

### 2. Code Validation
- Pre-scan for disallowed APIs: `fetch`, `document.*`, `window.open`, `eval`
- Force insertion of `p.pixelDensity(1)`, `p.noLoop()` if missing
- Reject or re-prompt on violations

### 3. Rate Limiting & Cost Control
- Server rate-limit: 30 requests/min per IP/session
- Client backoff: 1-2s between iterations
- Max iterations per session setting

### 4. Storage Quotas
- Cap: 200 attempts per session
- Image retention: LRU or manual prune
- Disk cleanup policy

### 5. Prompt Injection
- Strong system prompt with constraints
- Ignore instructions in images
- Never execute code/URLs from generated content

### 6. Input Sanitization
- Normalize tags (lowercase kebab-case)
- Clamp scores: 0-100

---

## Potential Challenges & Solutions

### 1. Model Returns Global Mode or Missing `noLoop()`
**Solution**: Detect via regex; auto-wrap or re-prompt; insert `p.noLoop()` if absent

### 2. Infinite Draw or Heavy Computation
**Solution**: Timeout and destroy instance; send runtime error + perf note back to model

### 3. Blank or Late Renders
**Solution**: Capture after two RAFs; enforce `p.background()` in setup; re-prompt if blank

### 4. JSON Parsing Issues
**Solution**: Strip code fences, forgiving parse (find first/last braces), validate with zod

### 5. Cyclic or Non-Converging Changes
**Solution**: 
- Request ONE targeted improvement per iteration
- Surface last 3-5 attempts summary with "avoid prior mistakes"
- Add simple score heuristic to steer (contrast, spacing, etc.)

### 6. Cost & Latency
**Solution**: 
- Smaller image size (512x512)
- Compress to WebP
- Throttle iterations
- Optional downscale before upload

---

## Implementation Phases

### Phase 1: Core Loop (1-2 days)
- ✓ Simple Express server with JSON file storage
- ✓ Agent generate/improve endpoints
- ✓ Sandboxed iframe SketchRunner
- ✓ Image upload & persistence
- ✓ Basic loop orchestration

### Phase 2: UI + Features (1 day)
- ✓ HistoryList with auto-scroll
- ✓ Scoring/tagging interface
- ✓ Query by score/tags
- ✓ Resume sessions on page reload

### Phase 3: Polish (0.5-1 day)
- ✓ CSP & sandbox enforcement
- ✓ Timeouts & error handling
- ✓ Better prompts
- ✓ README with examples

---

## Advanced Extensions (Future)

### Multi-User Support
- Add authentication
- Per-user quotas
- SSE/WebSocket for live updates

### Scaling
- Migrate to PostgreSQL + S3
- Queue (BullMQ) for model calls
- Containerize runner

### Better Correctness
- Static analysis of generated code
- AST transforms to inject guards
- Run sketches in Web Worker + OffscreenCanvas

### Retrieval-Augmented Prompting
- Vector search over past attempts
- Condition improvements on similar past successes

### Automated Scoring
- Heuristic scoring (contrast, balance, palette diversity)
- Model-based aesthetic scoring
- Best-of-N selection

---

## Key Architectural Decisions

1. **Client-orchestrated loop**: Keeps rendering in browser, simpler to implement
2. **REST over WebSocket**: Fewer moving parts, client knows when iterations finish
3. **SQLite + local files**: Fast persistence, minimal ops, easy migration
4. **iframe sandbox**: Safest practical way to run untrusted code
5. **WebP @ 512x512**: Good quality/size tradeoff for visual feedback

---

## Risk Mitigation

| Risk | Guardrail |
|------|-----------|
| Untrusted code escape | Strict sandbox + CSP, no same-origin, destroy after use |
| API overload | Server rate limits + client backoff + max iterations |
| Data growth | Retention policy, image cleanup, dimension limits |
| Poor improvement quality | Tighter prompts, critique memory, small targeted changes |

---

## Getting Started

1. **Initialize server**: 
   ```bash
   cd server && npm init -y
   npm install express cors multer @anthropic-ai/sdk
   npm install -D typescript @types/node @types/express @types/cors @types/multer tsx
   ```

2. **Initialize client**: 
   ```bash
   npm create vite@latest client -- --template react-ts
   cd client && npm install p5
   ```

3. **Create folder structure**: `mkdir -p data/sessions`

4. **Create API endpoints**: Sessions, images, attempts, agent (just read/write JSON)

5. **Implement SketchRunner**: Sandboxed iframe component

6. **Implement loop**: Generate → Render → Capture → Improve

7. **Add UI**: HistoryList with auto-scroll, scoring/tagging

8. **Test**: Start with "draw a circle" and iterate

**Run**:
```bash
# Terminal 1 (server)
cd server && npx tsx src/server.ts

# Terminal 2 (client)  
cd client && npm run dev
```

---

## Estimated Timeline

**MVP (Core + UI + Persistence)**: 2-4 days
**Full Feature Set (Scoring/Tagging/Resume)**: +1 day
**Production Hardening**: +1-2 days

**Total**: ~4-7 days for complete system
