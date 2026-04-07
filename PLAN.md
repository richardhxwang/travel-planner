# Travel Planner — Implementation Plan

## Overview
Single-file PWA (`index.html`) for iPhone, mobile-first, Notion-style.
Generic travel planner — supports any destination, up to 3 concurrent trips.
Deploy via Netlify Drop (drag & drop). API keys encrypted in LocalStorage.

---

## Architecture

- **Stack**: Vue 3 (CDN) + Tailwind CSS (CDN) + marked.js (CDN)
- **Single file**: All CSS/JS inline in `index.html`
- **PWA**: Inline manifest JSON, no service worker needed
- **AI**: OpenAI-compatible streaming, 5 providers (OpenAI/Groq/Kimi/Qwen/Gemini)
- **AI tool use**: Function calling — AI has full CRUD access to itinerary data
- **Security**: AES-GCM encrypted API keys, PBKDF2 master password

---

## AI Tools (22 total — AI controls itinerary via function calling)

### Trip-level
- `get_itinerary` — read full trip data (AI context)
- `create_trip` — create/initialize trip
- `update_trip` — update name, destination, dates, notes

### Day-level
- `add_day` — add a new day
- `update_day` — update day title/date
- `delete_day` — remove entire day
- `duplicate_day` — copy a day to another date
- `reorder_day` — reorder activities within a day

### Activity-level
- `add_activity` — add activity to a day
- `update_activity` — modify any activity fields
- `delete_activity` — remove activity
- `move_activity` — move activity between days or reorder
- `batch_update` — execute multiple operations atomically

### Route/Navigation
- `calculate_route` — calc travel time between two activities
- `update_all_routes` — recalculate all routes for a day

### Tips
- `add_tip` — add a travel tip
- `delete_tip` — remove a tip

### Version history
- `save_version` — snapshot current itinerary with label
- `restore_version` — revert to a past version
- `compare_versions` — show diff between two versions
- `get_versions` — list all saved versions

### Search
- `search_place` — search Google Places for a location

---

## Data Model

```typescript
interface Trip {
  id: string
  name: string           // "巴厘岛之旅"
  destination: string    // "Bali, Indonesia"
  startDate: string      // "2026-04-05"
  endDate: string
  days: Day[]
  tips: Tip[]
  notes: string
  createdAt: number
  updatedAt: number
}

interface Day {
  id: string
  date: string
  title: string          // "乌布文化日"
  activities: Activity[]
}

interface Activity {
  id: string
  time: string           // "09:00"
  endTime: string        // "11:00"
  type: 'attraction' | 'restaurant' | 'transport' | 'hotel' | 'other'
  name: string
  placeId?: string
  location?: { lat: number, lng: number }
  address?: string
  notes: string
  googleInfo?: {
    rating: number
    reviewCount: number
    openNow?: boolean
    hours?: string
    phone?: string
    website?: string
    priceLevel?: number
  }
  xiaohongshu?: {
    summary: string
    highlights: string[]
    posts: Array<{ title: string; url: string }>
    searchUrl: string
  }
  instagram?: {
    summary: string
    hashtags: string[]
    posts: Array<{ url: string; description: string }>
    searchUrl: string
  }
  reservation: {
    required: boolean
    urgency?: string
    links: Array<{ platform: string; url: string }>
    tip?: string
  }
  travelFromPrev?: {
    duration: string
    distance: string
    mode: 'driving' | 'walking' | 'transit'
  }
}

interface Tip {
  id: string
  category: string       // "交通" | "货币" | "文化" | "天气"
  content: string
}

interface ChatMessage {
  role: 'user' | 'assistant'
  content: string
  timestamp: number
  toolCalls?: any[]
}

interface Version {
  id: string
  label: string          // "规划乌布第一天"
  timestamp: number
  snapshot: Trip
}
```

---

## UI Structure

### Bottom Tab Bar (5 tabs)
1. **🤖 AI** — Chat interface, streaming responses, tool call feedback
2. **📅 Timeline** — Day selector + vertical timeline with activity cards
3. **🗺️ Map** — Full-screen Google Maps, colored markers, route polylines
4. **🍜 Browse** — Categorized view: Food / Attractions / Hotels / Shopping / Tips
5. **⋯ More** — Action sheet → Settings, Export, Import, Dark mode

