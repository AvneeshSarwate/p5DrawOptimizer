# P5.js Drawing Optimization Loop - Implementation Plan

## Overview

A full-stack TypeScript system that uses Claude with vision capabilities in an iterative feedback loop to progressively improve p5.js drawings. The agent generates drawing code, views the rendered result, critiques it, and produces improved versions.

---

## Architecture & Data Flow

### Components

1. **Client** (Vite + Vue 3 + TypeScript)
   - Orchestrates the optimization loop
   - **Calls Claude API directly** (vision + text generation)
   - Runs p5.js sketches in sandboxed iframe
   - Captures canvas as WebP images
   - Displays history with auto-scroll
   - Provides scoring/tagging interface

2. **Server** (Node + Express + TypeScript)
   - **Thin storage layer only**
   - Persists sessions & attempts as JSON files
   - Stores images directly to disk
   - Exposes simple REST API (sessions, attempts, images)
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
2. Client calls Claude API → generate initial code
3. Client runs code in SketchRunner (iframe + p5 instance mode)
4. Capture canvas → Blob(WebP, q=0.85) → POST /api/images
5. Persist attempt → POST /api/attempts {sessionId, code, imageUrl, critique}
6. Client calls Claude API → improve based on image + code
7. Repeat steps 3-6 until stopped
8. UI queries attempts → GET /api/sessions/:id/attempts
```

---

## Technology Stack

### Client
- **Build**: Vite (already set up)
- **Framework**: Vue 3 + TypeScript (Composition API)
- **Graphics**: p5.js (instance mode)
- **AI**: @anthropic-ai/sdk (calls Claude directly from browser)
- **Sandbox**: iframe with strict CSP
- **Styling**: Minimal inline CSS
- **HTTP**: fetch
- **Config**: API key in git-ignored config file

### Server
- **Runtime**: Node 20+
- **Framework**: Express (minimal setup)
- **Dependencies**: 
  - express
  - cors (for dev: Vite on :5173, Express on :3000)
- **Storage**: JSON files + images straight to disk
- **Role**: Thin storage layer (no AI logic)

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

PATCH /api/sessions/:id
Body: { active: boolean }
Returns: { session: Session }

GET /api/sessions/:id/attempts
Returns: { attempts: Attempt[] }
```

#### Image Upload
```typescript
POST /api/images?sessionId={sessionId}&attemptId={attemptId}
Body: Blob (raw binary WebP)
Content-Type: image/webp
Returns: { imageUrl: string }
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

### Components (Vue 3 Composition API)

#### 1. App.vue
- Manages session state and loop (running/stopped) with `ref()`
- Inputs: objective, canvas size
- Buttons: Start, Stop
- Resumes if active session exists (GET on `onMounted`)

#### 2. HistoryList.vue
- Displays last 5 attempts
- Shows: image thumbnail, code (collapsible), critique, metadata
- Score/tag editor for each attempt
- Auto-scrolls to new attempts using template ref + `scrollIntoView()`

#### 3. SketchRunner.vue
- Runs p5.js code in sandboxed iframe
- Emits `@rendered` or `@error` events
- Returns captured image as Blob via callback

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
2. Get canvas element and call `canvas.toBlob(callback, 'image/webp', 0.85)`
3. postMessage Blob to parent `{ type: 'image', blob }` (use Transferable if needed)
4. Parent uploads Blob directly to server with fetch

**Error Handling**:
- Surface uncaught errors in UI
- Persist attempt with error
- Send error text to `/api/agent/improve` for Claude to fix

### Loop Orchestration (Client-side)

```typescript
// App.vue (Composition API)
import { ref, onMounted } from 'vue';
import Anthropic from '@anthropic-ai/sdk';
import { config } from './config';

const anthropic = new Anthropic({ 
  apiKey: config.anthropicKey,
  dangerouslyAllowBrowser: true // Local use only
});

const running = ref(false);
const history = ref<Attempt[]>([]);

async function callClaude(prompt: string, imageUrl?: string) {
  const messages: any[] = [{
    role: 'user',
    content: imageUrl 
      ? [
          { type: 'image', source: { type: 'url', url: imageUrl } },
          { type: 'text', text: prompt }
        ]
      : prompt
  }];

  const response = await anthropic.messages.create({
    model: 'claude-3-5-sonnet-20241022',
    max_tokens: 2000,
    messages
  });

  return response.content[0].text; // Parse JSON from response
}

