# Bali Travel Planner — Implementation Plan

## Overview
Single-file PWA (`index.html`) for iPhone, mobile-first, Notion-style.
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

### Phase 1 — Core Framework
- [ ] HTML skeleton: PWA meta, CDN imports, Vue 3 app mount
- [ ] Bottom tab navigation + page transitions
- [ ] Settings page: master password + encrypted API key storage
- [ ] Basic chat: streaming AI responses + markdown rendering

### Phase 2 — AI Tool Use
- [ ] Define 22 tools as JSON schema for function calling
- [ ] AI system prompt + tool dispatch
- [ ] Tool execution engine (each tool = JS function that mutates store)
- [ ] Tool call feedback in chat (e.g. "✅ 已添加 Locavore 到 4/7 午餐")
- [ ] Auto save_version before every mutating tool call

### Phase 3 — Timeline UI
- [ ] Horizontal date selector (swipe)
- [ ] Vertical timeline with time axis + activity cards
- [ ] Card expand/collapse animation
- [ ] Travel segment cards between activities
- [ ] LocalStorage persistence

### Phase 4 — Google Maps
- [ ] Maps JS API init + markers by type (colored)
- [ ] Route polylines between activities (numbered ①②③)
- [ ] Bottom drawer: place list
- [ ] Places API: enrich activity after AI adds it
- [ ] Directions API: fill `travelFromPrev`

### Phase 5 — Browse & Settings
- [ ] Browse tab: category pills + card grid
- [ ] Tips sub-tab
- [ ] Export / Import JSON
- [ ] Dark mode toggle
- [ ] Version history viewer

### Phase 6 — Polish
- [ ] All animations (slide, fade, stagger, skeleton)
- [ ] Quick reply buttons in chat
- [ ] Offline-friendly (graceful degradation)
- [ ] PWA "add to home screen" prompt

---

## Key File
`/Users/richard/Project/Bali travel/index.html` — the only file
