# Desktop agent tooling comparison June 2026

## Executive overview

This research compares Manus, OpenAI Codex, and Claude Desktop as practical desktop agents for office-style work on macOS. The tests mix technical inspection with hands-on user tasks: spreadsheet analysis, image understanding, image generation, audio transcription, reusable skills, deep research, and web-page artifact generation. The main pattern is that each tool has different strengths: Codex is strongest when it can use local files, shell tools, and existing developer workflows; Manus is effective for hosted agent tasks and produced the best Estonian lecture transcription; Claude is polished for chat and document-style output but depends more on product mode and external skills for some desktop tasks.

The comparison also highlights cost and quota differences. All tests used roughly comparable $20 consumer/pro subscriptions where possible, but Manus charges finite credits per task, while ChatGPT Plus and Claude Pro are quota-limited by usage windows. That makes apparently similar tasks feel very different in practice, especially for long-running work such as transcription or generated web artifacts.

## Introduction

### Manus

[macOS Desktop Agentic Tools for the Office Worker: A Sophisticated Comparison of Manus, Claude Desktop (Chat, Cowork, Code), and OpenAI Codex](macos-desktop-agent-comparison.md)

Prepared by Manus AI.

[Comparison of Manus, Codex, and Claude Desktop macOS Applications](manus-codex-claude-desktop-macos-comparison.md)
Scope: macOS desktop applications and their architecture, local execution model, sandboxing/isolation, cloud dependency, and security posture.
Primary sources: uploaded reverse-engineering and research notes for Manus, Codex, Claude Desktop, and Claude Code sandboxing.

### Comparison with user tests

The Manus-authored comparison presents Manus as the most accessible office-worker tool: it has native document-style output, strong multimedia handling, and a polished non-technical user experience. The hands-on tests mostly confirm that, but add an important practical correction: Manus can be excellent, especially for audio transcription, wide research, image work, and beautiful generated webpages, but it is painfully expensive in credits. Its generated webpages can also be tied to the Manus hosting/runtime model; using them outside the `manus.space` domain may require a special web server setup. Manus can read and generate images, using Nano Banana 2 from Google Gemini for image generation.

Codex looks less polished in the report for non-technical office workers, but the user tests show why it is still very strong: it works directly with local files, keeps generated results in `Documents/Codex/`, and avoids accidental overwrites by working in its own output area. Word, Excel, and PowerPoint skills are built in. Codex can read images and generate images through ChatGPT image generation models, and it is especially good when the task benefits from local files, shell tools, repeatable scripts, and transparent artifacts.

Claude is the most sophisticated agent tool overall, especially in how it reasons, structures work, and can be guided by careful prompting. The report's privacy and local-execution strengths for Claude Cowork still matter, but the user tests show clearer capability limits: Claude reads images well, including OCR-style extraction, but does not generate or edit images. It also has no native audio transcription support; audio work requires an added skill, local model such as Whisper, remote API, or MCP provider. Claude does not have a dedicated Wide Research or Deep Research mode here, but it can be prompted toward deeper research behavior with enough context and guidance.

### Overview by ChatGPT

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

### Claude.app

[Claude.app Architecture Diagram](../research-claude-desktop-architecture.md) 
[Claude.app Research Notes](../research-claude-desktop-virtualization.md)
[Claude Code Sandbox Architecture](../claude-code-sandbox.md)
[Claude Code: Tracking User Frustration](../claude-code-tracking-swearing.md)

## Cost

This section compares the practical cost model of each product during the tests. All comparisons were made with $20 subscriptions: Manus Pro, ChatGPT Plus, and Claude Pro.

### Claude Pro

Quota-limited. Usage resets weekly and every five hours.

![Claude usage](Claude-usage.png)

### ChatGPT Plus (for Codex)

Quota-limited. Usage resets weekly and every five hours.

![Codex usage](Codex-usage.png)

### Manus Pro

Finite monthly credits. Manus also provides additional free daily credits for lite-model usage. Manus is the most expensive option in these tests. Credit spending feels similar to API pricing per million tokens.

![Manus usage](Manus-usage.png)

