# Captioneer7-Gemini

A compact, single-purpose batch image captioning system powered by Google Gemini's vision API. Reproduces the proven captioning workflow from Captioneer6 — system prompt loading, trigger word/classifier prefixing, batch processing with progress tracking, and ZIP export — while replacing the OpenRouter backend with native Gemini `generateContent` calls and adding a first-class **persona preset** system for switching between captioning styles instantly.

This is a browser-only React + TypeScript + Vite web application. No Tauri, no desktop shell, no video features. Images in, captions out.

---

## Functionality

### Core Features

1. **Batch Image Captioning** — Select multiple images (file picker or folder picker), send each to Gemini's vision model with a system prompt, receive a text caption, post-process it, and collect all captions into a downloadable ZIP.

2. **Persona Preset System** — A persona is a named bundle of `{ systemPrompt, triggerWord, classifier, model }`. Users can:
   - Select from a dropdown of saved personas.
   - Create a new persona by filling in fields and clicking "Save Persona".
   - Delete a persona from the dropdown.
   - Override any field temporarily without saving (edits only persist if explicitly saved).
   - Personas are stored in `localStorage` as a JSON array under the key `captioneer7_personas`.
   - A "Default" persona always exists and cannot be deleted. Its system prompt is the same default used in Captioneer6: *"You are an expert image captioner. Write a detailed description of the image inside a single paragraph, with no feedback or commentary."*

3. **System Prompt File Upload** — Upload a `.md` or `.txt` file to populate the system prompt field. The uploaded content replaces the current text in the system prompt textarea. The prompt can be locked (persisted to localStorage) or unlocked/cleared, exactly as Captioneer6 behaves.

4. **Trigger Word & Classifier Prefixing** — Optional fields prepended to every caption: `"{triggerWord} {classifier}, {caption}"`. Same post-processing rules as Captioneer6.

5. **Caption Post-Processing** — Identical pipeline to Captioneer6:
   - Trim whitespace.
   - Replace `\r\n` with `\n`.
   - Collapse multiple newlines into `. ` (period-space).
   - Strip `[bracketed_tokens]` via regex `/\[[\w_-]+\]\s*/g`.
   - Remove double periods.
   - Normalize multiple spaces to single space.
   - Prepend trigger word + classifier prefix with comma separator.

6. **Progress & Logging** — Real-time progress bar, per-file log console with colour-coded entries (info=cyan, success=green, error=red, detail=dim), completion stats (success count, failed count, elapsed time).

7. **ZIP Download** — On completion, a "Download Captions ZIP" button generates a ZIP via JSZip containing one `.txt` file per successfully captioned image, named to match the source image's base name.

8. **Settings Modal** — A modal overlay for configuring the Gemini API key (persisted in localStorage). The model name field is also editable here.

### Excluded Features (from Captioneer6)

- No video splicer / frame extractor.
- No Splicer tab or Pipeline tab.
- No Tauri integration — pure browser app.
- No video file handling whatsoever (no `extractFrame15FromVideo`, no video MIME detection).
- No `@tauri-apps/api` dependency.

---

## Technical Implementation

### Stack

| Layer | Technology |
|-------|-----------|
| Framework | React 19 + TypeScript |
| Bundler | Vite 7 |
| ZIP generation | JSZip |
| API | Google Gemini REST API (`generateContent`) |
| Styling | Inline CSS-in-JS (no CSS libraries) |
| State | React hooks + localStorage |

### Project Structure

```
captioneer7-gemini/
├── index.html
├── package.json
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── vite.config.ts
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── types/
│   │   └── index.ts
│   ├── styles/
│   │   ├── theme.ts
│   │   └── globals.css
│   ├── services/
│   │   ├── gemini.ts
│   │   ├── imageUtils.ts
│   │   └── zipService.ts
│   ├── hooks/
│   │   ├── useCaptioner.ts
│   │   ├── useLogger.ts
│   │   └── useLocalStorage.ts
│   └── components/
│       ├── Header.tsx
│       ├── CaptionerPanel.tsx
│       ├── shared/
│       │   ├── LogConsole.tsx
│       │   ├── ProgressBar.tsx
│       │   └── SettingsModal.tsx
│       └── ui/
│           ├── Button.tsx
│           ├── Input.tsx
│           └── Panel.tsx
```

### Key Differences from Captioneer6

| Area | Captioneer6 | Captioneer7-Gemini |
|------|------------|-------------------|
| API | OpenRouter (`/api/v1/chat/completions`) | Gemini (`/v1beta/models/{model}:generateContent`) |
| Tabs | 3 tabs (Splicer, Captioner, Pipeline) | No tabs — single Captioner view |
| Personas | None (manual prompt/trigger/classifier) | Full persona preset system |
| Desktop | Tauri v2 wrapper | Browser-only (Vite dev server) |
| Video | Full video extraction pipeline | No video support |
| Model default | `x-ai/grok-4-fast` | `gemini-2.5-flash` |

---

### Data Models (`src/types/index.ts`)

```typescript
// ── Captioner Types ──

export interface CaptionFile {
  file: File;
  name: string;
  size: number;
}

export interface CaptionResult {
  name: string;
  caption: string;
  success: boolean;
  error?: string;
}

export interface Persona {
  id: string;           // crypto.randomUUID(), or "default" for the built-in
  name: string;         // display name, e.g. "LoRA Training - Woman"
  systemPrompt: string;
  triggerWord: string;
  classifier: string;
  model: string;
}

export interface CaptionSettings {
  systemPrompt: string;
  triggerWord: string;
  classifier: string;
  apiKey: string;
  model: string;
}

export interface CaptionProgress {
  current: number;
  total: number;
}

export interface CaptionStats {
  success: number;
  failed: number;
  time: number;
}

// ── Log Types ──

export type LogType = 'info' | 'success' | 'error' | 'detail';

export interface LogEntry {
  time: string;
  message: string;
  type: LogType;
}

// ── Storage Keys ──

export const STORAGE_KEYS = {
  SYSTEM_PROMPT: 'captioneer7_system_prompt',
  API_KEY: 'captioneer7_api_key',
  MODEL: 'captioneer7_model',
  TRIGGER: 'captioneer7_trigger',
  CLASSIFIER: 'captioneer7_classifier',
  PROMPT_FILENAME: 'captioneer7_prompt_filename',
  PERSONAS: 'captioneer7_personas',
  ACTIVE_PERSONA_ID: 'captioneer7_active_persona',
} as const;

export const DEFAULT_SYSTEM_PROMPT =
  'You are an expert image captioner. Write a detailed description of the image inside a single paragraph, with no feedback or commentary.';

export const DEFAULT_MODEL = 'gemini-2.5-flash';

export const DEFAULT_PERSONA: Persona = {
  id: 'default',
  name: 'Default',
  systemPrompt: DEFAULT_SYSTEM_PROMPT,
  triggerWord: '',
  classifier: '',
  model: DEFAULT_MODEL,
};
```

---

### Gemini API Service (`src/services/gemini.ts`)

This replaces `openrouter.ts`. The Google Gemini REST API uses a different request format from the OpenAI-compatible format used by OpenRouter.