async function runLoop() {
  while (running.value) {
    // 1. Call Claude (initial or improvement)
    const lastAttempt = history.value[history.value.length - 1];
    const prompt = !lastAttempt
      ? `Generate p5.js code for: ${objective}`
      : `Improve this p5.js code based on the rendered image...`;
    
    const { code, critique } = await callClaude(
      prompt, 
      lastAttempt?.imageUrl
    );
    
    // 2. Run code and capture
    const imageBlob = await sketchRunner.value.render(code, { w: 512, h: 512 });
    
    // 3. Upload image (raw blob)
    const attemptId = generateId();
    const { imageUrl } = await uploadImage(imageBlob, sessionId, attemptId);
    
    // 4. Persist attempt
    const attempt = await saveAttempt({ sessionId, code, imageUrl, critique });
    history.value.push(attempt);
    
    // 5. Backoff to respect rate limits
    await new Promise(resolve => setTimeout(resolve, 1500));
  }
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
import fs from 'fs/promises';
import path from 'path';

const app = express();

app.use(cors()); // Allow requests from Vite dev server
app.use(express.json()); // For JSON endpoints

// Handle raw binary uploads for images
app.use('/api/images', express.raw({ type: 'image/webp', limit: '10mb' }));

app.use('/data', express.static('./data')); // Serve images

app.listen(3000, () => console.log('Server running on :3000'));
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

### Client-side Claude Integration

**Browser calls Claude directly**:
```typescript
// src/config.ts (git-ignored or with placeholder)
export const config = {
  anthropicKey: import.meta.env.VITE_ANTHROPIC_KEY || 'sk-ant-...',
  serverUrl: 'http://localhost:3000'
};
```

```bash
# .env (git-ignored)
VITE_ANTHROPIC_KEY=sk-ant-...
```

**Benefits**:
- Faster iteration (no server restart for prompt changes)
- Simpler server (just storage)
- All AI logic in one place (client)

### Image Handling
```typescript
// POST /api/images?sessionId=xxx&attemptId=yyy
app.post('/api/images', async (req, res) => {
  const { sessionId, attemptId } = req.query;
  const buffer = req.body; // Already a Buffer from express.raw()
  
  const imagePath = path.join('./data/sessions', sessionId, 'images', `${attemptId}.webp`);
  await fs.mkdir(path.dirname(imagePath), { recursive: true });
  await fs.writeFile(imagePath, buffer);
  
  res.json({ imageUrl: `/data/sessions/${sessionId}/images/${attemptId}.webp` });
});
```

**Client-side** (Vue):
```typescript
// Get blob from canvas
canvas.toBlob(async (blob) => {
  const response = await fetch(
    `http://localhost:3000/api/images?sessionId=${sessionId}&attemptId=${attemptId}`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'image/webp' },
      body: blob
    }
  );
  const { imageUrl } = await response.json();
}, 'image/webp', 0.85);
```

**Simple and efficient**: Raw binary transfer, no encoding/decoding overhead.

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
- ✓ Simple Express server with JSON file storage (sessions, attempts, images)
- ✓ Client-side Claude integration
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
2. **Client-side Claude calls**: Faster iteration, simpler server, acceptable for local-only prototype
3. **REST over WebSocket**: Fewer moving parts, client knows when iterations finish
4. **JSON files instead of database**: Human-readable, git-friendly, zero setup
5. **Iframe sandbox**: Safest practical way to run untrusted code
6. **WebP @ 512x512**: Good quality/size tradeoff for visual feedback

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
   mkdir server && cd server && npm init -y
   npm install express cors
   npm install -D typescript @types/node @types/express @types/cors tsx
   ```

2. **Initialize client** (already done):
   ```bash
   npm install p5 @anthropic-ai/sdk
   npm install -D @types/p5
   ```

3. **Add API key**:
   ```bash
   # .env (git-ignored)
   echo "VITE_ANTHROPIC_KEY=sk-ant-your-key-here" > .env
   ```

4. **Create folder structure**: `mkdir -p data/sessions`

5. **Create API endpoints**: Sessions, images, attempts (just read/write JSON - no AI logic)

6. **Implement Claude service**: Client-side wrapper for API calls

7. **Implement SketchRunner**: Sandboxed iframe component

8. **Implement loop**: Claude call → Render → Capture → Save

9. **Add UI**: HistoryList with auto-scroll, scoring/tagging

10. **Test**: Start with "draw a circle" and iterate

**Run** (dev mode with hot-reload):
```bash
# Terminal 1 (server)
cd server && npx tsx src/server.ts

# Terminal 2 (Vue dev server with hot-reload)
npm run dev
```

---

## Estimated Timeline

**MVP (Core + UI + Persistence)**: 2-4 days
**Full Feature Set (Scoring/Tagging/Resume)**: +1 day
**Production Hardening**: +1-2 days

**Total**: ~4-7 days for complete system