![Manus credits history](Manus-credits-history.png)

A single audio transcription consumed about 25% of the monthly credits.

## Create expense overview from Excel worksheet

This test checks whether the agent can inspect a real spreadsheet, understand which rows belong in the analysis, avoid a hidden trap, and produce a new Excel workbook with useful charts.

PROMPT: Read hobby.xls worksheet "fotokas" and create overview about the expenses. Write new Excel file with diagrams.

There was a catch in the Excel worksheet. After the main rows, there was a "shopping list" with wishlist items that had not been bought. If the agent ignored that boundary, it got the wrong total sum. The first row also contained a `=SUM()` formula with the correct range.

### Claude Cowork

2 minutes. Generated Excel looks nice. Includes a pie chart with expense categories. Total sum is correct.

![Claude excel](fotokas-excel-claude.png)

### Codex

7 minutes. Generated Excel looks fine. Includes column charts for "Expenses by category" and "Top 10 single expenses". Total sum is correct.

![Codex excel](fotokas-excel-codex.png)

### Manus

Credits used: 94. Time worked: 3m 30s. Generated Excel looks nice. Includes a pie chart and a column chart with expense categories. Also includes worksheets by category and all items. However, the total sum is incorrect because the "shopping list" rows were included.

![Manus excel](fotokas-excel-manus.png)

## Identify fighter plane (from image)

This test checks visual identification and reasoning from an image. The agent had to look at a photo of aircraft, infer the likely fighter model, and explain the evidence rather than just describe the image.

Pasted image with resolution 2048 x 1365.

![fighter planes](pasted_image_QSm41Z.png)

### Claude

[identify-fighter-plane-claude.md](identify-fighter-plane-claude.md)

### Codex

[identify-fighter-plane-codex.md](identify-fighter-plane-codex.md)

### Manus

[identify-fighter-plane-manus.md](identify-fighter-plane-manus.md)

## Image generation from existing photo

This test checks whether the agent can transform an existing photo into a new generated image while preserving the basic scene and following a culturally specific style prompt.

PROMPT: Two people sitting on swing. Generate picture where those two are clowns like Piip and Tuut.

### Codex

![Codex](clowns-garden-swing-codex.png)

### Manus

![Manus](clowns-garden-swing-manus.png)

### Claude

Claude does not support image generation in this setup.

## Audio transcription

This test checks long-form audio transcription for Estonian speech. It compares whether the agent can handle a nearly two-hour lecture, what transcription backend it uses, how long the work takes, and whether the resulting text is good enough for summarization.