```typescript
import type { CaptionSettings } from '../types';
import { DEFAULT_SYSTEM_PROMPT } from '../types';

export interface CaptionRequest {
  base64Image: string;
  mimeType: string;
  settings: CaptionSettings;
}

export async function captionImage(request: CaptionRequest): Promise<string> {
  const { base64Image, mimeType, settings } = request;
  const model = settings.model || 'gemini-2.5-flash';

  const url = `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${settings.apiKey}`;

  const response = await fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      systemInstruction: {
        parts: [{ text: settings.systemPrompt || DEFAULT_SYSTEM_PROMPT }],
      },
      contents: [
        {
          parts: [
            {
              inlineData: {
                mimeType,
                data: base64Image,
              },
            },
            {
              text: 'Please describe this image.',
            },
          ],
        },
      ],
      generationConfig: {
        maxOutputTokens: 1024,
      },
    }),
  });

  if (!response.ok) {
    let errorMessage = `Gemini API request failed (${response.status})`;
    try {
      const error = await response.json();
      errorMessage = error.error?.message || errorMessage;
    } catch {
      // response not JSON
    }
    throw new Error(errorMessage);
  }

  const data = await response.json();
  return data.candidates?.[0]?.content?.parts?.[0]?.text || '';
}

export function postProcessCaption(
  rawCaption: string,
  triggerWord: string,
  classifier: string
): string {
  const processed = rawCaption
    .trim()
    .replace(/\r\n/g, '\n')
    .replace(/\n+/g, '. ')
    .replace(/\[[\w_-]+\]\s*/g, '')
    .replace(/\.\s*\./g, '.')
    .replace(/\s+/g, ' ')
    .trim();

  const prefix = [triggerWord.trim(), classifier.trim()].filter(Boolean).join(' ');
  return prefix ? `${prefix}, ${processed}` : processed;
}
```

**Key differences from `openrouter.ts`:**
- URL pattern: `https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={apiKey}` — the API key goes in the query string, not an `Authorization` header.
- No `Authorization`, `HTTP-Referer`, or `X-Title` headers needed.
- Request body uses `systemInstruction.parts[].text` instead of a system message in the messages array.
- Image data is sent as `contents[].parts[].inlineData.{mimeType, data}` instead of `image_url.url` with a data URI.
- Response path: `data.candidates[0].content.parts[0].text` instead of `data.choices[0].message.content`.
- `generationConfig.maxOutputTokens` replaces `max_tokens`.

---

### Image Utilities (`src/services/imageUtils.ts`)

Simplified from Captioneer6 — all video-related functions removed.

```typescript
const IMAGE_EXTENSIONS = ['.jpg', '.jpeg', '.png', '.gif', '.webp', '.bmp'];

export function isImageFile(filename: string): boolean {
  return IMAGE_EXTENSIONS.some(ext => filename.toLowerCase().endsWith(ext));
}

export function getBaseName(filename: string): string {
  return filename.replace(/\.[^/.]+$/, '');
}

export function formatFileSize(bytes: number): string {
  if (bytes < 1024) return bytes + ' B';
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB';
  return (bytes / (1024 * 1024)).toFixed(1) + ' MB';
}

export function fileToBase64(file: File | Blob): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      const result = reader.result as string;
      resolve(result.split(',')[1]);
    };
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
}
```

---

### ZIP Service (`src/services/zipService.ts`)

Simplified — only caption ZIP creation and download trigger retained.

```typescript
import JSZip from 'jszip';
import type { CaptionResult } from '../types';

export async function createCaptionsZip(
  results: CaptionResult[],
  onProgress?: (percent: number) => void
): Promise<Blob> {
  const zip = new JSZip();
  for (const result of results) {
    if (result.success && result.caption) {
      zip.file(`${result.name}.txt`, result.caption);
    }
  }
  return zip.generateAsync(
    { type: 'blob' },
    onProgress ? (meta) => onProgress(meta.percent) : undefined
  );
}

export function triggerDownload(blob: Blob, filename: string): void {
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  setTimeout(() => URL.revokeObjectURL(url), 1000);
}
```

---

### Hooks

#### `src/hooks/useLogger.ts`

Identical to Captioneer6:

```typescript
import { useState, useCallback, useRef, useEffect } from 'react';
import type { LogEntry, LogType } from '../types';

export function useLogger() {
  const [logs, setLogs] = useState<LogEntry[]>([]);
  const containerRef = useRef<HTMLDivElement>(null);

  const addLog = useCallback((message: string, type: LogType = 'info') => {
    const time = new Date().toLocaleTimeString('en-US', { hour12: false });
    setLogs(prev => [...prev, { time, message, type }]);
  }, []);

  const clearLogs = useCallback(() => setLogs([]), []);

  useEffect(() => {
    if (containerRef.current) {
      containerRef.current.scrollTop = containerRef.current.scrollHeight;
    }
  }, [logs]);

  return { logs, addLog, clearLogs, containerRef };
}
```

#### `src/hooks/useLocalStorage.ts`

Identical to Captioneer6:

```typescript
import { useState, useCallback } from 'react';

export function useLocalStorage<T>(key: string, defaultValue: T): [T, (value: T) => void, () => void] {
  const [state, setState] = useState<T>(() => {
    try {
      const stored = localStorage.getItem(key);
      if (stored === null) return defaultValue;
      return JSON.parse(stored) as T;
    } catch {
      return defaultValue;
    }
  });

  const setValue = useCallback((value: T) => {
    setState(value);
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch {
      // localStorage full or unavailable
    }
  }, [key]);

  const removeValue = useCallback(() => {
    setState(defaultValue);
    try {
      localStorage.removeItem(key);
    } catch {
      // ignore
    }
  }, [key, defaultValue]);

  return [state, setValue, removeValue];
}
```

#### `src/hooks/useCaptioner.ts`

Adapted from Captioneer6 — imports from `gemini.ts` instead of `openrouter.ts`, removes all video handling, adds persona support:

