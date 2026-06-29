# Desktop agent tooling comparison June 2026

## Introduction

### Manus

[macOS Desktop Agentic Tools for the Office Worker](macos-desktop-agent-comparison.md) A Sophisticated Comparison: Manus, Claude Desktop (Chat · Cowork · Code), and OpenAI Codex
Prepared by Manus AI

[Comparison of Manus, Codex, and Claude Desktop macOS Applications](manus-codex-claude-desktop-macos-comparison.md)
Scope: macOS desktop applications and their architecture, local execution model, sandboxing/isolation, cloud dependency, and security posture.
Primary sources: uploaded reverse-engineering and research notes for Manus, Codex, Claude Desktop, and Claude Code sandboxing.

### ChatGPT

[macOS Desktop Agentic Tools Compared](deep-research-report-3.md)

[macOS Agentic Tools for Office Work](deep-research-report-4.md)

## Decompilation

### Manus.app

[Manus for macOS — technical architecture and reverse-engineering notes](manus-research-by-codex.md) by Codex

[Manus macOS App Technical Overview](manus-research-by-manus.md) by Manus

[Manus Desktop Application - Technical Overview](manus-research-by-claude-haiku.md) by Claude Code Haiku

[Manus Desktop App - Technical Architecture Overview](manus-research-by-claude-sonnet.md) by Claude Code Sonnet

### Codex.app

[Codex.app Technical Overview](codex-research-by-claude.md) by Claude Code Haiku

[Codex.app for macOS: a static technical overview](codex-research-by-codex.md) by Codex


## Cost

All comparisons where made with $20 subscriptions - Manus Pro, ChatGTP Plus, Claude Pro

### Claude Pro

Quota limited. Resets weekly and in every 5 hours

![Claude usage](Claude-usage.png)

### ChatGPT Plus (for Codex)

Quota limited. Resets weekly and in every 5 hours

![Codex usage](Codex-usage.png)

### Manus Pro

Finite monthly credits. Free additional daily credits for lite model usage. Manus is most expensive. Credits spend is similar like API pricing per Millions of tokens.

![Manus usage](Manus-usage.png)

![Manus credits history](Manus-credits-history.png)

Single audio transcription consumes 25% of montly credits!

## Create expense overview from Excel worksheet

PROMPT: Read hobby.xls worksheet "fotokas" and create overview about the expenses. Write new Excel file with diagrams

There was a catch/trick in the excel worksheet. After main rows there was "shopping list" with whichful items not bought. So if agent ignored that then it got wrong total sum. Also in the first row there was =SUM() formula with correct range.

### Claude Cowork

2 minutes. Generated Excel looks nice. Includes pie chart with expence categories. Total sum is right

![Claude excel](fotokas-excel-claude.png)

### Codex

7 minutes. Generated Excel looks fine. Includes column chart "Expenses by category" and "Top 10 single expenses". Total sum is right

![Codex excel](fotokas-excel-codex.png)

### Manus

Credits used 94. Time worked 3m 30s. Generated Excel looks nice. Includes pie chart and column chart with expence categories. Also worksheets by category and all items. But the total sum is incorrect because "shopping list" was also included.

![Manus excel](fotokas-excel-manus.png)

## Identify fighter plane (from image)

Pasted image with resolution 2048 × 1365

![fighter planes](pasted_image_QSm41Z.png)

### Claude

[identify-fighter-plane-claude.md](identify-fighter-plane-claude.md) 

### Codex

[identify-fighter-plane-codex.md](identify-fighter-plane-codex.md) 

### Manus

[identify-fighter-plane-manus.md](identify-fighter-plane-manus.md)

## Image generation from existing photo

PROMPT: Two people sitting on swing. Generate picture where those two are clowns like Piip and Tuut

### Codex

![Codex](clowns-garden-swing-codex.png)

### Manus

![Manus](clowns-garden-swing-manus.png)

### Claude

Claude is not supporting image generations

## Audio transcription

Source: https://www.youtube.com/watch?v=weJlLDyMrIg

Title: Eesti.ai töötuba: Arutelupartnerist töökaaslaseni ehk kuidas tehisaru toetab töötamist

