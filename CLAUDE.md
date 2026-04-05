# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file PWA for iPhone travel planning in Bali. The entire app lives in one file: `index.html`.

**Deploy**: Drag `index.html` to Netlify Drop — no build step, no dependencies to install.

## Architecture

- **Single file**: All HTML, CSS, and JS inline in `index.html` — no bundler, no separate files
- **Stack**: Vue 3 (CDN) + Tailwind CSS (CDN) + marked.js (CDN) — all loaded from CDN, nothing installed
- **State**: Vue reactive store in memory; persisted to LocalStorage as JSON
- **AI**: OpenAI-compatible streaming API with function calling (22 tools) — AI has full CRUD access to itinerary data
- **Security**: API keys encrypted with AES-GCM (PBKDF2 master password → key → encrypt all keys before storing in LocalStorage)

## AI Tool System

The AI controls the itinerary exclusively through function calling. Tools are defined as JSON schemas and dispatched to JS functions that mutate the Vue store. Before any mutating tool call, a version snapshot is auto-saved (max 50).

Tool categories: trip CRUD, day CRUD, activity CRUD, route calculation, tips, version history, place search.

## Data Model

Key types (see `PLAN.md` for full TypeScript interfaces):
- `Trip` → has `days[]`, `tips[]`, `notes`
- `Day` → has `date`, `title`, `activities[]`
- `Activity` → has `type` (attraction/restaurant/transport/hotel/other), `time`, `googleInfo`, `xiaohongshu`, `instagram`, `reservation`, `travelFromPrev`

## UI Structure

5 bottom tabs: **AI** (chat + streaming), **Timeline** (day selector + vertical activity cards), **Map** (Google Maps + route polylines), **Browse** (category grid), **More** (settings/export/import/dark mode).

Visual style: Notion-inspired — white/`#F7F6F3` cards, `#37352F` text, 8px radius, Inter/SF Pro.

## AI Providers

5 providers configured (endpoint + model list in `PLAN.md`): OpenAI, Groq, Kimi, Qwen, Gemini. Each provider has a starred default model. Web search is provider-specific (OpenAI uses `web_search_preview` tool, Kimi uses `search` param, Qwen uses `enable_search`, Gemini uses `google_search` tool).

## Workflow Rules

After every set of changes is complete:
1. **Update `PLAN.md`**: Mark completed items with `[x]`, add new items if scope changed.
2. **Commit and push to repo**: Commit all changed files with a descriptive message, then push to `origin main` using the GitHub token from 1Password (`op item get "Github MBA token"`).

Never skip these steps — the repo and plan must always reflect the current state.