```typescript
import { useState, useCallback, useRef } from 'react';
import JSZip from 'jszip';
import type {
  CaptionFile, CaptionResult, CaptionSettings,
  CaptionProgress, CaptionStats, Persona, LogType,
} from '../types';
import { STORAGE_KEYS, DEFAULT_MODEL, DEFAULT_PERSONA } from '../types';
import { captionImage, postProcessCaption } from '../services/gemini';
import { fileToBase64, getBaseName } from '../services/imageUtils';

function loadPersonas(): Persona[] {
  try {
    const raw = localStorage.getItem(STORAGE_KEYS.PERSONAS);
    if (raw) {
      const parsed = JSON.parse(raw) as Persona[];
      // Ensure default persona always exists
      if (!parsed.find(p => p.id === 'default')) {
        parsed.unshift(DEFAULT_PERSONA);
      }
      return parsed;
    }
  } catch { /* ignore */ }
  return [DEFAULT_PERSONA];
}

export function useCaptioner(addLog: (msg: string, type: LogType) => void) {
  const [files, setFiles] = useState<CaptionFile[]>([]);
  const [results, setResults] = useState<CaptionResult[]>([]);
  const [isProcessing, setIsProcessing] = useState(false);
  const [isComplete, setIsComplete] = useState(false);
  const [progress, setProgress] = useState<CaptionProgress>({ current: 0, total: 0 });
  const [stats, setStats] = useState<CaptionStats>({ success: 0, failed: 0, time: 0 });

  // ── Persona state ──
  const [personas, setPersonas] = useState<Persona[]>(loadPersonas);
  const [activePersonaId, setActivePersonaId] = useState<string>(
    () => localStorage.getItem(STORAGE_KEYS.ACTIVE_PERSONA_ID) || 'default'
  );

  const [settings, setSettings] = useState<CaptionSettings>(() => ({
    systemPrompt: localStorage.getItem(STORAGE_KEYS.SYSTEM_PROMPT) || DEFAULT_PERSONA.systemPrompt,
    triggerWord: localStorage.getItem(STORAGE_KEYS.TRIGGER) || '',
    classifier: localStorage.getItem(STORAGE_KEYS.CLASSIFIER) || '',
    apiKey: localStorage.getItem(STORAGE_KEYS.API_KEY) || '',
    model: localStorage.getItem(STORAGE_KEYS.MODEL) || DEFAULT_MODEL,
  }));

  const zipRef = useRef<JSZip | null>(null);

  const updateSetting = useCallback(<K extends keyof CaptionSettings>(key: K, value: CaptionSettings[K]) => {
    setSettings(prev => ({ ...prev, [key]: value }));
  }, []);

  // ── Persona operations ──

  const selectPersona = useCallback((personaId: string) => {
    const persona = personas.find(p => p.id === personaId);
    if (!persona) return;
    setActivePersonaId(personaId);
    localStorage.setItem(STORAGE_KEYS.ACTIVE_PERSONA_ID, personaId);
    setSettings(prev => ({
      ...prev,
      systemPrompt: persona.systemPrompt,
      triggerWord: persona.triggerWord,
      classifier: persona.classifier,
      model: persona.model,
    }));
    addLog(`Persona loaded: "${persona.name}"`, 'success');
  }, [personas, addLog]);

  const savePersona = useCallback((name: string) => {
    const existing = personas.find(p => p.id === activePersonaId && p.id !== 'default');
    let updated: Persona[];
    let savedId: string;

    if (existing) {
      // Update existing persona
      updated = personas.map(p =>
        p.id === existing.id
          ? { ...p, name, systemPrompt: settings.systemPrompt, triggerWord: settings.triggerWord, classifier: settings.classifier, model: settings.model }
          : p
      );
      savedId = existing.id;
    } else {
      // Create new persona
      const newPersona: Persona = {
        id: crypto.randomUUID(),
        name,
        systemPrompt: settings.systemPrompt,
        triggerWord: settings.triggerWord,
        classifier: settings.classifier,
        model: settings.model,
      };
      updated = [...personas, newPersona];
      savedId = newPersona.id;
    }

    setPersonas(updated);
    setActivePersonaId(savedId);
    localStorage.setItem(STORAGE_KEYS.PERSONAS, JSON.stringify(updated));
    localStorage.setItem(STORAGE_KEYS.ACTIVE_PERSONA_ID, savedId);
    addLog(`Persona saved: "${name}"`, 'success');
  }, [personas, activePersonaId, settings, addLog]);

  const deletePersona = useCallback((personaId: string) => {
    if (personaId === 'default') return; // Cannot delete default
    const updated = personas.filter(p => p.id !== personaId);
    setPersonas(updated);
    localStorage.setItem(STORAGE_KEYS.PERSONAS, JSON.stringify(updated));
    if (activePersonaId === personaId) {
      selectPersona('default');
    }
    addLog('Persona deleted', 'info');
  }, [personas, activePersonaId, selectPersona, addLog]);

  // ── Processing ──

  const processFiles = useCallback(async () => {
    if (!settings.apiKey) {
      addLog('Error: API key not configured', 'error');
      return false;
    }
    if (files.length === 0) {
      addLog('Error: No files selected', 'error');
      return false;
    }

    setIsProcessing(true);
    setIsComplete(false);
    setProgress({ current: 0, total: files.length });
    setResults([]);
    const startTime = Date.now();

    // Persist current prefixes
    localStorage.setItem(STORAGE_KEYS.TRIGGER, settings.triggerWord);
    localStorage.setItem(STORAGE_KEYS.CLASSIFIER, settings.classifier);

    addLog(`Starting batch processing of ${files.length} files...`, 'info');
    addLog(`Model: ${settings.model}`, 'info');

    const zip = new JSZip();
    zipRef.current = zip;
    let successCount = 0;
    let failedCount = 0;
    const captionResults: CaptionResult[] = [];

    for (let i = 0; i < files.length; i++) {
      const cf = files[i];
      const baseName = getBaseName(cf.name);

      try {
        addLog(`Processing: ${cf.name}`, 'info');

        const mimeType = cf.file.type || 'image/jpeg';
        const base64Image = await fileToBase64(cf.file);

        addLog(`  -> Sending to Gemini vision API...`, 'detail');
        const rawCaption = await captionImage({
          base64Image,
          mimeType,
          settings,
        });

        const finalCaption = postProcessCaption(rawCaption, settings.triggerWord, settings.classifier);
        zip.file(`${baseName}.txt`, finalCaption);

        const result: CaptionResult = { name: baseName, caption: finalCaption, success: true };
        captionResults.push(result);
        setResults(prev => [...prev, result]);
        successCount++;
        addLog(`  + Caption generated for ${cf.name}`, 'success');
      } catch (error) {
        failedCount++;
        const msg = error instanceof Error ? error.message : 'Unknown error';
        addLog(`  x Failed: ${msg}`, 'error');
        const result: CaptionResult = { name: baseName, caption: '', success: false, error: msg };
        captionResults.push(result);
        setResults(prev => [...prev, result]);
      }

      setProgress({ current: i + 1, total: files.length });

      // Rate limit delay (Gemini free tier: 15 RPM, paid: higher)
      if (i < files.length - 1) {
        await new Promise(resolve => setTimeout(resolve, 500));
      }
    }

    const elapsed = Math.round((Date.now() - startTime) / 1000);
    setStats({ success: successCount, failed: failedCount, time: elapsed });
    addLog('\u2500'.repeat(40), 'info');
    addLog(`Processing complete! Success: ${successCount} | Failed: ${failedCount} | Time: ${elapsed}s`, 'success');

    setIsProcessing(false);
    setIsComplete(true);
    return true;
  }, [files, settings, addLog]);

  const resetCaptioner = useCallback(() => {
    setFiles([]);
    setResults([]);
    setIsComplete(false);
    setProgress({ current: 0, total: 0 });
    setStats({ success: 0, failed: 0, time: 0 });
    zipRef.current = null;
  }, []);

  return {
    files, setFiles, results, isProcessing, isComplete, progress, stats, settings,
    updateSetting, processFiles, zipRef, resetCaptioner,
    // Persona API
    personas, activePersonaId, selectPersona, savePersona, deletePersona,
  };
}
```

---

### Theme (`src/styles/theme.ts`)

Identical to Captioneer6 — same dark theme with orange accent and cyan highlights:

```typescript
export const theme = {
  colors: {
    bg: '#0a0e14',
    bgSecondary: '#0d1117',
    bgPanel: 'rgba(13, 17, 23, 0.8)',
    bgInput: 'rgba(0, 0, 0, 0.3)',

    border: '#21262d',
    borderLight: '#30363d',
    borderFocus: '#ff6b35',

    text: '#c9d1d9',
    textMuted: '#8b949e',
    textDim: '#484f58',
    textWhite: '#ffffff',

    accent: '#ff6b35',
    accentSecondary: '#f7931e',
    accentGlow: 'rgba(255, 107, 53, 0.3)',

    cyan: '#00d4ff',
    cyanGlow: 'rgba(0, 212, 255, 0.5)',
    mint: '#00ffc8',

    success: '#22c55e',
    error: '#ef4444',
    warning: '#ffaa00',

    gradientAccent: 'linear-gradient(135deg, #ff6b35 0%, #f7931e 100%)',
    gradientCyan: 'linear-gradient(90deg, #00d4ff 0%, #00ffc8 100%)',
    gradientBg: 'linear-gradient(135deg, #0a0e14 0%, #0d1117 50%, #0a0e14 100%)',
  },

  fonts: {
    mono: '"JetBrains Mono", "Fira Code", "SF Mono", monospace',
    display: '"Orbitron", "JetBrains Mono", sans-serif',
  },

  radii: {
    sm: '4px',
    md: '8px',
    lg: '12px',
    xl: '16px',
  },

  spacing: {
    xs: '4px',
    sm: '8px',
    md: '16px',
    lg: '24px',
    xl: '32px',
    xxl: '48px',
  },
} as const;
```

### Global CSS (`src/styles/globals.css`)

Identical to Captioneer6:

```css
@import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500;600;700&family=Orbitron:wght@500;700;900&display=swap');

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body, #root {
  height: 100%;
  width: 100%;
}

body {
  background: #0a0e14;
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', monospace;
  color: #c9d1d9;
  overflow-x: hidden;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

@keyframes slideIn {
  from { transform: translateY(10px); opacity: 0; }
  to { transform: translateY(0); opacity: 1; }
}

input::placeholder { color: #444; }
input:focus {
  border-color: #ff6b35 !important;
  box-shadow: 0 0 0 3px rgba(255, 107, 53, 0.1) !important;
  outline: none;
}

button { font-family: inherit; }
button:hover:not(:disabled) { transform: translateY(-1px); }
button:active:not(:disabled) { transform: translateY(0); }
button:disabled { cursor: not-allowed; opacity: 0.4; transform: none; }

::-webkit-scrollbar { width: 6px; height: 6px; }
::-webkit-scrollbar-track { background: rgba(0,0,0,0.2); border-radius: 3px; }
::-webkit-scrollbar-thumb { background: rgba(255,255,255,0.1); border-radius: 3px; }
::-webkit-scrollbar-thumb:hover { background: rgba(255,255,255,0.2); }
```