### Visual Style (Notion-inspired)
- Background: `#FFFFFF` / dark: `#191919`
- Card: `#F7F6F3` / dark: `#202020`
- Text: `#37352F` / dark: `#E3E2DE`
- Border radius: cards 8px, buttons 6px
- Font: Inter / SF Pro system font, 15px body
- Icons: emoji for type indicators

### Activity Card (Timeline, collapsed)
```
┌────────────────────────────┐
│ 09:00  🏛️  德格拉朗梯田     │
│ ⭐ 4.6 (1.2k) · ⏱ 1.5h    │
│ 📕 "早上去光线最好" · ✅无需预约│
└────────────────────────────┘
```

### Activity Card (expanded, tap to open)
- Full details: address, hours, phone
- AI notes
- 小红书: summary + 2-3 clickable post links
- Instagram: summary + hashtags + post links
- Reservation: required/not + booking links
- Action buttons: Navigate / Call / Website

---

## AI Providers

| Provider | Endpoint | Models | Web Search |
|----------|----------|--------|------------|
| OpenAI | api.openai.com/v1 | gpt-5-nano⭐, gpt-5-mini, gpt-4o-mini, gpt-5, gpt-5.2, gpt-4.1 | web_search_preview tool |
| Groq | api.groq.com/openai/v1 | llama-3.3-70b-versatile⭐, llama-4-scout-17b-16e-instruct, qwen-qwq-32b | ❌ |
| Kimi | api.moonshot.cn/v1 | kimi-k2.5⭐, kimi-k2-thinking, moonshot-v1-128k | search param |
| Qwen | dashscope.aliyuncs.com/... | qwen3.5-plus⭐, qwen3-max, qwen3-plus | enable_search: true |
| Gemini | generativelanguage.googleapis.com/... | gemini-3-flash⭐, gemini-2.5-flash, gemini-3.1-pro-preview, gemini-2.5-pro | google_search tool |

---

## Security
1. User sets **master password** on first launch
2. Master password → PBKDF2 → AES-GCM key
3. All API keys encrypted in LocalStorage as ciphertext
4. On open: enter master password → decrypt to memory → cleared on page close
5. Google Maps API key also encrypted same way

---

## Version History
- Every AI modification auto-saves a version snapshot before applying changes
- User can view diff: "你改了什么" → AI compares versions
- User can say "还原到上一个版本" → AI calls `restore_version`
- Max 50 versions stored

---

## Implementation Phases

### Phase 1 — Core Framework ✅
- [x] HTML skeleton: PWA meta, CDN imports, Vue 3 app mount
- [x] Bottom tab navigation + page transitions
- [x] Settings page: master password + encrypted API key storage (AES-GCM + PBKDF2)
- [x] Basic chat: streaming AI responses + markdown rendering
- [x] Lock/unlock screen with master password

### Phase 2 — AI Tool Use ✅
- [x] Define 22 tools as JSON schema for function calling
- [x] AI system prompt with xiaohongshu/instagram/reservation instructions
- [x] Tool execution engine (each tool = JS function that mutates store)
- [x] Tool call feedback in chat (collapsible chips with arg summary)
- [x] Auto save_version before every mutating tool call
- [x] Multi-turn tool call loop (up to 8 depth)
- [x] 5 provider support: OpenAI, Groq, Kimi, Qwen, Gemini
- [x] Web search toggle (per-provider: tool or param)

### Phase 3 — Timeline UI ✅
- [x] Horizontal date selector (swipe)
- [x] Vertical timeline with time axis + activity cards
- [x] Card expand/collapse with chevron animation
- [x] Travel segment cards between activities (mode icon + duration)
- [x] LocalStorage persistence (encrypted)
- [x] Long-press context menu (edit/delete)
- [x] Add day / add activity modals

### Phase 4 — Google Maps ✅
- [x] Maps JS API dynamic init with encrypted key
- [x] Colored numbered markers by type (①②③)
- [x] Route polylines between activities
- [x] Day selector chips in map header
- [x] Placeholder when no API key configured