Audio file in mp3 format. Total length 1h 57m

Prompt: Transcribe that Estonian language lecture

Manus is clear winner here. Transcript is high quality and so is the summary.
Codex and Claude rely only on local Whisper transcription model which is comparably slower (on my machine) and quality do not much with Manus.

For comparison see also transcription from Gemini NotebookLM - [gemini-notebooklm-transcription.txt](gemini-notebooklm-transcription.txt)

### Manus

260625_092911_00.mp3
Audio · 107.16 MB
Transcribe that Estonian language lecture
Got it! I will transcribe the Estonian language lecture for you now.

Transcription is progressing well — 20 out of 22 chunks are already done. The lecture is approximately 1 hour 57 minutes long. Almost there!

[estonian_lecture_transcription.txt](estonian_lecture_transcription.txt)

Additional prompt: Tee mulle kokkuvõte jagades loengu olulisemateks teemadeks

[Loengu kokkuvõte: Tehisintellekti praktiline kasutamine ja vaipkoodimine](<Loengu kokkuvõte_ Tehisintellekti praktiline kasutamine ja vaipkoodimine.md>)

Credits used 926
Time worked 22m 26s

### Codex

I’ll take a look at the audio file first: format, duration, and what local transcription tools are already available in the workspace. If there’s a local speech model installed, I’ll use that; otherwise I’ll surface the cleanest next step.

Listed files, ran a commandRan which ffprobe
Listed files in 260625_092911_00.mp3

Good news: this folder already contains transcript-like sidecar files for the MP3 (.txt, .srt, .vtt, .tsv, .json). I’m checking whether those are complete and usable before doing any heavier audio work.

The existing transcript is substantial, but the opening has classic speech-recognition noise/hallucination (“Kõik, kõik…” repeated during likely silence or room noise). I’m going to see which transcriber produced it and whether we can rerun with better Estonian settings or cleanup.

Ran 5 commandsRan which whisper

Ran which faster-whisper

There is an OpenAI Whisper CLI at /opt/homebrew/bin/whisper, but the default python3 on this shell cannot import it, so it’s probably installed under Homebrew’s Python. I’m checking its environment and model options now.

Ran 4 commandsRan find ~/.cache/whisper -maxdepth 1 -type f -print 2>/dev/null
Ran du -sh ~/.cache/whisper 2>/dev/null
Ran head -1 /opt/homebrew/bin/whisper
Ran /opt/homebrew/bin/whisper --help

The local cache has medium, large-v3, and large-v3-turbo models available, so we can rerun without downloading. For a two-hour lecture, I’ll favor large-v3-turbo first: it should be much better than the noisy sidecar and still finish in a reasonable time.

Before starting the full pass, I’m checking whether PyTorch can use Apple Silicon acceleration. That determines whether large-v3 is sensible or whether large-v3-turbo is the practical choice.

#### user notes

So basically Codex found whisper in my computer and tried to use that for transcription without any existing skills

[Whisper transcription in srt format](260625_092911_00.txt)

Whisper is using local models. This actual transcription was generated with "large-v3-turbo" model. Transcrition takes several hours because I do not have GPU

### Claude Cowork / Code

You can get skill for Claude from Github - https://github.com/spillwavesolutions/whisper-transcribe

Without the skill Claude will not do anything with mp3 audio files.

Claude does not support audio transcription in the server side.

## Skills

All the 3 apps are supporting skills

Testing the same Supadata skill for generating summarys from Youtube transcriptions

### PROMPT

/supadata fetch native transcript https://www.youtube.com/watch?v=H-SgQP3Hif0
Tee eesti keelne kokkuvõte

### Manus

[supadata-transcript-manus.md](supadata-transcript-manus.md)

### Codex

[supadata-transcript-codex.md](supadata-transcript-codex.md)

### Claude chat

[supadata-transcript-claude.md](supadata-transcript-claude.md)

## Deep research / Wide research