---

### UI Components

#### `src/components/ui/Button.tsx`

Identical to Captioneer6:

```typescript
import type { CSSProperties, ReactNode, ButtonHTMLAttributes } from 'react';
import { theme } from '../../styles/theme';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger' | 'cyan' | 'success';
  size?: 'sm' | 'md' | 'lg';
  fullWidth?: boolean;
  children: ReactNode;
}

export function Button({ variant = 'primary', size = 'md', fullWidth = false, children, style, ...props }: ButtonProps) {
  const baseStyle: CSSProperties = {
    display: 'flex',
    alignItems: 'center',
    justifyContent: 'center',
    gap: '8px',
    fontFamily: theme.fonts.mono,
    fontWeight: 600,
    letterSpacing: '1px',
    textTransform: 'uppercase',
    border: 'none',
    borderRadius: theme.radii.md,
    cursor: 'pointer',
    transition: 'all 0.2s ease',
    width: fullWidth ? '100%' : undefined,
  };

  const sizeStyles: Record<string, CSSProperties> = {
    sm: { padding: '6px 14px', fontSize: '10px' },
    md: { padding: '12px 20px', fontSize: '12px' },
    lg: { padding: '20px 40px', fontSize: '16px', fontWeight: 700, letterSpacing: '2px', borderRadius: theme.radii.lg },
  };

  const variantStyles: Record<string, CSSProperties> = {
    primary: {
      background: theme.colors.gradientAccent,
      color: '#000',
      boxShadow: `0 4px 20px ${theme.colors.accentGlow}`,
    },
    secondary: {
      background: 'transparent',
      border: `1px solid ${theme.colors.borderLight}`,
      color: theme.colors.textMuted,
    },
    danger: {
      background: 'transparent',
      border: '1px solid rgba(239, 68, 68, 0.3)',
      color: theme.colors.error,
    },
    cyan: {
      background: 'transparent',
      border: `1px solid ${theme.colors.cyan}`,
      color: theme.colors.cyan,
    },
    success: {
      background: 'linear-gradient(135deg, #22c55e 0%, #16a34a 100%)',
      color: '#fff',
      boxShadow: '0 4px 20px rgba(34, 197, 94, 0.3)',
    },
  };

  return (
    <button
      style={{
        ...baseStyle,
        ...sizeStyles[size],
        ...variantStyles[variant],
        ...style,
      }}
      {...props}
    >
      {children}
    </button>
  );
}
```

#### `src/components/ui/Input.tsx`

Identical to Captioneer6:

```typescript
import type { CSSProperties, InputHTMLAttributes } from 'react';
import { theme } from '../../styles/theme';

interface InputProps extends InputHTMLAttributes<HTMLInputElement> {
  label?: string;
}

const labelStyle: CSSProperties = {
  display: 'block',
  fontSize: '11px',
  color: '#666',
  marginBottom: '8px',
  textTransform: 'uppercase',
  letterSpacing: '1px',
  fontFamily: theme.fonts.mono,
};

const inputStyle: CSSProperties = {
  width: '100%',
  background: theme.colors.bgInput,
  border: '1px solid rgba(255,255,255,0.08)',
  borderRadius: theme.radii.md,
  padding: '14px 16px',
  color: '#fff',
  fontSize: '14px',
  fontFamily: theme.fonts.mono,
  outline: 'none',
  transition: 'all 0.2s ease',
  boxSizing: 'border-box',
};

export function Input({ label, style, ...props }: InputProps) {
  return (
    <div style={{ marginBottom: '16px' }}>
      {label && <label style={labelStyle}>{label}</label>}
      <input style={{ ...inputStyle, ...style }} {...props} />
    </div>
  );
}
```

#### `src/components/ui/Panel.tsx`

Identical to Captioneer6:

```typescript
import type { CSSProperties, ReactNode } from 'react';
import { theme } from '../../styles/theme';

interface PanelProps {
  children: ReactNode;
  number?: number;
  title?: string;
  subtitle?: string;
  style?: CSSProperties;
  fullWidth?: boolean;
}

export function Panel({ children, number, title, subtitle, style, fullWidth }: PanelProps) {
  return (
    <div style={{
      background: 'rgba(255,255,255,0.02)',
      border: '1px solid rgba(255,255,255,0.06)',
      borderRadius: theme.radii.lg,
      padding: '28px',
      position: 'relative',
      gridColumn: fullWidth ? '1 / -1' : undefined,
      ...style,
    }}>
      {(number || title) && (
        <div style={{
          display: 'flex', alignItems: 'center', gap: '12px', marginBottom: '20px',
        }}>
          {number && (
            <div style={{
              width: '28px', height: '28px',
              background: theme.colors.gradientAccent,
              borderRadius: '6px',
              display: 'flex', alignItems: 'center', justifyContent: 'center',
              fontSize: '13px', fontWeight: 700, color: '#000', flexShrink: 0,
            }}>
              {number}
            </div>
          )}
          <div>
            {title && (
              <div style={{
                fontSize: '14px', fontWeight: 600, color: '#fff',
                letterSpacing: '0.5px', textTransform: 'uppercase',
              }}>
                {title}
              </div>
            )}
            {subtitle && (
              <div style={{ fontSize: '12px', color: '#666', marginTop: '4px' }}>
                {subtitle}
              </div>
            )}
          </div>
        </div>
      )}
      {children}
    </div>
  );
}
```

---

### Shared Components

#### `src/components/shared/LogConsole.tsx`

Identical to Captioneer6:

```typescript
import type { RefObject, CSSProperties } from 'react';
import type { LogEntry } from '../../types';
import { theme } from '../../styles/theme';

interface LogConsoleProps {
  logs: LogEntry[];
  containerRef: RefObject<HTMLDivElement | null>;
  maxHeight?: string;
  title?: string;
}

const logColors: Record<string, string> = {
  info: theme.colors.cyan,
  success: theme.colors.success,
  error: theme.colors.error,
  detail: theme.colors.textDim,
};

export function LogConsole({ logs, containerRef, maxHeight = '200px', title }: LogConsoleProps) {
  const sectionStyle: CSSProperties = {
    background: theme.colors.bgPanel,
    border: `1px solid ${theme.colors.border}`,
    borderRadius: theme.radii.md,
    padding: theme.spacing.md,
  };

  const headerStyle: CSSProperties = {
    display: 'flex', alignItems: 'center', gap: '10px',
    marginBottom: '12px', paddingBottom: '12px',
    borderBottom: `1px solid ${theme.colors.border}`,
  };

  const consoleStyle: CSSProperties = {
    background: '#010409',
    border: `1px solid ${theme.colors.border}`,
    borderRadius: theme.radii.sm,
    padding: '12px',
    maxHeight,
    overflowY: 'auto',
    fontFamily: theme.fonts.mono,
    fontSize: '11px',
  };

  return (
    <div style={sectionStyle}>
      {title && (
        <div style={headerStyle}>
          <span style={{ color: theme.colors.cyan, fontSize: '14px' }}>{'\u25A4'}</span>
          <span style={{
            fontFamily: theme.fonts.display, fontSize: '12px', fontWeight: 700,
            letterSpacing: '2px', color: theme.colors.textMuted,
          }}>
            {title}
          </span>
        </div>
      )}
      <div style={consoleStyle} ref={containerRef}>
        {logs.length === 0 ? (
          <p style={{ color: theme.colors.borderLight, fontStyle: 'italic' }}>Awaiting input...</p>
        ) : (
          logs.map((log, i) => (
            <div key={i} style={{
              padding: '3px 0',
              borderBottom: '1px solid rgba(255,255,255,0.03)',
              display: 'flex', gap: '12px',
            }}>
              <span style={{ color: '#555', flexShrink: 0 }}>[{log.time}]</span>
              <span style={{ color: logColors[log.type] || theme.colors.textMuted }}>
                {log.message}
              </span>
            </div>
          ))
        )}
      </div>
    </div>
  );
}
```