Source: [YouTube lecture](https://www.youtube.com/watch?v=weJlLDyMrIg)

Title: Eesti.ai töötuba: Arutelupartnerist töökaaslaseni ehk kuidas tehisaru toetab töötamist

Audio file in MP3 format. Total length: 1h 57m.

Prompt: Transcribe that Estonian language lecture.

Manus is the clear winner here. The transcript is high quality, and so is the summary.
Codex and Claude rely on local Whisper transcription, which is much slower on my machine, and the quality does not match Manus.

For comparison, see also the transcription from Gemini NotebookLM: [gemini-notebooklm-transcription.txt](gemini-notebooklm-transcription.txt).

### Manus

`260625_092911_00.mp3`
Audio · 107.16 MB
Transcribe that Estonian language lecture
Got it! I will transcribe the Estonian language lecture for you now.

Transcription is progressing well — 20 out of 22 chunks are already done. The lecture is approximately 1 hour 57 minutes long. Almost there!

[estonian_lecture_transcription.txt](estonian_lecture_transcription.txt)

Additional prompt: Tee mulle kokkuvõte jagades loengu olulisemateks teemadeks

[Loengu kokkuvõte: Tehisintellekti praktiline kasutamine ja vaipkoodimine](<Loengu kokkuvõte_ Tehisintellekti praktiline kasutamine ja vaipkoodimine.md>)

Credits used: 926.
Time worked: 22m 26s.

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

[Whisper transcription in text format](260625_092911_00.txt)

Whisper uses local models. This transcription was generated with the `large-v3-turbo` model. Transcription takes several hours because I do not have a GPU.

### Claude Cowork / Code

You can get a skill for Claude from GitHub: [spillwavesolutions/whisper-transcribe](https://github.com/spillwavesolutions/whisper-transcribe).

Without the skill, Claude will not do anything with MP3 audio files.

Claude does not support server-side audio transcription in this setup.

## Skills

This test checks skill/plugin support across the three apps by using the same Supadata skill to fetch a YouTube transcript and generate an Estonian summary.

All three apps support skills.

Testing the same Supadata skill for generating summaries from YouTube transcriptions.

### PROMPT

/supadata fetch native transcript [https://www.youtube.com/watch?v=H-SgQP3Hif0](https://www.youtube.com/watch?v=H-SgQP3Hif0)
Tee eesti keelne kokkuvõte

### Manus

[supadata-transcript-manus.md](supadata-transcript-manus.md)

### Codex

[supadata-transcript-codex.md](supadata-transcript-codex.md)

### Claude chat

[supadata-transcript-claude.md](supadata-transcript-claude.md)

## Deep research / Wide research

This test compares long-form research modes. ChatGPT has Deep Research, Manus has Wide Research, and Claude was tested with a prompt that asked it to perform "Deep Research" even though it does not expose an equivalent dedicated mode here.

[ChatGPT Deep Research](https://en.wikipedia.org/wiki/ChatGPT_Deep_Research)

From Wikipedia, the free encyclopedia

Deep Research is an AI agent integrated into ChatGPT, which generates cited reports on a user-specified topic by autonomously browsing the web for 5 to 30 minutes.

Release: February 3, 2025

### PROMPT

My company is using Microsoft 365 for email, calendars, Teams, and SharePoint for files. If we change to Google Workspace, what are the major changes? What might we miss? We have E5 licenses for 1,200 users, and we would choose the Enterprise license for Workspace.

### ChatGPT Deep Research

Codex does not directly support this feature, but since the subscription also allows use of [chatgpt.com](https://chatgpt.com/), we tried that.

[Migration from Microsoft 365 E5 to Google Workspace Enterprise](deep-research-report-1.md)

### Manus Wide Research

Since the Manus desktop app is basically a frontend for [manus.im/app](https://manus.im/app), the functionality is basically the same.

[Migration from Microsoft 365 E5 to Google Workspace Enterprise](manus-wide-research-1.md)

### Claude chat

Since Claude does not have a directly comparable method, I added the phrase "Deep Research" to the prompt.

[Migration from Microsoft 365 E5 to Google Workspace Enterprise](claude-deep-research-1.md)

## Artifact / web page from research

This test checks whether research output can be turned into a usable, visual web artifact. The comparison used a Canon EOS R5 versus EOS R5 Mark II research task, then asked each tool to create a richer interactive or artifact-style page.

### PROMPT

Compare two Canon cameras:

[Canon EOS R5 review](https://www.the-digital-picture.com/Reviews/Canon-EOS-R5.aspx)

[Canon EOS R5 Mark II review](https://www.the-digital-picture.com/Reviews/Canon-EOS-R5-Mark-II.aspx)

### Claude chat

[Canon EOS R5 vs. Canon EOS R5 Mark II — Comprehensive Comparison](claude-deep-research-2.md)

Additional prompt: Create fancy Claude Artifact with illustration. Find and add photos and illustrations about the actual products, memory cards they support, batteries, grips, etc.

![Claude artifact page](1afce8ee-a8d4-4169-a02a-ee5fa0ea32f6.png)

### ChatGPT custom - HTML + CSS + JavaScript

[Canon EOS R5 versus Canon EOS R5 Mark II](deep-research-report-2.md)

Additional prompt: Find and add photos and illustrations about the actual products, memory cards they support, batteries, grips, etc.

![ChatGPT web page](d9bd72a1-4b2b-4c53-b9c5-a15a9116faae.png)

### Manus

Most expensive result page from: "Generate a webpage from the `manus-wide-research-2.md` file."

Credits used: 526.
Time worked: 22m 22s.

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

![Manus web page](005e3e3c-b24e-4901-8387-6bfc7ff255b8.png)