[ChatGPT Deep Research](https://en.wikipedia.org/wiki/ChatGPT_Deep_Research)

From Wikipedia, the free encyclopedia

Deep Research is an AI agent integrated into ChatGPT, which generates cited reports on a user-specified topic by autonomously browsing the web for 5 to 30 minutes.

Release: February 3, 2025

### PROMPT

My company is using Microsoft M365 for e-mail, calendars, Teams and Sharepoint for files. If we change to the Google Workspace what are the major changes? What we might miss? We have E5 license (1200 users) and we choose Enterprise license for Workspace

### ChatGPT Deep Research

Codex does not directly support this feature but since the subscruption allow to use https://chatgpt.com/ web usage also then we tried that

[Migration from Microsoft 365 E5 to Google Workspace Enterprise](deep-research-report-1.md)

### Manus Wide Research

Since Manus desktop app is basically frontend of https://manus.im/app then the functionality is basically the same

[Migration from Microsoft 365 E5 to Google Workspace Enterprise](manus-wide-research-1.md)

### Claude chat

Since the Claude does not have direct comparabe method I applied sentence "Deep Research" to the prompt.

[Migration from Microsoft 365 E5 to Google Workspace Enterprise](claude-deep-research-1.md)

## Artifact / Web page from research

After doing research with chat you can produce Web page as artifact for more interactive view

### PROMPT

Compare two Canon cameras 
https://www.the-digital-picture.com/Reviews/Canon-EOS-R5.aspx
https://www.the-digital-picture.com/Reviews/Canon-EOS-R5-Mark-II.aspx

### Claude chat

[Canon EOS R5 vs. Canon EOS R5 Mark II — Comprehensive Comparison](claude-deep-research-2.md)

Additional prompt: Create fancy Claude Artifact with illustration Find and add photos and illustrations about the actual products and memory cards they support and batteries and grips etc.

![alt text](1afce8ee-a8d4-4169-a02a-ee5fa0ea32f6.png)

### ChatGPT Custom - HTML + CSS + Javascript

[Canon EOS R5 versus Canon EOS R5 Mark II](deep-research-report-2.md)

Additional prompt: Find and add photos and illustrations about the actual products and memory cards they support and batteries and grips etc.

![ChatGTP Web page](d9bd72a1-4b2b-4c53-b9c5-a15a9116faae.png)

### Manus

Most expensive result page from "Generate a webpage from the "manus-wide-research-2.md" file

Credits used 526
Time worked 22m 22s

[Canon EOS R5 vs. Canon EOS R5 Mark II — Comprehensive Comparison](manus-wide-research-2.md)

Claude Code helped to explain what it is

PROMPT: Analyze canon-comparison directory and describe what it is

**`canon-comparison`** is a full-stack web application that serves as a detailed **product comparison page for the Canon EOS R5 vs R5 Mark II cameras**.

**Tech stack:**
- **Frontend:** React 19 + TypeScript + Vite, Tailwind CSS v4, Radix UI component library (shadcn/ui style), Framer Motion for animations, Recharts for data visualization
- **Backend:** Express.js server (`server/index.ts`), bundled with esbuild
- **Routing:** Wouter (lightweight React router)

**What it does:**
The app is a single-page editorial-style comparison site with these sections (each a dedicated component):
- `NavBar` — sticky navigation with camera quick-jump links
- `Hero` — full-bleed side-by-side camera silhouettes
- `SpecComparison` — asymmetric two-column spec table with red "winner" badges per row
- `PerformanceSection` — likely charts/benchmarks (Recharts)
- `UseCaseSection` — use-case grid with icon badges
- `AccessoriesSection` — horizontal scrollable accessory gallery
- `PricingSection` — pricing breakdown
- `ExecutiveSummary` — key takeaways

**Design intent** (documented in `ideas.md`): "Editorial Contrast" — Swiss typographic style on a warm off-white background with Canon Red (`#CC0000`) as the signature accent. Serif display fonts (Playfair Display), body copy (Source Serif 4), and DM Sans for UI labels.

It also includes a `ManusDialog` component and `vite-plugin-manus-runtime` in devDeps, suggesting it was **generated or scaffolded by the Manus AI agent platform** as a demo/showcase project.

![Manus Web page](005e3e3c-b24e-4901-8387-6bfc7ff255b8.png)