### Phase 5 — Browse & Settings ✅
- [x] Browse tab: category pills + card grid + search
- [x] Tips sub-tab with category badges
- [x] Export / Import JSON
- [x] Dark mode toggle (persisted)
- [x] Version history viewer with restore
- [x] All 5 provider API key management
- [x] Provider/model selection

### Phase 6 — Polish ✅
- [x] Slide/fade tab transitions
- [x] Quick reply chips in chat (context-aware)
- [x] Toast notifications
- [x] IME-safe Chinese input (composing check)
- [x] buildMsgs fix: tool call history preserved across turns
- [x] PWA "add to home screen" prompt (beforeinstallprompt + iOS Safari detection)
- [x] Offline-friendly (graceful degradation: offline banner, AI guard, online/offline events)

### Phase 7 — UI Redesign ✅
Design reference: LumiChat (color system) + FurNote (tab bar)

- [x] **LumiChat color tokens**: CSS custom properties — accent `#10a37f`, dark bg `#000`/`#1a1a1a`, light bg `#fff`/`#f4f4f5`, borders `#2e2e2e`/`#d9d9d9`
- [x] **SF Symbols–style SVG icons**: 25+ inline SVGs replacing ALL emoji (stroke-based, 1.5px, round caps)
- [x] **FurNote-style tab bar**: backdrop blur, scale animation on active, tint color highlight
- [x] **LumiChat input bar**: separated circular send/stop buttons with shadow, backdrop blur bar
- [x] **User message bubbles**: right-aligned rounded pill with accent green bg
- [x] **Map detail sheet**: tap marker → slide-up bottom sheet with full activity details (google/xiaohongshu/instagram/reservation)
- [x] **Activity detail in expanded cards**: xiaohongshu highlights + posts, instagram hashtags + posts, reservation links (already implemented in Phase 3)

### Phase 8 — Generalization & Multi-trip ✅
- [x] Remove all Bali-specific hardcoding (titles, prompts, defaults, localStorage keys)
- [x] Geolocation: user position blue dot, smart map center (trip → user loc → world)
- [x] Multi-trip: up to 3 concurrent plans, switcher UI in settings, per-trip isolation
- [x] AI session memory: encrypted chat history per trip, restored on unlock/switch
- [x] Legacy single-trip data auto-migration

### Phase 9 — E2E Testing & Bug Fixes ✅
- [x] AI icon: sparkles SVG (replaced scary robot)
- [x] Map persistence: v-show outside transition (no DOM destruction on tab switch)
- [x] Google Directions API: real road routes (fallback to straight lines)
- [x] Input bar: flat single-layer design (removed inner border)
- [x] Browse cards: clickable expand with full details (Google/XHS/IG/reservation/navigate)
- [x] All emoji → SVG icons (eye, eyeOff, lightbulb, clock, globe, etc.)
- [x] Fake link removal: no fabricated post URLs, only real search buttons (小红书搜索/Instagram搜索)
- [x] Friendly error messages: 429→服务繁忙, 401→Key无效, network errors in Chinese
- [x] Retry button on error messages
- [x] Chat memory fix: show all AI messages including tool-call turns (_skip only for API)
- [x] System prompt: explicit "no fake URLs" instruction
- [x] Navigate button: "在 Google Maps 中导航" in browse/map detail cards
- [x] **Google Places API integration**: search_place tool now calls real Places Text Search + Details API
- [x] Real data: website, phone, rating, reviews, hours, priceLevel, address, GPS all from Google
- [x] XHS/IG posts stripped at tool layer (LLMs always fabricate URLs), platform search buttons only
- [x] Tool call depth limit raised to 15 (web search uses many turns)
- [x] System prompt rewritten for efficient workflow: search_place → create_trip → add_day → add_activity

---

## Deployment
- **Repo**: https://github.com/richardhxwang/travel-planner
- **GitHub Pages**: https://richardhxwang.github.io/travel-planner/
- **Local**: `python3 -m http.server 8899` → http://localhost:8899
- **Netlify**: Drag `index.html` to Netlify Drop

## Key File
`index.html` — the only file