#### `src/components/shared/ProgressBar.tsx`

Identical to Captioneer6:

```typescript
import type { CSSProperties } from 'react';
import { theme } from '../../styles/theme';

interface ProgressBarProps {
  percent: number;
  variant?: 'accent' | 'cyan';
  height?: number;
  label?: string;
}

export function ProgressBar({ percent, variant = 'accent', height = 8, label }: ProgressBarProps) {
  const fillGradient = variant === 'cyan' ? theme.colors.gradientCyan : theme.colors.gradientAccent;

  const containerStyle: CSSProperties = {
    height: `${height}px`,
    background: 'rgba(0,0,0,0.3)',
    borderRadius: `${height / 2}px`,
    overflow: 'hidden',
    marginBottom: label ? '8px' : '0',
  };

  const fillStyle: CSSProperties = {
    height: '100%',
    background: fillGradient,
    borderRadius: `${height / 2}px`,
    transition: 'width 0.3s ease',
    width: `${Math.min(100, Math.max(0, percent))}%`,
  };

  return (
    <div>
      <div style={containerStyle}>
        <div style={fillStyle} />
      </div>
      {label && (
        <div style={{ fontSize: '11px', color: theme.colors.textMuted, letterSpacing: '0.5px' }}>
          {label}
        </div>
      )}
    </div>
  );
}
```

#### `src/components/shared/SettingsModal.tsx`

Adapted for Gemini — label says "Gemini API Key", placeholder shows Gemini key pattern:

```typescript
import type { CSSProperties } from 'react';
import { theme } from '../../styles/theme';
import { Input } from '../ui/Input';
import { Button } from '../ui/Button';

interface SettingsModalProps {
  apiKey: string;
  model: string;
  onApiKeyChange: (value: string) => void;
  onModelChange: (value: string) => void;
  onSave: () => void;
  onClose: () => void;
}

export function SettingsModal({ apiKey, model, onApiKeyChange, onModelChange, onSave, onClose }: SettingsModalProps) {
  const overlayStyle: CSSProperties = {
    position: 'fixed',
    top: 0, left: 0, right: 0, bottom: 0,
    background: 'rgba(0,0,0,0.8)',
    backdropFilter: 'blur(8px)',
    display: 'flex',
    alignItems: 'center',
    justifyContent: 'center',
    zIndex: 1000,
    animation: 'fadeIn 0.2s ease',
  };

  const modalStyle: CSSProperties = {
    background: 'linear-gradient(135deg, #1a1a1c 0%, #141416 100%)',
    border: '1px solid rgba(255,255,255,0.1)',
    borderRadius: theme.radii.xl,
    padding: '36px',
    width: '500px',
    maxWidth: '90vw',
    maxHeight: '80vh',
    overflowY: 'auto',
    boxShadow: '0 20px 60px rgba(0,0,0,0.5)',
  };

  return (
    <div style={overlayStyle} onClick={onClose} onKeyDown={(e) => { if (e.key === 'Escape') onClose(); }}>
      <div style={modalStyle} onClick={(e) => e.stopPropagation()}>
        <div style={{
          display: 'flex', alignItems: 'center', justifyContent: 'space-between',
          marginBottom: '30px', paddingBottom: '20px',
          borderBottom: '1px solid rgba(255,255,255,0.06)',
        }}>
          <span style={{ fontSize: '20px', fontWeight: 700, color: '#fff' }}>API Settings</span>
          <button
            style={{
              background: 'transparent', border: 'none', color: '#666',
              fontSize: '24px', cursor: 'pointer', padding: '4px', lineHeight: 1,
            }}
            onClick={onClose}
          >
            x
          </button>
        </div>

        <Input
          label="Gemini API Key"
          type="password"
          value={apiKey}
          onChange={(e) => onApiKeyChange(e.target.value)}
          placeholder="AIza..."
        />

        <Input
          label="Model Name"
          type="text"
          value={model}
          onChange={(e) => onModelChange(e.target.value)}
          placeholder="gemini-2.5-flash"
        />

        <div style={{ fontSize: '12px', color: '#666', marginTop: '8px', marginBottom: '16px' }}>
          Your API key is stored locally and never sent except to Google's Gemini API.
        </div>

        <Button variant="primary" fullWidth onClick={() => { onSave(); onClose(); }}>
          Save Settings
        </Button>
      </div>
    </div>
  );
}
```

---

### Header (`src/components/Header.tsx`)

Updated branding — title says "CAPTIONEER7", version badge says "GEMINI":

```typescript
import type { CSSProperties } from 'react';
import { theme } from '../styles/theme';

interface HeaderProps {
  onSettingsClick: () => void;
}

export function Header({ onSettingsClick }: HeaderProps) {
  const headerStyle: CSSProperties = {
    display: 'flex',
    alignItems: 'center',
    justifyContent: 'space-between',
    padding: '20px 30px',
    borderBottom: '1px solid rgba(255,255,255,0.06)',
  };

  const logoStyle: CSSProperties = {
    display: 'flex', alignItems: 'center', gap: '16px',
  };

  const iconStyle: CSSProperties = {
    width: '42px', height: '42px',
    background: theme.colors.gradientAccent,
    borderRadius: '8px',
    display: 'flex', alignItems: 'center', justifyContent: 'center',
    fontSize: '20px',
    boxShadow: `0 4px 20px ${theme.colors.accentGlow}`,
  };

  const titleStyle: CSSProperties = {
    fontFamily: theme.fonts.display,
    fontSize: '24px', fontWeight: 900, letterSpacing: '3px',
    background: 'linear-gradient(90deg, #ffffff 0%, #a0a0a0 100%)',
    WebkitBackgroundClip: 'text',
    WebkitTextFillColor: 'transparent',
  };

  const versionStyle: CSSProperties = {
    fontSize: '10px', color: '#666',
    padding: '3px 8px',
    background: 'rgba(255,255,255,0.05)',
    borderRadius: '4px', letterSpacing: '1px',
  };

  const settingsBtnStyle: CSSProperties = {
    background: 'rgba(255,255,255,0.05)',
    border: '1px solid rgba(255,255,255,0.1)',
    borderRadius: '8px',
    padding: '10px 18px', color: '#888',
    cursor: 'pointer', fontSize: '12px',
    fontFamily: theme.fonts.mono,
    display: 'flex', alignItems: 'center', gap: '8px',
    transition: 'all 0.2s ease',
  };

  return (
    <header style={headerStyle}>
      <div style={logoStyle}>
        <div style={iconStyle}>{'\u26A1'}</div>
        <span style={titleStyle}>CAPTIONEER7</span>
        <span style={versionStyle}>GEMINI</span>
      </div>
      <button style={settingsBtnStyle} onClick={onSettingsClick}>
        {'\u2699'} Settings
      </button>
    </header>
  );
}
```

---

### CaptionerPanel (`src/components/CaptionerPanel.tsx`)

This is the main UI panel. It combines Captioneer6's CaptionerPanel layout with the new persona dropdown. No tab system — this is the only panel. The 2x2 grid layout is retained. A persona selector bar is added above the grid.

**Layout (top to bottom):**

1. **Source Bar** — horizontal bar with "FROM FOLDER" and "FROM FILES" buttons (no "FROM SPLICER" since there is no splicer).
2. **Persona Bar** — horizontal bar with: persona `<select>` dropdown, "Save Persona" button (opens inline name input), "Delete" button (only visible when non-default persona is active).
3. **2x2 Grid**:
   - Panel 1: System Prompt (upload zone / locked state / textarea preview) — same as Captioneer6.
   - Panel 2: Selected Files list — same as Captioneer6 but images only, no video icons.
   - Panel 3: Trigger Word input.
   - Panel 4: Classifier input.
4. **Start Button** — full-width primary button.
5. **Progress Section** — progress bar, stats grid (on completion), log console, download button.

