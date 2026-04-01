# Captioneer7-Gemini

Batch image captioning powered by Google Gemini's vision API. Load images, describe them with AI, and export captions as a ZIP — all from your browser or as a native desktop app.

Captioneer7-Gemini is the Gemini-native successor to [Captioneer6](https://github.com/MushroomFleet/Captioneer6), stripped down to focus entirely on image captioning with a new **persona preset system** for switching between captioning styles instantly.

## 🖥️ Get Started

### Option 1: Windows Installer (Recommended)

Download the latest **MSI installer** from [Releases](https://github.com/MushroomFleet/captioneer7-gemini/releases) and install. No dependencies required — everything is bundled.

### Option 2: Live Demo

Try it directly in your browser — no install needed:

**[https://scuffedepoch.com/captioneer7/](https://scuffedepoch.com/captioneer7/)**

---

## ✨ Features

- **Batch Image Captioning** — Select multiple images via file picker or folder picker, caption them all in one run with Gemini's vision models, and download results as a ZIP of `.txt` files.
- **Persona Presets** — Save and switch between named captioning profiles. Each persona bundles a system prompt, trigger word, classifier, and model into a one-click preset.
- **System Prompt Upload** — Load a `.md` or `.txt` file to set your captioning instructions. Lock the prompt to persist it across sessions.
- **Trigger Word & Classifier Prefixing** — Automatically prepend trigger words and classifiers to every caption, formatted as `"triggerWord classifier, caption"` — ideal for LoRA training datasets.
- **Caption Post-Processing** — Automatic cleanup: whitespace normalization, bracketed token removal, double period cleanup, and newline collapsing.
- **Real-Time Progress** — Live progress bar, colour-coded log console, and completion stats (success/failed counts, elapsed time).
- **ZIP Export** — One-click download of all captions as a ZIP archive, with each `.txt` file named to match its source image.

## 🔑 Setup

1. Get a **Google Gemini API key** from [Google AI Studio](https://aistudio.google.com/apikey).
2. Open the app and click the **Settings** gear icon.
3. Paste your API key and click Save.

Your API key is stored locally in your browser's `localStorage` — it is never sent anywhere except directly to Google's Gemini API.

## 📖 How to Use

1. **Choose a Persona** — Select a captioning style from the persona dropdown, or create your own with a custom system prompt, trigger word, classifier, and model.
2. **Load Images** — Click **Select Files** to pick individual images, or **Select Folder** to load an entire directory.
3. **Configure (Optional)** — Edit the system prompt, trigger word, or classifier fields. Upload a `.md`/`.txt` file to set the system prompt from a file.
4. **Start Captioning** — Click **Start Captioning**. Watch progress in real-time via the progress bar and log console.
5. **Download Results** — When processing completes, click **Download Captions ZIP** to save all captions.

### Creating a Persona

1. Edit the system prompt, trigger word, classifier, and/or model fields to your liking.
2. Type a name in the persona name field (e.g., "LoRA Training - Woman").
3. Click **Save Persona** — it's now available in the dropdown for future sessions.

### Model Selection

The default model is `gemini-2.5-flash`. You can change this in the persona settings or the Settings modal to use any Gemini vision model available to your API key.

---

## 🏗️ Build from Source

### Prerequisites

- [Node.js](https://nodejs.org/) (v18+)
- [Rust](https://www.rust-lang.org/tools/install) (for desktop builds only)

### Web Development

```bash
npm install
npm run dev
```

### Desktop (Tauri v2)

```bash
npm install
npm run tauri:dev     # development
npm run tauri:build   # production MSI installer
```

### Web Production Build

```bash
npm run build:web
```

Output goes to `dist-web/`.

---

## 🤖 Building with Agents

A detailed implementation plan is included in [`Captioneer7-Gemini.md`](Captioneer7-Gemini.md) — a comprehensive markdown specification covering every component, data model, API integration detail, and styling rule. This plan was designed for AI coding agents to build the entire application from a single document.

If you're interested in using AI agents for software development, this project serves as a reference for how to structure a build plan that agents can follow end-to-end.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | React 19 + TypeScript |
| Bundler | Vite 7 |
| Desktop | Tauri v2 |
| ZIP Generation | JSZip |
| API | Google Gemini REST API (`generateContent`) |
| Styling | Inline CSS-in-JS |
| State | React hooks + localStorage |

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{captioneer7_gemini,
  title = {Captioneer7-Gemini: Batch Image Captioning with Google Gemini Vision API},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/captioneer7-gemini},
  version = {1.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