```typescript
import { useRef, useState, type CSSProperties, type DragEvent } from 'react';
import type { CaptionFile, CaptionSettings, CaptionProgress, CaptionStats, Persona, LogEntry, LogType } from '../types';
import { STORAGE_KEYS } from '../types';
import { theme } from '../styles/theme';
import { Button } from './ui/Button';
import { Panel } from './ui/Panel';
import { Input } from './ui/Input';
import { ProgressBar } from './shared/ProgressBar';
import { LogConsole } from './shared/LogConsole';
import { isImageFile, formatFileSize } from '../services/imageUtils';
import { triggerDownload } from '../services/zipService';
import JSZip from 'jszip';

interface CaptionerPanelProps {
  files: CaptionFile[];
  isProcessing: boolean;
  isComplete: boolean;
  progress: CaptionProgress;
  stats: CaptionStats;
  settings: CaptionSettings;
  logs: LogEntry[];
  logContainerRef: React.RefObject<HTMLDivElement | null>;
  zipRef: React.RefObject<JSZip | null>;
  onSetFiles: (files: CaptionFile[]) => void;
  onUpdateSetting: <K extends keyof CaptionSettings>(key: K, value: CaptionSettings[K]) => void;
  onProcess: () => void;
  addLog: (msg: string, type: LogType) => void;
  // Persona props
  personas: Persona[];
  activePersonaId: string;
  onSelectPersona: (id: string) => void;
  onSavePersona: (name: string) => void;
  onDeletePersona: (id: string) => void;
}

export function CaptionerPanel({
  files, isProcessing, isComplete, progress, stats, settings, logs, logContainerRef,
  zipRef, onSetFiles, onUpdateSetting, onProcess, addLog,
  personas, activePersonaId, onSelectPersona, onSavePersona, onDeletePersona,
}: CaptionerPanelProps) {
  const fileInputRef = useRef<HTMLInputElement>(null);
  const promptInputRef = useRef<HTMLInputElement>(null);
  const [isPromptLocked, setIsPromptLocked] = useState(() => !!localStorage.getItem(STORAGE_KEYS.SYSTEM_PROMPT));
  const [promptFileName, setPromptFileName] = useState(() => localStorage.getItem(STORAGE_KEYS.PROMPT_FILENAME) || '');
  const [isSavingPersona, setIsSavingPersona] = useState(false);
  const [personaNameInput, setPersonaNameInput] = useState('');

  // ── File selection (images only) ──
  const handleFilesSelect = (fileList: FileList | null) => {
    if (!fileList) return;
    const selected = Array.from(fileList);
    const valid = selected.filter(f => isImageFile(f.name));
    const captionFiles: CaptionFile[] = valid.map(f => ({
      file: f,
      name: f.name,
      size: f.size,
    }));
    onSetFiles(captionFiles);
    addLog(`Selected ${captionFiles.length} image files for captioning`, 'info');
  };

  // ── Prompt upload ──
  const handlePromptUpload = (file: File) => {
    const reader = new FileReader();
    reader.onload = (event) => {
      const content = event.target?.result as string;
      onUpdateSetting('systemPrompt', content);
      setPromptFileName(file.name);
      setIsPromptLocked(false);
    };
    reader.readAsText(file);
  };

  const lockPrompt = () => {
    localStorage.setItem(STORAGE_KEYS.SYSTEM_PROMPT, settings.systemPrompt);
    localStorage.setItem(STORAGE_KEYS.PROMPT_FILENAME, promptFileName);
    setIsPromptLocked(true);
    addLog(`System prompt locked: "${promptFileName}"`, 'success');
  };

  const unlockPrompt = () => {
    localStorage.removeItem(STORAGE_KEYS.SYSTEM_PROMPT);
    localStorage.removeItem(STORAGE_KEYS.PROMPT_FILENAME);
    setIsPromptLocked(false);
    onUpdateSetting('systemPrompt', '');
    setPromptFileName('');
    addLog('System prompt unlocked and cleared', 'info');
  };

  const handlePromptDrop = (e: DragEvent) => {
    e.preventDefault();
    const file = e.dataTransfer.files[0];
    if (file) handlePromptUpload(file);
  };

  // ── Download ──
  const downloadCaptions = async () => {
    if (!zipRef.current) return;
    addLog('Generating captions ZIP...', 'info');
    const blob = await zipRef.current.generateAsync({ type: 'blob' });
    const filename = `captions_${new Date().toISOString().slice(0, 10)}.zip`;
    triggerDownload(blob, filename);
    addLog(`Downloaded: ${filename}`, 'success');
  };

  // ── Persona save flow ──
  const handleSavePersona = () => {
    if (!personaNameInput.trim()) return;
    onSavePersona(personaNameInput.trim());
    setIsSavingPersona(false);
    setPersonaNameInput('');
  };

  const canStart = files.length > 0 && settings.apiKey && !isProcessing;
  const progressPercent = progress.total > 0 ? (progress.current / progress.total) * 100 : 0;

  const containerStyle: CSSProperties = {
    maxWidth: '1100px', margin: '0 auto', padding: '24px 30px',
    display: 'flex', flexDirection: 'column', gap: '24px',
  };

  const gridStyle: CSSProperties = {
    display: 'grid', gridTemplateColumns: '1fr 1fr', gap: '24px',
  };

  const barStyle: CSSProperties = {
    background: theme.colors.bgPanel, border: `1px solid ${theme.colors.border}`,
    borderRadius: theme.radii.md, padding: theme.spacing.md,
    display: 'flex', gap: '12px', alignItems: 'center', flexWrap: 'wrap',
  };

  const barLabelStyle: CSSProperties = {
    fontFamily: theme.fonts.display, fontSize: '12px', fontWeight: 700,
    letterSpacing: '2px', color: theme.colors.textMuted,
  };

  const uploadZone: CSSProperties = {
    border: '2px dashed rgba(255,255,255,0.1)', borderRadius: theme.radii.md,
    padding: '40px', textAlign: 'center', cursor: 'pointer',
    transition: 'all 0.3s ease', background: 'rgba(0,0,0,0.2)',
  };

  const lockedZone: CSSProperties = {
    ...uploadZone, borderColor: theme.colors.success, borderStyle: 'solid',
    background: 'rgba(34, 197, 94, 0.05)', cursor: 'default',
  };

  const selectStyle: CSSProperties = {
    background: theme.colors.bgInput,
    border: '1px solid rgba(255,255,255,0.08)',
    borderRadius: theme.radii.md,
    padding: '8px 12px',
    color: '#fff',
    fontSize: '12px',
    fontFamily: theme.fonts.mono,
    outline: 'none',
    minWidth: '200px',
  };

  return (
    <div style={containerStyle}>
      {/* Source Bar */}
      <div style={barStyle}>
        <span style={barLabelStyle}>SOURCE:</span>
        <Button variant="secondary" size="sm" onClick={() => {
          const input = document.createElement('input');
          input.type = 'file';
          input.accept = 'image/*';
          input.multiple = true;
          input.setAttribute('webkitdirectory', '');
          input.onchange = () => handleFilesSelect(input.files);
          input.click();
        }}>
          FROM FOLDER
        </Button>
        <Button variant="secondary" size="sm" onClick={() => fileInputRef.current?.click()}>
          FROM FILES
        </Button>
        <input ref={fileInputRef} type="file" accept="image/*" multiple onChange={(e) => handleFilesSelect(e.target.files)} style={{ display: 'none' }} />
      </div>

      {/* Persona Bar */}
      <div style={barStyle}>
        <span style={barLabelStyle}>PERSONA:</span>
        <select
          style={selectStyle}
          value={activePersonaId}
          onChange={(e) => onSelectPersona(e.target.value)}
        >
          {personas.map(p => (
            <option key={p.id} value={p.id}>{p.name}</option>
          ))}
        </select>

        {isSavingPersona ? (
          <>
            <input
              style={{ ...selectStyle, minWidth: '160px' }}
              type="text"
              placeholder="Persona name..."
              value={personaNameInput}
              onChange={(e) => setPersonaNameInput(e.target.value)}
              onKeyDown={(e) => { if (e.key === 'Enter') handleSavePersona(); }}
              autoFocus
            />
            <Button variant="success" size="sm" onClick={handleSavePersona}>Save</Button>
            <Button variant="secondary" size="sm" onClick={() => setIsSavingPersona(false)}>Cancel</Button>
          </>
        ) : (
          <Button variant="cyan" size="sm" onClick={() => {
            const active = personas.find(p => p.id === activePersonaId);
            setPersonaNameInput(active?.id === 'default' ? '' : active?.name || '');
            setIsSavingPersona(true);
          }}>
            Save Persona
          </Button>
        )}

        {activePersonaId !== 'default' && !isSavingPersona && (
          <Button variant="danger" size="sm" onClick={() => onDeletePersona(activePersonaId)}>
            Delete
          </Button>
        )}
      </div>

      {/* 2x2 Grid */}
      <div style={gridStyle}>
        {/* Panel 1: System Prompt */}
        <Panel number={1} title="System Prompt" subtitle="Upload your captioning instruction file">
          <input ref={promptInputRef} type="file" accept=".md,.txt" onChange={(e) => { if (e.target.files?.[0]) handlePromptUpload(e.target.files[0]); }} style={{ display: 'none' }} />

          {isPromptLocked ? (
            <div style={lockedZone}>
              <div style={{
                display: 'inline-flex', alignItems: 'center', gap: '6px',
                background: 'rgba(34, 197, 94, 0.15)', color: theme.colors.success,
                padding: '6px 12px', borderRadius: '20px', fontSize: '12px', fontWeight: 600, marginBottom: '12px',
              }}>
                LOCKED
              </div>
              <div style={{ fontSize: '13px', color: '#888', marginBottom: '8px' }}>{promptFileName}</div>
              <div style={{ fontSize: '11px', color: '#555' }}>
                {settings.systemPrompt.slice(0, 100)}...
              </div>
              <Button variant="danger" size="sm" onClick={unlockPrompt} style={{ marginTop: '12px' }}>
                Unlock & Clear
              </Button>
            </div>
          ) : settings.systemPrompt ? (
            <div style={uploadZone}>
              <div style={{ fontSize: '13px', color: '#888', marginBottom: '8px' }}>{promptFileName}</div>
              <div style={{ fontSize: '11px', color: '#555', marginBottom: '12px' }}>
                {settings.systemPrompt.slice(0, 100)}...
              </div>
              <Button variant="primary" size="sm" onClick={lockPrompt}>
                Lock Prompt
              </Button>
            </div>
          ) : (
            <div
              style={uploadZone}
              onClick={() => promptInputRef.current?.click()}
              onDragOver={(e) => e.preventDefault()}
              onDrop={handlePromptDrop}
            >
              <div style={{ fontSize: '32px', marginBottom: '12px', opacity: 0.5 }}>{'📄'}</div>
              <div style={{ fontSize: '13px', color: '#888', marginBottom: '8px' }}>Drop .md file or click to upload</div>
              <div style={{ fontSize: '11px', color: '#555' }}>Custom system prompt for captioning</div>
            </div>
          )}
        </Panel>

        {/* Panel 2: File List */}
        <Panel number={2} title="Selected Files" subtitle="Images to caption">
          {files.length > 0 ? (
            <div style={{ maxHeight: '240px', overflowY: 'auto', background: 'rgba(0,0,0,0.2)', borderRadius: theme.radii.md, padding: '12px' }}>
              {files.slice(0, 10).map((f, i) => (
                <div key={i} style={{
                  display: 'flex', alignItems: 'center', gap: '10px', padding: '8px 0',
                  borderBottom: '1px solid rgba(255,255,255,0.04)', fontSize: '12px',
                }}>
                  <div style={{
                    width: '24px', height: '24px', background: 'rgba(255, 107, 53, 0.1)',
                    borderRadius: '4px', display: 'flex', alignItems: 'center', justifyContent: 'center', fontSize: '10px',
                  }}>
                    IMG
                  </div>
                  <span style={{ flex: 1, color: '#aaa', overflow: 'hidden', textOverflow: 'ellipsis', whiteSpace: 'nowrap' }}>
                    {f.name}
                  </span>
                  <span style={{ color: '#555', fontSize: '11px' }}>{formatFileSize(f.size)}</span>
                </div>
              ))}
              {files.length > 10 && (
                <div style={{ padding: '8px 0', color: '#666', textAlign: 'center', fontSize: '12px' }}>
                  ... and {files.length - 10} more files
                </div>
              )}
            </div>
          ) : (
            <div style={{ ...uploadZone, padding: '30px' }}>
              <div style={{ fontSize: '32px', marginBottom: '12px', opacity: 0.5 }}>{'🖼'}</div>
              <div style={{ fontSize: '13px', color: '#888' }}>No files selected</div>
              <div style={{ fontSize: '11px', color: '#555' }}>Use the source buttons above</div>
            </div>
          )}
        </Panel>

        {/* Panel 3: Trigger Word */}
        <Panel number={3} title="Trigger Word" subtitle="Optional prefix for all captions">
          <Input
            type="text"
            value={settings.triggerWord}
            onChange={(e) => onUpdateSetting('triggerWord', e.target.value)}
            placeholder="e.g., sks, ohwx, xyz"
          />
        </Panel>

        {/* Panel 4: Classifier */}
        <Panel number={4} title="Classifier" subtitle="Subject type or category">
          <Input
            type="text"
            value={settings.classifier}
            onChange={(e) => onUpdateSetting('classifier', e.target.value)}
            placeholder="e.g., woman, man, style, object"
          />
        </Panel>
      </div>

      {/* Start Button */}
      <Button variant="primary" size="lg" fullWidth disabled={!canStart} onClick={onProcess}>
        {isProcessing ? (
          <><span style={{ animation: 'spin 1s linear infinite', display: 'inline-block' }}>{'\u2699'}</span> Processing...</>
        ) : (
          <>{'\u25B6'} Start Captioning</>
        )}
      </Button>

      {/* Progress Section */}
      {(isProcessing || logs.length > 0) && (
        <div style={{
          background: 'rgba(255, 107, 53, 0.03)', border: '1px solid rgba(255, 107, 53, 0.15)',
          borderRadius: theme.radii.lg, padding: '28px',
        }}>
          <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: '16px' }}>
            <span style={{ fontSize: '14px', fontWeight: 600, color: theme.colors.accent }}>
              {isProcessing ? 'Processing...' : isComplete ? 'Complete' : 'Ready'}
            </span>
            <span style={{ fontSize: '13px', color: '#888' }}>
              {progress.current} / {progress.total}
            </span>
          </div>

          <ProgressBar percent={progressPercent} variant="accent" height={8} />

          {isComplete && (
            <div style={{ display: 'grid', gridTemplateColumns: 'repeat(3, 1fr)', gap: '16px', marginTop: '20px', marginBottom: '20px' }}>
              <div style={{ background: 'rgba(0,0,0,0.2)', borderRadius: theme.radii.md, padding: '16px', textAlign: 'center' }}>
                <div style={{ fontSize: '24px', fontWeight: 700, color: theme.colors.accent }}>{stats.success}</div>
                <div style={{ fontSize: '11px', color: '#666', marginTop: '4px', textTransform: 'uppercase', letterSpacing: '1px' }}>Success</div>
              </div>
              <div style={{ background: 'rgba(0,0,0,0.2)', borderRadius: theme.radii.md, padding: '16px', textAlign: 'center' }}>
                <div style={{ fontSize: '24px', fontWeight: 700, color: stats.failed > 0 ? theme.colors.error : theme.colors.success }}>{stats.failed}</div>
                <div style={{ fontSize: '11px', color: '#666', marginTop: '4px', textTransform: 'uppercase', letterSpacing: '1px' }}>Failed</div>
              </div>
              <div style={{ background: 'rgba(0,0,0,0.2)', borderRadius: theme.radii.md, padding: '16px', textAlign: 'center' }}>
                <div style={{ fontSize: '24px', fontWeight: 700, color: theme.colors.accent }}>{stats.time}s</div>
                <div style={{ fontSize: '11px', color: '#666', marginTop: '4px', textTransform: 'uppercase', letterSpacing: '1px' }}>Time</div>
              </div>
            </div>
          )}

          <LogConsole logs={logs} containerRef={logContainerRef} maxHeight="200px" />

          {isComplete && stats.success > 0 && (
            <Button variant="success" size="lg" fullWidth onClick={downloadCaptions} style={{ marginTop: '20px' }}>
              Download Captions ZIP
            </Button>
          )}
        </div>
      )}
    </div>
  );
}
```

---

### Root App (`src/App.tsx`)

Simplified — no tabs, no splicer, no pipeline. Single captioner view:

```typescript
import { useState, type CSSProperties } from 'react';
import { STORAGE_KEYS } from './types';
import { theme } from './styles/theme';

import { useLogger } from './hooks/useLogger';
import { useCaptioner } from './hooks/useCaptioner';

import { Header } from './components/Header';
import { CaptionerPanel } from './components/CaptionerPanel';
import { SettingsModal } from './components/shared/SettingsModal';

export default function App() {
  const [showSettings, setShowSettings] = useState(false);

  const logger = useLogger();
  const captioner = useCaptioner(logger.addLog);

  const [tempApiKey, setTempApiKey] = useState(captioner.settings.apiKey);
  const [tempModel, setTempModel] = useState(captioner.settings.model);

  const handleOpenSettings = () => {
    setTempApiKey(captioner.settings.apiKey);
    setTempModel(captioner.settings.model);
    setShowSettings(true);
  };

  const handleSaveSettings = () => {
    captioner.updateSetting('apiKey', tempApiKey);
    captioner.updateSetting('model', tempModel);
    localStorage.setItem(STORAGE_KEYS.API_KEY, tempApiKey);
    localStorage.setItem(STORAGE_KEYS.MODEL, tempModel);
    logger.addLog('API settings updated', 'success');
  };

  const appStyle: CSSProperties = {
    display: 'flex',
    flexDirection: 'column',
    minHeight: '100vh',
    background: theme.colors.gradientBg,
    fontFamily: theme.fonts.mono,
    color: theme.colors.text,
  };

  const mainStyle: CSSProperties = {
    flex: 1,
    overflowY: 'auto',
  };

  const footerStyle: CSSProperties = {
    display: 'flex',
    alignItems: 'center',
    justifyContent: 'center',
    padding: '16px',
    borderTop: `1px solid ${theme.colors.border}`,
    fontSize: '11px',
    color: theme.colors.textDim,
    letterSpacing: '2px',
    fontFamily: theme.fonts.display,
  };

  return (
    <div style={appStyle}>
      <Header onSettingsClick={handleOpenSettings} />

      <main style={mainStyle}>
        <CaptionerPanel
          files={captioner.files}
          isProcessing={captioner.isProcessing}
          isComplete={captioner.isComplete}
          progress={captioner.progress}
          stats={captioner.stats}
          settings={captioner.settings}
          logs={logger.logs}
          logContainerRef={logger.containerRef}
          zipRef={captioner.zipRef}
          onSetFiles={captioner.setFiles}
          onUpdateSetting={captioner.updateSetting}
          onProcess={captioner.processFiles}
          addLog={logger.addLog}
          personas={captioner.personas}
          activePersonaId={captioner.activePersonaId}
          onSelectPersona={captioner.selectPersona}
          onSavePersona={captioner.savePersona}
          onDeletePersona={captioner.deletePersona}
        />
      </main>

      <footer style={footerStyle}>
        CAPTIONEER7 — IMAGES + PERSONAS = CAPTIONS
      </footer>

      {showSettings && (
        <SettingsModal
          apiKey={tempApiKey}
          model={tempModel}
          onApiKeyChange={setTempApiKey}
          onModelChange={setTempModel}
          onSave={handleSaveSettings}
          onClose={() => setShowSettings(false)}
        />
      )}
    </div>
  );
}
```

---

### Entry Point (`src/main.tsx`)

Identical to Captioneer6:

```typescript
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './styles/globals.css'
import App from './App.tsx'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

---

### `index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>CAPTIONEER7 — Gemini Batch Image Captioner</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

## Configuration Files

### `package.json`

```json
{
  "name": "captioneer7-gemini",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "eslint .",
    "preview": "vite preview"
  },
  "dependencies": {
    "jszip": "^3.10.1",
    "react": "^19.2.0",
    "react-dom": "^19.2.0"
  },
  "devDependencies": {
    "@eslint/js": "^9.39.1",
    "@types/react": "^19.2.7",
    "@types/react-dom": "^19.2.3",
    "@vitejs/plugin-react": "^5.1.1",
    "eslint": "^9.39.1",
    "eslint-plugin-react-hooks": "^7.0.1",
    "eslint-plugin-react-refresh": "^0.4.24",
    "globals": "^16.5.0",
    "typescript": "~5.9.3",
    "typescript-eslint": "^8.48.0",
    "vite": "^7.3.1"
  }
}
```

**Removed from Captioneer6:** `@tauri-apps/api`, `@tauri-apps/cli`, `@types/node`. No Tauri dependency.

### `vite.config.ts`

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    strictPort: false,
  },
  build: {
    target: 'esnext',
    assetsInlineLimit: 0,
  },
});
```

**Changes from Captioneer6:** Port changed to Vite default `5173` (was `1420` for Tauri). Removed COOP/COEP headers (only needed for SharedArrayBuffer in video processing). Removed Tauri-specific `clearScreen`, `envPrefix` options.

### `tsconfig.json`

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}
```

### `tsconfig.app.json`

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "target": "ES2022",
    "useDefineForClassFields": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["src"]
}
```

---

## Build & Run Sequence

```bash
# 1. Create project directory
mkdir captioneer7-gemini && cd captioneer7-gemini

# 2. Initialize and install
npm init -y
# (then replace package.json contents with the version above)
npm install

# 3. Create directory structure
mkdir -p src/types src/styles src/services src/hooks src/components/ui src/components/shared

# 4. Create all files per the structure above

# 5. Run development server
npm run dev
# Opens at http://localhost:5173

# 6. Production build
npm run build
# Output in dist/
```

---

## Testing Scenarios

1. **No API key** — Click "Start Captioning" without configuring API key. Expect: error log "Error: API key not configured", no processing starts.

2. **No files selected** — Click "Start Captioning" with API key but no images. Expect: error log "Error: No files selected".

3. **Single image** — Select one `.jpg` file, configure API key, click Start. Expect: progress 1/1, caption generated, ZIP downloadable with one `.txt` file.

4. **Batch of 20 images** — Select a folder with 20 images. Expect: progress updates 1/20 through 20/20, ~500ms delay between each, stats show totals on completion.

5. **Mixed valid/invalid files** — Select files including `.pdf` and `.docx` alongside images. Expect: only image files appear in the file list.

6. **Persona save/load** — Fill in system prompt + trigger word + classifier, save as persona "LoRA Woman". Switch to Default persona (fields clear). Switch back to "LoRA Woman" (fields restore). Reload page — persona persists.

7. **Persona delete** — Create persona, delete it. Expect: dropdown reverts to Default, persona no longer in list.

8. **System prompt lock/unlock** — Upload a `.md` file, lock it, reload the page. Expect: prompt persists. Unlock and clear — prompt field empty.

9. **API error** — Use invalid API key. Expect: per-file error logged with Gemini error message, failed count increments, processing continues to next file.

10. **Large batch** — 100+ images. Expect: UI remains responsive, progress bar updates smoothly, log auto-scrolls.

---

## Gemini API Key Setup

Users need a Google Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey). The key format starts with `AIza`. Free tier allows 15 requests per minute for `gemini-2.5-flash`. The 500ms delay between requests keeps the app under this limit for sequential processing.
