# Nourish Riot — Project Blueprint

---

## 1. Project Overview

Nourish Riot is a single-file personal recipe app for home cooks with an aesthetic that blends yoga grounding with punk-rock rawness — dark backgrounds, electric acid-green and hot-pink accents, torn-edge card graphics, and zine-style typography. It lives in one `index.html` file with no build step: vanilla JS, vanilla CSS, Supabase (Postgres + Storage + Realtime), and the Anthropic Claude API handle everything. Users can capture recipes four ways (photo scan, voice dictation, pasted text, or manual form), browse a filterable/searchable card grid, view full recipe details, and share recipes as plain text. All devices stay in sync via Supabase Realtime subscriptions.

---

## 2. File Structure

Everything lives in a single `index.html`. Internal organization:

```
index.html
│
├── <head>
│   ├── <meta charset="UTF-8">
│   ├── <meta name="viewport" content="width=device-width, initial-scale=1">
│   ├── <title>Nourish Riot</title>
│   ├── Google Fonts link (Black Ops One, Barlow Condensed 300/400/600/700, Noto Serif italic)
│   ├── Supabase CDN: https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js
│   └── <style> — ALL CSS (see Section 4)
│
└── <body>
    │
    ├── <!-- GRAIN OVERLAY -->
    │   <div class="grain"></div>
    │
    ├── <!-- HEADER -->
    │   <header>
    │     <div class="logo">NOURISH<span>RIOT</span></div>
    │     <div class="header-right">
    │       <div class="sync-dot" id="syncDot"></div>
    │       <div class="add-wrapper">
    │         <button class="add-btn" id="addBtn">+ Add Recipe</button>
    │         <div class="add-dropdown" id="addDropdown">
    │           <button data-action="photo">📷 Upload / Scan</button>
    │           <button data-action="voice">🎙 Voice Dictate</button>
    │           <button data-action="text">📝 Paste Text</button>
    │           <button data-action="manual">✏️ Type Manually</button>
    │         </div>
    │       </div>
    │     </div>
    │   </header>
    │
    ├── <!-- TOOLBAR -->
    │   <div class="toolbar">
    │     <input class="search-input" id="searchInput" placeholder="Search recipes…" />
    │     <div class="filter-pills" id="filterPills">
    │       <!-- rendered by renderPills() -->
    │     </div>
    │   </div>
    │
    ├── <!-- MAIN GRID -->
    │   <main>
    │     <div class="section-header">
    │       <h2 id="sectionTitle">All Recipes</h2>
    │       <span class="recipe-count" id="recipeCount"></span>
    │       <select class="sort-select" id="sortSelect">
    │         <option value="date">Newest First</option>
    │         <option value="name">Name A–Z</option>
    │       </select>
    │     </div>
    │     <div class="recipe-grid" id="recipeGrid">
    │       <!-- rendered by renderGrid() -->
    │     </div>
    │   </main>
    │
    ├── <!-- TOAST -->
    │   <div class="toast" id="toast"></div>
    │
    ├── <!-- VIEW MODAL -->
    │   <div class="modal-overlay" id="viewModal">
    │     <div class="modal view-modal">
    │       <button class="modal-close" id="viewClose">✕</button>
    │       <div id="viewContent"></div>
    │       <div class="view-actions">
    │         <button id="editBtn">Edit</button>
    │         <button id="shareBtn">Share</button>
    │         <button id="deleteBtn">Delete</button>
    │       </div>
    │     </div>
    │   </div>
    │
    ├── <!-- FORM MODAL -->
    │   <div class="modal-overlay" id="formModal">
    │     <div class="modal form-modal">
    │       <button class="modal-close" id="formClose">✕</button>
    │       <h2 id="formTitle">Add Recipe</h2>
    │       <form id="recipeForm">
    │         <!-- see Section 6 for full field list -->
    │       </form>
    │     </div>
    │   </div>
    │
    ├── <!-- CAPTURE MODAL -->
    │   <div class="modal-overlay" id="captureModal">
    │     <div class="modal capture-modal">
    │       <button class="modal-close" id="captureClose">✕</button>
    │       <div class="capture-tabs">
    │         <button class="tab-btn active" data-tab="photo">Photo</button>
    │         <button class="tab-btn" data-tab="voice">Voice</button>
    │         <button class="tab-btn" data-tab="text">Text</button>
    │       </div>
    │       <div id="captureBody">
    │         <!-- tab content rendered by switchCaptureTab() -->
    │       </div>
    │     </div>
    │   </div>
    │
    └── <script>
        ├── // ── CONFIG
        ├── // ── STATE
        ├── // ── SUPABASE HELPERS
        ├── // ── PHOTO HELPERS
        ├── // ── RENDER
        ├── // ── VIEW MODAL
        ├── // ── FORM MODAL
        ├── // ── CAPTURE MODAL
        ├── // ── ANTHROPIC AI
        └── // ── INIT
```

---

## 3. Database Setup

Run the following SQL in the Supabase SQL editor **before** writing any app code.

### 3.1 Create `recipes` table

```sql
create table recipes (
  id            text primary key,
  name          text not null,
  category      text,
  desc          text,
  prep          text,
  cook          text,
  servings      integer,
  ingredients   jsonb,
  steps         jsonb,
  notes         text,
  inspired_by   text,
  photo_url     text,
  date_added    date,
  date_modified date,
  created_at    timestamptz default now()
);
```

### 3.2 Enable Realtime

```sql
alter publication supabase_realtime add table recipes;
```

### 3.3 Row Level Security (open policy — no auth)

```sql
alter table recipes enable row level security;

create policy "allow all"
  on recipes for all
  using (true)
  with check (true);
```

### 3.4 Storage bucket

```sql
insert into storage.buckets (id, name, public)
values ('recipe-photos', 'recipe-photos', true);

create policy "public read"
  on storage.objects for select
  using (bucket_id = 'recipe-photos');

create policy "public upload"
  on storage.objects for insert
  with check (bucket_id = 'recipe-photos');

create policy "public delete"
  on storage.objects for delete
  using (bucket_id = 'recipe-photos');
```

> **Verify:** After running, confirm the `recipe-photos` bucket appears in Supabase Storage and has Public access enabled in the dashboard.

---

## 4. CSS Architecture

### 4.1 Custom Properties (Design Tokens)

Declare on `:root`:

```css
:root {
  --void:   #0d0d0d;
  --ash:    #1a1a1a;
  --smoke:  #2b2b2b;
  --gravel: #3d3d3d;
  --fog:    #888888;
  --bone:   #e8e2d9;
  --acid:   #c8f000;
  --plasma: #ff2d78;
  --zen:    #7ecec4;

  --font-head: 'Black Ops One', cursive;
  --font-ui:   'Barlow Condensed', sans-serif;
  --font-body: 'Noto Serif', serif;

  --radius: 4px;
  --card-w: 280px;
}
```

### 4.2 Global Reset & Base

```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
html { font-size: 16px; }
body {
  background: var(--void);
  color: var(--bone);
  font-family: var(--font-ui);
  min-height: 100vh;
  overflow-x: hidden;
}
a { color: inherit; text-decoration: none; }
button { cursor: pointer; border: none; background: none; font-family: inherit; }
input, select, textarea {
  font-family: var(--font-ui);
  background: var(--smoke);
  color: var(--bone);
  border: 1px solid var(--gravel);
  border-radius: var(--radius);
}
```

### 4.3 Grain Texture Overlay

```css
.grain {
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: 9999;
  opacity: 0.04;
  mix-blend-mode: overlay;
  background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 256 256' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noise'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noise)'/%3E%3C/svg%3E");
  background-size: 256px 256px;
}
```

### 4.4 Header

```css
header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 24px;
  height: 64px;
  background: var(--ash);
  border-bottom: 3px solid var(--acid);
  position: sticky;
  top: 0;
  z-index: 100;
  /* skewed plasma stripe */
  overflow: hidden;
}
header::after {
  content: '';
  position: absolute;
  top: 0; right: -20px;
  width: 120px; height: 100%;
  background: var(--plasma);
  opacity: 0.12;
  transform: skewX(-18deg);
  pointer-events: none;
}

.logo {
  font-family: var(--font-head);
  font-size: 1.6rem;
  letter-spacing: 0.04em;
  color: var(--bone);
}
.logo span { color: var(--acid); }

.header-right {
  display: flex;
  align-items: center;
  gap: 14px;
}
```

### 4.5 Sync Dot

```css
.sync-dot {
  width: 10px; height: 10px;
  border-radius: 50%;
  background: var(--acid);
  box-shadow: 0 0 8px var(--acid);
  transition: background 0.3s, box-shadow 0.3s;
}
.sync-dot.syncing {
  background: var(--zen);
  box-shadow: 0 0 8px var(--zen);
  animation: pulse 1s ease-in-out infinite;
}
.sync-dot.error {
  background: var(--plasma);
  box-shadow: 0 0 8px var(--plasma);
  animation: none;
}
@keyframes pulse { 0%,100% { opacity:1 } 50% { opacity:0.3 } }
```

### 4.6 Add Recipe Button (parallelogram)

```css
.add-btn {
  font-family: var(--font-head);
  font-size: 0.85rem;
  letter-spacing: 0.08em;
  color: var(--void);
  background: var(--acid);
  padding: 10px 22px 10px 18px;
  clip-path: polygon(8px 0%, 100% 0%, calc(100% - 8px) 100%, 0% 100%);
  transition: background 0.2s, transform 0.15s;
}
.add-btn:hover { background: #d8ff10; transform: translateY(-1px); }

.add-wrapper { position: relative; }

.add-dropdown {
  display: none;
  position: absolute;
  top: calc(100% + 8px);
  right: 0;
  background: var(--ash);
  border: 1px solid var(--gravel);
  border-radius: var(--radius);
  min-width: 200px;
  z-index: 200;
  box-shadow: 0 8px 32px rgba(0,0,0,0.6);
}
.add-dropdown.open { display: block; }
.add-dropdown button {
  display: block;
  width: 100%;
  text-align: left;
  padding: 12px 16px;
  color: var(--bone);
  font-size: 0.95rem;
  transition: background 0.15s, color 0.15s;
}
.add-dropdown button:hover { background: var(--smoke); color: var(--acid); }
```

### 4.7 Toolbar

```css
.toolbar {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  align-items: center;
  padding: 16px 24px;
  background: var(--ash);
  border-bottom: 1px solid var(--gravel);
}

.search-input {
  flex: 1;
  min-width: 200px;
  padding: 10px 14px;
  font-size: 0.95rem;
  outline: none;
}
.search-input:focus { border-color: var(--acid); }

.filter-pills { display: flex; flex-wrap: wrap; gap: 8px; }

.pill {
  font-family: var(--font-ui);
  font-weight: 600;
  font-size: 0.8rem;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  padding: 6px 14px;
  border-radius: 99px;
  border: 1px solid var(--gravel);
  color: var(--fog);
  background: transparent;
  transition: all 0.2s;
}
.pill.active, .pill:hover {
  border-color: var(--acid);
  color: var(--acid);
  background: rgba(200,240,0,0.08);
}
```

### 4.8 Section Header

```css
.section-header {
  display: flex;
  align-items: baseline;
  gap: 12px;
  padding: 24px 24px 12px;
}
.section-header h2 {
  font-family: var(--font-head);
  font-size: 1.4rem;
  color: var(--bone);
}
.recipe-count { font-size: 0.85rem; color: var(--fog); }
.sort-select {
  margin-left: auto;
  padding: 6px 10px;
  font-size: 0.85rem;
  outline: none;
}
```

### 4.9 Recipe Grid & Cards

```css
.recipe-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(var(--card-w), 1fr));
  gap: 20px;
  padding: 0 24px 48px;
}

.recipe-card {
  background: var(--ash);
  border: 1px solid var(--gravel);
  border-radius: 6px;
  overflow: hidden;
  cursor: pointer;
  position: relative;
  transition: transform 0.2s, box-shadow 0.2s;
}
.recipe-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 12px 32px rgba(0,0,0,0.5);
}

/* Acid underline slides in on hover */
.recipe-card::after {
  content: '';
  position: absolute;
  bottom: 0; left: 0;
  width: 0; height: 3px;
  background: var(--acid);
  transition: width 0.25s ease;
}
.recipe-card:hover::after { width: 100%; }

/* Card image area with torn-edge clip */
.card-image {
  width: 100%;
  aspect-ratio: 4/3;
  object-fit: cover;
  display: block;
  /* torn mountain silhouette on bottom edge */
  clip-path: polygon(
    0% 0%, 100% 0%, 100% 78%,
    95% 88%, 90% 76%, 85% 90%,
    78% 80%, 72% 92%, 65% 82%,
    58% 95%, 50% 83%, 42% 93%,
    35% 80%, 28% 90%, 20% 78%,
    12% 88%, 5% 76%, 0% 85%
  );
}

/* Gradient fallback when no photo */
.card-image-placeholder {
  width: 100%;
  aspect-ratio: 4/3;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 3rem;
  clip-path: polygon(
    0% 0%, 100% 0%, 100% 78%,
    95% 88%, 90% 76%, 85% 90%,
    78% 80%, 72% 92%, 65% 82%,
    58% 95%, 50% 83%, 42% 93%,
    35% 80%, 28% 90%, 20% 78%,
    12% 88%, 5% 76%, 0% 85%
  );
}

/* 6 dark gradient palettes, cycled by id hash */
.card-image-placeholder.g0 { background: linear-gradient(135deg, #1a1a2e, #16213e); }
.card-image-placeholder.g1 { background: linear-gradient(135deg, #1a0a0a, #2d1b1b); }
.card-image-placeholder.g2 { background: linear-gradient(135deg, #0a1a0a, #1b2d1b); }
.card-image-placeholder.g3 { background: linear-gradient(135deg, #1a0a1a, #2d1b2d); }
.card-image-placeholder.g4 { background: linear-gradient(135deg, #0a1a1a, #1b2d2d); }
.card-image-placeholder.g5 { background: linear-gradient(135deg, #1a1a0a, #2d2d1b); }

/* Category badge — top-left on image */
.category-badge {
  position: absolute;
  top: 10px; left: 10px;
  font-family: var(--font-ui);
  font-weight: 700;
  font-size: 0.7rem;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  padding: 3px 8px;
  background: rgba(13,13,13,0.75);
  border: 1px solid var(--gravel);
  border-radius: 3px;
  color: var(--fog);
  backdrop-filter: blur(4px);
}

.card-body { padding: 14px 16px 18px; }
.card-name {
  font-family: var(--font-head);
  font-size: 1.05rem;
  color: var(--bone);
  margin-bottom: 4px;
  line-height: 1.2;
}
.card-desc {
  font-family: var(--font-body);
  font-style: italic;
  font-size: 0.82rem;
  color: var(--fog);
  line-height: 1.4;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
.card-meta {
  display: flex;
  gap: 10px;
  margin-top: 10px;
  font-size: 0.78rem;
  color: var(--fog);
}
```

### 4.10 Skeleton Cards

```css
.skeleton {
  animation: shimmer 1.4s ease-in-out infinite;
}
@keyframes shimmer {
  0%,100% { opacity: 0.5 }
  50%      { opacity: 1 }
}
.skeleton-img {
  width: 100%; aspect-ratio: 4/3;
  background: var(--smoke);
}
.skeleton-line {
  height: 14px; border-radius: 3px;
  background: var(--smoke);
  margin: 10px 16px 4px;
}
.skeleton-line.short { width: 55%; }
```

### 4.11 Modals (shared)

```css
.modal-overlay {
  display: none;
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.85);
  z-index: 500;
  align-items: center;
  justify-content: center;
  padding: 16px;
  overflow-y: auto;
}
.modal-overlay.open { display: flex; }

.modal {
  background: var(--ash);
  border: 1px solid var(--gravel);
  border-radius: 6px;
  width: 100%;
  max-width: 680px;
  position: relative;
  padding: 32px;
  max-height: 90vh;
  overflow-y: auto;
}

.modal-close {
  position: absolute;
  top: 16px; right: 16px;
  color: var(--fog);
  font-size: 1.2rem;
  transition: color 0.15s;
}
.modal-close:hover { color: var(--plasma); }
```

### 4.12 View Modal — Recipe Detail

```css
.view-hero { width: 100%; max-height: 300px; object-fit: cover; border-radius: 4px; margin-bottom: 20px; }
.view-name { font-family: var(--font-head); font-size: 2rem; color: var(--bone); line-height: 1.1; }
.view-desc { font-family: var(--font-body); font-style: italic; color: var(--fog); margin: 8px 0 20px; }

.stat-bar {
  display: flex;
  gap: 20px;
  padding: 14px 0;
  border-top: 1px solid var(--gravel);
  border-bottom: 1px solid var(--gravel);
  margin-bottom: 24px;
}
.stat-item { display: flex; flex-direction: column; align-items: center; }
.stat-value { font-family: var(--font-head); font-size: 1.3rem; color: var(--acid); }
.stat-label { font-size: 0.72rem; color: var(--fog); text-transform: uppercase; letter-spacing: 0.1em; }

.section-label {
  font-family: var(--font-head);
  font-size: 0.9rem;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  color: var(--plasma);
  margin-bottom: 10px;
}

/* Ingredient bullets */
.ingredient-list { list-style: none; }
.ingredient-list li::before { content: '✦ '; color: var(--plasma); }
.ingredient-list li { padding: 4px 0; font-size: 0.95rem; }

/* Step numbers */
.step-item { display: flex; gap: 14px; margin-bottom: 14px; align-items: flex-start; }
.step-num {
  font-family: var(--font-head);
  font-size: 0.9rem;
  color: var(--void);
  background: var(--plasma);
  min-width: 28px; height: 28px;
  display: flex; align-items: center; justify-content: center;
  flex-shrink: 0;
}
.step-text { font-family: var(--font-body); font-style: italic; line-height: 1.6; }

.notes-block {
  background: var(--smoke);
  border-left: 3px solid var(--zen);
  padding: 12px 16px;
  border-radius: 0 4px 4px 0;
  font-family: var(--font-body);
  font-style: italic;
  color: var(--fog);
  font-size: 0.9rem;
}

.view-actions {
  display: flex;
  gap: 12px;
  margin-top: 28px;
  padding-top: 20px;
  border-top: 1px solid var(--gravel);
}
.view-actions button {
  font-family: var(--font-ui);
  font-weight: 700;
  letter-spacing: 0.06em;
  font-size: 0.85rem;
  text-transform: uppercase;
  padding: 10px 20px;
  border-radius: var(--radius);
}
#editBtn   { background: var(--acid);   color: var(--void); }
#shareBtn  { background: var(--zen);    color: var(--void); }
#deleteBtn { background: transparent; border: 1px solid var(--plasma); color: var(--plasma); }
#deleteBtn:hover { background: var(--plasma); color: var(--void); }
```

### 4.13 Form Modal

```css
.form-modal h2 { font-family: var(--font-head); font-size: 1.6rem; margin-bottom: 24px; }
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
.form-group { display: flex; flex-direction: column; gap: 6px; margin-bottom: 16px; }
.form-group label { font-size: 0.8rem; text-transform: uppercase; letter-spacing: 0.08em; color: var(--fog); }
.form-group input,
.form-group select,
.form-group textarea {
  padding: 10px 12px;
  font-size: 0.95rem;
  outline: none;
  width: 100%;
}
.form-group input:focus,
.form-group select:focus,
.form-group textarea:focus { border-color: var(--acid); }
.form-group textarea { resize: vertical; min-height: 80px; }

/* Dynamic list items (ingredients / steps) */
.list-item-row { display: flex; gap: 8px; align-items: center; margin-bottom: 8px; }
.list-item-row input { flex: 1; }
.list-item-row .remove-btn { color: var(--plasma); font-size: 1.1rem; padding: 4px 8px; }
.add-list-item {
  font-size: 0.82rem;
  color: var(--zen);
  text-decoration: underline;
  padding: 4px 0;
}

.photo-preview { width: 100%; max-height: 200px; object-fit: cover; border-radius: 4px; margin-top: 8px; }

.form-submit {
  font-family: var(--font-head);
  font-size: 1rem;
  color: var(--void);
  background: var(--acid);
  padding: 14px 32px;
  clip-path: polygon(8px 0%, 100% 0%, calc(100% - 8px) 100%, 0% 100%);
  width: 100%;
  margin-top: 8px;
  transition: background 0.2s;
}
.form-submit:hover { background: #d8ff10; }
```

### 4.14 Capture Modal

```css
.capture-modal { max-width: 560px; }
.capture-tabs { display: flex; gap: 0; margin-bottom: 24px; border-bottom: 2px solid var(--gravel); }
.tab-btn {
  font-family: var(--font-ui);
  font-weight: 700;
  font-size: 0.85rem;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--fog);
  padding: 10px 20px;
  border-bottom: 2px solid transparent;
  margin-bottom: -2px;
  transition: color 0.2s, border-color 0.2s;
}
.tab-btn.active { color: var(--acid); border-bottom-color: var(--acid); }

.capture-drop-zone {
  border: 2px dashed var(--gravel);
  border-radius: 6px;
  padding: 40px 20px;
  text-align: center;
  color: var(--fog);
  cursor: pointer;
  transition: border-color 0.2s;
}
.capture-drop-zone:hover,
.capture-drop-zone.drag-over { border-color: var(--acid); color: var(--acid); }

.capture-preview { width: 100%; max-height: 240px; object-fit: cover; border-radius: 4px; margin: 12px 0; }

.capture-btn {
  font-family: var(--font-ui);
  font-weight: 700;
  font-size: 0.85rem;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: var(--void);
  background: var(--acid);
  padding: 12px 24px;
  border-radius: var(--radius);
  width: 100%;
  margin-top: 12px;
  transition: background 0.2s;
}
.capture-btn:disabled { opacity: 0.4; cursor: not-allowed; }
.capture-btn:not(:disabled):hover { background: #d8ff10; }

.voice-status { text-align: center; color: var(--zen); font-size: 1rem; min-height: 40px; }
.voice-transcript { background: var(--smoke); padding: 12px; border-radius: 4px; min-height: 80px; font-size: 0.9rem; margin-top: 12px; }
```

### 4.15 Toast

```css
.toast {
  position: fixed;
  bottom: 24px; left: 50%;
  transform: translateX(-50%) translateY(80px);
  background: var(--ash);
  border: 1px solid var(--gravel);
  border-left: 4px solid var(--acid);
  color: var(--bone);
  padding: 12px 20px;
  border-radius: var(--radius);
  font-size: 0.9rem;
  z-index: 9000;
  transition: transform 0.3s ease;
  white-space: nowrap;
}
.toast.show { transform: translateX(-50%) translateY(0); }
.toast.error { border-left-color: var(--plasma); }
```

---

## 5. JavaScript Architecture

### 5.1 Config Block

```js
const CONFIG = {
  SUPABASE_URL:  'YOUR_SUPABASE_URL',
  SUPABASE_KEY:  'YOUR_SUPABASE_ANON_KEY',
  ANTHROPIC_KEY: 'YOUR_ANTHROPIC_API_KEY',
  BUCKET:        'recipe-photos',
  MODEL:         'claude-sonnet-4-20250514',
};
```

> **Security note:** The Anthropic key is exposed in client-side JS. This is acceptable for a personal single-user app. For any multi-user deployment, proxy the Anthropic call through a server.

### 5.2 Supabase Initialization

```js
const { createClient } = supabase;
const db = createClient(CONFIG.SUPABASE_URL, CONFIG.SUPABASE_KEY);
```

### 5.3 State

```js
const state = {
  recipes:      [],       // full local array, source of truth for renders
  activeFilter: 'All',    // selected category pill label
  searchQuery:  '',       // current search string
  sortMode:     'date',   // 'date' | 'name'
  editingId:    null,     // uuid of recipe being edited, null for new
  captureTab:   'photo',  // 'photo' | 'voice' | 'text'
  mediaRecorder: null,    // active MediaRecorder instance during voice recording
  audioChunks:   [],      // recorded audio blobs
  recognition:   null,    // SpeechRecognition instance
  isRecording:  false,
};
```

### 5.4 Data Mapping Helpers

```js
// DB row (snake_case) → app object (camelCase)
function fromRow(row) → RecipeObject

// App object → DB row for upsert
function toRow(recipe) → DbRow
```

`fromRow` maps `row.photo_url → recipe.photoUrl`, `row.date_added → recipe.dateAdded`, etc. Arrays stored as JSONB come back as JS arrays already.

### 5.5 ID and Gradient Helpers

```js
function newId() → string   // crypto.randomUUID()

// Deterministic gradient index 0–5 from recipe id string
function gradientIndex(id) → 0..5
// Sum char codes of id, modulo 6
```

### 5.6 Category Config

```js
const CATEGORIES = ['All','Breakfast','Lunch','Dinner','Dessert','Snack','Drinks'];
const CATEGORY_EMOJI = {
  Breakfast: '🌅', Lunch: '🥗', Dinner: '🍽️',
  Dessert: '🍰', Snack: '🫙', Drinks: '🍵', All: '🍴'
};
```

### 5.7 Supabase Helpers

```js
// Load all recipes from DB, sort by created_at desc, populate state.recipes, re-render
async function loadRecipes() → void

// Subscribe to postgres_changes for INSERT/UPDATE/DELETE on recipes table
function subscribeRealtime() → void

// Handler called by realtime subscription
function handleRealtimeChange(payload) → void
// - INSERT: push fromRow(payload.new) into state.recipes
// - UPDATE: replace matching record
// - DELETE: remove by id
// - Call renderGrid() after each

// Upsert recipe object to DB
async function saveRecipeToDB(recipe) → void   // calls db.from('recipes').upsert(toRow(recipe))

// Delete recipe from DB by id
async function deleteRecipeFromDB(id) → void   // calls db.from('recipes').delete().eq('id', id)
```

### 5.8 Photo Helpers

```js
// Upload File object to Supabase storage, return public URL string
async function uploadPhoto(file) → string
// Path: `${Date.now()}_${file.name}`
// Gets public URL via db.storage.from(CONFIG.BUCKET).getPublicUrl(path)

// Delete photo by URL (extract path from URL, call db.storage.from(CONFIG.BUCKET).remove([path]))
async function deletePhoto(url) → void
```

### 5.9 Render Functions

```js
// Render 6 skeleton cards into #recipeGrid during initial load
function renderSkeletons() → void

// Render category pill buttons into #filterPills
// Includes 'All' + each CATEGORY value
// Marks active pill by state.activeFilter
function renderPills() → void

// Compute filtered + sorted subset of state.recipes based on
// state.activeFilter, state.searchQuery, state.sortMode
// Render into #recipeGrid, update #sectionTitle and #recipeCount
function renderGrid() → void

// Build and return HTML string for a single recipe card
function buildCardHTML(recipe) → string

// Build full recipe detail HTML for view modal
function buildViewHTML(recipe) → string
```

### 5.10 Sync Dot Helper

```js
// state: 'synced' | 'syncing' | 'error'
function setSyncState(state) → void
// Sets class on #syncDot element
```

### 5.11 Toast Helper

```js
// Show toast message, auto-hide after 3s
// isError=true applies .error class for plasma border
function showToast(message, isError = false) → void
```

### 5.12 View Modal

```js
// Open view modal for recipe with given id
// Sets innerHTML of #viewContent to buildViewHTML(recipe)
function openView(id) → void

// Share recipe via navigator.share if available, else copy to clipboard
// Format: name, category, prep/cook/servings, ingredients, steps, notes
function shareRecipe(id) → void

// Confirm and delete recipe
// Calls deletePhoto if photo_url exists, then deleteRecipeFromDB
// Removes from state.recipes, closes modal, re-renders grid, shows toast
async function deleteRecipe(id) → void
```

### 5.13 Form Modal

```js
// Open form modal. If id is provided, pre-fill with existing recipe (edit mode).
// If id is null, show blank form (add mode).
// If prefillData object is provided, populate fields from it (post-capture flow).
function openForm(id = null, prefillData = null) → void

// Read all form field values, build recipe object
function collectFormData() → RecipeObject

// Add a new text input row to the ingredients or steps dynamic list
// listId: 'ingredientsList' | 'stepsList'
function addListItem(listId, value = '') → void

// Remove a list item row from ingredients or steps
function removeListItem(btn) → void

// Get array of values from a dynamic list
function getListValues(listId) → string[]

// Main save handler: validate, upload photo if new file selected,
// upsert to DB, update state, close modal, show toast
async function saveRecipe(e) → void
```

### 5.14 Capture Modal

```js
// Open capture modal on the specified tab ('photo' | 'voice' | 'text')
function openCapture(tab) → void

// Switch capture modal to specified tab and render its content
function switchCaptureTab(tab) → void

// Render the photo tab UI (drop zone + file input + camera button + preview)
function renderPhotoTab() → void

// Render the voice tab UI (mic button, status, transcript area)
function renderVoiceTab() → void

// Render the text tab UI (large textarea + extract button)
function renderTextTab() → void

// Handle file selected from input or dropped onto drop zone
// Sets preview image, stores file ref for AI call
function handlePhotoFile(file) → void

// Attempt to open device camera via getUserMedia, capture snapshot to canvas,
// convert to Blob, call handlePhotoFile
async function openCamera() → void

// Start SpeechRecognition (or MediaRecorder fallback) for voice input
function startVoice() → void

// Stop voice recording, update UI
function stopVoice() → void

// Core AI call — sends image (base64), voice transcript, or pasted text
// to Claude API and returns structured recipe JSON
// type: 'image' | 'text'
// content: base64 data URL string (image) or plain text string
async function aiParse(type, content) → RecipeObject

// Called after successful AI extraction:
// Closes capture modal, opens form modal pre-filled with extracted data
function prefillForm(recipeData) → void
```

### 5.15 Anthropic AI Helper (detail)

```js
async function aiParse(type, content) → RecipeObject {
  // Build messages array:
  // - If type === 'image': use vision message with base64 image
  // - If type === 'text':  use text message with raw content
  //
  // System prompt instructs Claude to extract a recipe and return ONLY
  // a JSON object matching the recipe schema (no markdown, no prose).
  //
  // POST to https://api.anthropic.com/v1/messages
  // Headers: x-api-key, anthropic-version: 2023-06-01,
  //          content-type: application/json
  //          anthropic-dangerous-direct-browser-access: true
  //
  // Model: CONFIG.MODEL
  // max_tokens: 2048
  //
  // Parse response.content[0].text as JSON
  // Return parsed object (or throw on failure)
}
```

**System prompt text:**
```
You are a recipe extraction assistant. The user will provide a recipe via image or text.
Extract the recipe and return ONLY a valid JSON object with these exact keys:
name, category (one of: Breakfast, Lunch, Dinner, Dessert, Snack, Drinks), desc,
prep, cook, servings (integer), ingredients (array of strings), steps (array of strings),
notes, inspired_by.
If a field cannot be determined, use null. Return no other text, no markdown fences.
```

### 5.16 Init

```js
document.addEventListener('DOMContentLoaded', async () => {
  renderSkeletons();
  renderPills();
  wireEvents();
  await loadRecipes();
  subscribeRealtime();
});
```

### 5.17 Event Wiring (`wireEvents`)

All DOM event listeners set up once in `wireEvents()`:

| Element | Event | Handler |
|---|---|---|
| `#addBtn` | click | toggle `#addDropdown.open` |
| `document` | click | close dropdown if click outside |
| `#addDropdown button[data-action]` | click | route to `openCapture(tab)` or `openForm()` |
| `#searchInput` | input | debounce 250ms → update `state.searchQuery`, `renderGrid()` |
| `#filterPills` | click (delegated) | update `state.activeFilter`, `renderPills()`, `renderGrid()` |
| `#sortSelect` | change | update `state.sortMode`, `renderGrid()` |
| `#recipeGrid` | click (delegated) | `openView(card.dataset.id)` |
| `#viewClose` | click | close view modal |
| `#editBtn` | click | `openForm(state.viewingId)` |
| `#shareBtn` | click | `shareRecipe(state.viewingId)` |
| `#deleteBtn` | click | `deleteRecipe(state.viewingId)` |
| `#formClose` | click | close form modal |
| `#recipeForm` | submit | `saveRecipe(e)` |
| `#captureClose` | click | close capture modal |
| `.tab-btn` | click | `switchCaptureTab(tab)` |
| `#viewModal`, `#formModal`, `#captureModal` | click on overlay | close that modal |
| `document` | keydown `Escape` | close topmost open modal |

---

## 6. Component Breakdown

### 6.1 Header

**HTML structure:** logo div + header-right (sync-dot + add-wrapper).
**Inputs:** none (static).
**Outputs:** opens add dropdown on button click.
**Behavior:** skewed plasma stripe is a CSS `::after` pseudo-element. Sync dot class updated by `setSyncState()`. Dropdown closes on outside click via document listener.

---

### 6.2 Toolbar

**HTML:** search input + filter pills container.
**Inputs:** user typing, pill clicks.
**Outputs:** updates `state.searchQuery` / `state.activeFilter`, triggers `renderGrid()`.
**Behavior:** search is debounced 250ms. Pills re-render on filter change to toggle `.active` class.

---

### 6.3 Recipe Grid

**HTML:** CSS grid of `.recipe-card` divs.
**Inputs:** `state.recipes`, `state.activeFilter`, `state.searchQuery`, `state.sortMode`.
**Outputs:** clicking a card calls `openView(id)`.
**Behavior:**
- Filter logic: if activeFilter !== 'All', only show matching category. Also apply searchQuery against name + ingredients joined + category (case-insensitive).
- Sort: 'date' → by `dateAdded` descending; 'name' → alphabetical.
- Each card has `data-id` attribute for event delegation.
- Gradient index computed from `id` string char code sum mod 6.
- Shows skeleton cards during initial load.

---

### 6.4 View Modal (`#viewModal`)

**HTML:** `.modal-overlay > .modal.view-modal` with `#viewContent` div + action buttons.
**Inputs:** recipe id passed to `openView()`.
**Outputs:** opens form modal (edit), triggers share, triggers delete.
**Behavior:**
- `buildViewHTML(recipe)` generates: hero image (if photoUrl) or placeholder, name, desc, stat bar (prep/cook/serves), ingredient list with ✦ bullets, numbered steps with plasma squares, notes block, inspired_by line.
- `state.viewingId` is set so action buttons know which recipe to act on.
- Delete: shows `confirm()` dialog before proceeding.
- Share: builds plain text string → `navigator.share()` if available, else `navigator.clipboard.writeText()` + toast.
- Closes when overlay background clicked or Escape pressed.

---

### 6.5 Form Modal (`#formModal`)

**HTML structure:**
```
.modal.form-modal
  h2 (Add Recipe / Edit Recipe)
  form#recipeForm
    .form-group  name (required)
    .form-row
      .form-group  category (select)
      .form-group  servings (number)
    .form-row
      .form-group  prep (text)
      .form-group  cook (text)
    .form-group  desc (textarea, 2 rows)
    .form-group  Ingredients
      #ingredientsList  (dynamic rows)
      button.add-list-item "＋ Add Ingredient"
    .form-group  Steps
      #stepsList  (dynamic rows)
      button.add-list-item "＋ Add Step"
    .form-group  notes (textarea)
    .form-group  inspired_by (text)
    .form-group  Photo
      input[type=file accept="image/*"]
      img.photo-preview (hidden until file selected)
    button.form-submit "Save Recipe"
```

**Inputs:** recipe data (edit) or prefill data (post-capture) or blank (manual).
**Outputs:** calls `saveRecipe()` on submit.
**Behavior:**
- `openForm(id, prefillData)`: if `id` is set, find recipe in `state.recipes` and populate. If `prefillData`, use that. Otherwise blank.
- Dynamic ingredient/step lists: each row is `input[type=text]` + remove button. "Add" link appends a new row. Remove button calls `removeListItem(btn)`.
- Photo field: change event shows preview, stores `File` in `state.pendingPhotoFile`.
- On save: validate name not empty. Upload photo if `state.pendingPhotoFile` exists. Build recipe object (new `id` for adds, existing for edits). `date_added` only set on creates. `date_modified` always updated. Upsert to DB. Update `state.recipes`. Close modal. Toast.

---

### 6.6 Capture Modal (`#captureModal`)

Three tabs:

**Photo tab:**
- Drop zone div (handles `dragover`, `dragleave`, `drop`)
- Hidden `input[type=file accept="image/*,image/heic"]`
- "📷 Choose File" button triggers file input click
- "📸 Use Camera" button → `openCamera()`
- Preview `<img>` shown after file selected
- "Extract Recipe" button → `aiParse('image', base64DataUrl)`

**Voice tab:**
- "🎙 Start Recording" / "⏹ Stop" toggle button
- Status line (colored text, e.g. "Listening…", "Transcript ready")
- Transcript `<div>` shows interim/final SpeechRecognition results
- "Extract Recipe" button (enabled when transcript non-empty) → `aiParse('text', transcript)`
- **Primary:** Web Speech API (`window.SpeechRecognition || window.webkitSpeechRecognition`), continuous mode with interim results
- **Fallback message** if SpeechRecognition not available (e.g., Firefox): show instruction to paste transcript instead

**Text tab:**
- `<textarea>` placeholder: "Paste a recipe from anywhere…"
- "Extract Recipe" button → `aiParse('text', textarea.value)`

**All tabs:** after successful AI parse, call `prefillForm(data)` which closes capture modal and opens pre-filled form modal.

---

## 7. API Integrations

### 7.1 Supabase — Database

**Client init:**
```js
const db = createClient(CONFIG.SUPABASE_URL, CONFIG.SUPABASE_KEY);
```

**Load:**
```js
const { data, error } = await db
  .from('recipes')
  .select('*')
  .order('created_at', { ascending: false });
```

**Upsert:**
```js
await db.from('recipes').upsert(toRow(recipe));
```

**Delete:**
```js
await db.from('recipes').delete().eq('id', id);
```

**Error handling:** All DB calls wrapped in try/catch. On error: `setSyncState('error')`, `showToast(error.message, true)`.

---

### 7.2 Supabase — Storage

**Upload:**
```js
const path = `${Date.now()}_${file.name}`;
const { error } = await db.storage.from(CONFIG.BUCKET).upload(path, file, {
  cacheControl: '3600', upsert: false
});
const { data } = db.storage.from(CONFIG.BUCKET).getPublicUrl(path);
return data.publicUrl;
```

**Delete:**
```js
// Extract path from URL (everything after /object/public/recipe-photos/)
const path = url.split('/recipe-photos/')[1];
await db.storage.from(CONFIG.BUCKET).remove([path]);
```

---

### 7.3 Supabase — Realtime

```js
db.channel('recipes-changes')
  .on('postgres_changes',
    { event: '*', schema: 'public', table: 'recipes' },
    handleRealtimeChange)
  .subscribe((status) => {
    if (status === 'SUBSCRIBED') setSyncState('synced');
    else setSyncState('syncing');
  });
```

**`handleRealtimeChange(payload)`:**
- `INSERT`: push `fromRow(payload.new)` to front of `state.recipes`
- `UPDATE`: find by `payload.new.id`, replace with `fromRow(payload.new)`
- `DELETE`: filter out `payload.old.id`
- Re-render grid after each

---

### 7.4 Anthropic Claude API

**Endpoint:** `https://api.anthropic.com/v1/messages`

**Required headers:**
```
x-api-key: CONFIG.ANTHROPIC_KEY
anthropic-version: 2023-06-01
content-type: application/json
anthropic-dangerous-direct-browser-access: true
```

> The `anthropic-dangerous-direct-browser-access` header is **required** for direct browser calls. Without it, the API will reject the request with a CORS/security error.

**Image request shape:**
```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 2048,
  "system": "...extraction system prompt...",
  "messages": [{
    "role": "user",
    "content": [
      {
        "type": "image",
        "source": {
          "type": "base64",
          "media_type": "image/jpeg",
          "data": "<base64 string without data: prefix>"
        }
      },
      { "type": "text", "text": "Extract the recipe from this image." }
    ]
  }]
}
```

**Text request shape:**
```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 2048,
  "system": "...extraction system prompt...",
  "messages": [{
    "role": "user",
    "content": "<raw recipe text or voice transcript>"
  }]
}
```

**Response parsing:**
```js
const json = await response.json();
const text = json.content[0].text.trim();
const recipe = JSON.parse(text);
```

**Base64 conversion:**
```js
// Convert File to base64 data URL, then strip the prefix
function fileToBase64(file) → Promise<string>
// reader.readAsDataURL(file) → resolve with result.split(',')[1]
```

**Image media type:** detect from `file.type` (`image/jpeg`, `image/png`, `image/gif`, `image/webp`). HEIC images may need conversion — show an error and ask user to convert first if `file.type` is not one of the four supported types.

---

## 8. Build Order

Build in this sequence to minimize blocked dependencies:

### Phase 1 — Scaffold & Design System
1. Create `index.html` with `<head>` (fonts, CDN links)
2. Add all CSS: tokens, reset, grain, header, toolbar, modal base
3. Add static HTML for header, toolbar, main grid area, all three modals (empty/placeholder content)
4. Verify visual layout looks correct with hardcoded placeholder cards

### Phase 2 — Data Layer
5. Run Supabase SQL (table + RLS + storage bucket)
6. Add CONFIG block and Supabase client init
7. Implement `fromRow()`, `toRow()`, `newId()`, `gradientIndex()`
8. Implement `loadRecipes()` — load from DB, store in `state.recipes`
9. Verify data loads by logging to console

### Phase 3 — Render Layer
10. Implement `renderSkeletons()`
11. Implement `renderPills()` with static CATEGORIES
12. Implement `buildCardHTML()` — cards with gradient fallback, category badge, photo
13. Implement `renderGrid()` with filter + search + sort logic
14. Wire search input, pill clicks, sort select
15. Verify grid renders correctly with DB data

### Phase 4 — View Modal
16. Implement `buildViewHTML()`
17. Implement `openView()`, close logic
18. Implement `shareRecipe()`
19. Verify full detail view displays correctly

### Phase 5 — Form Modal (Manual Entry)
20. Build form HTML with all fields, dynamic list rows
21. Implement `openForm()`, `collectFormData()`, `addListItem()`, `removeListItem()`, `getListValues()`
22. Implement `saveRecipe()` — upsert, local state update, toast
23. Implement `deleteRecipe()` — confirm, delete from DB + storage, state update
24. Wire Edit / Delete buttons in view modal
25. Verify full add/edit/delete cycle works manually

### Phase 6 — AI Capture
26. Implement `aiParse()` with system prompt and both content shapes
27. Implement `fileToBase64()`
28. Implement `renderPhotoTab()` — drop zone, file input, camera, preview, extract button
29. Implement `openCamera()` via `getUserMedia`
30. Implement `renderVoiceTab()` — SpeechRecognition, status display
31. Implement `renderTextTab()` — textarea + extract
32. Implement `prefillForm()` — close capture modal, open pre-filled form modal
33. Verify each capture flow end-to-end

### Phase 7 — Realtime & Sync
34. Implement `subscribeRealtime()` and `handleRealtimeChange()`
35. Implement `setSyncState()`
36. Open app in two browser tabs and verify live sync

### Phase 8 — Polish
37. Add `showToast()` across all error/success paths
38. Add photo upload/delete in storage during save/delete
39. Verify all Escape / overlay-click close behaviors
40. Final visual polish pass (animations, hover states, grain overlay opacity)
41. Test on mobile viewport

---

## 9. Testing Checklist

### Data Operations
- [ ] Recipe created via manual form appears in grid immediately
- [ ] Recipe edited via form shows updated data in grid and detail view
- [ ] Recipe deleted is removed from grid, photo deleted from storage
- [ ] Opening app fresh loads all recipes from DB
- [ ] Two browser tabs stay in sync via Realtime (insert/update/delete visible in other tab within ~1s)

### Capture Flows
- [ ] Photo upload: JPEG/PNG file → AI extracts → form pre-filled → save works
- [ ] Photo drag-and-drop works on drop zone
- [ ] Camera capture: device camera opens, snapshot taken, AI extracts
- [ ] Voice: SpeechRecognition starts/stops, transcript captured, AI extracts
- [ ] Text paste: pasted text sent to AI, result pre-fills form
- [ ] AI returns null fields gracefully (form shows blank, not "null")

### Grid & Search
- [ ] Category pills filter correctly including "All"
- [ ] Search matches recipe name
- [ ] Search matches ingredient (e.g., searching "garlic" surfaces recipes with garlic)
- [ ] Sort by name sorts alphabetically A–Z
- [ ] Sort by date shows newest first
- [ ] Recipe count updates with filter/search
- [ ] Skeleton cards show during initial load, replaced by real cards

### UI/Visual
- [ ] Grain texture overlay visible (subtle) without blocking clicks
- [ ] Card torn-edge clip-path renders correctly (not clipped weirdly)
- [ ] Gradient fallback shows for recipes without photos, using 6 distinct palettes
- [ ] Category badge visible on each card image area
- [ ] Card hover: lifts 5px, acid underline slides in
- [ ] Add Recipe button: parallelogram shape visible
- [ ] Sync dot is green/synced after initial load, teal/pulsing during load, plasma on error
- [ ] Toast appears and auto-dismisses on save and delete

### Modals
- [ ] All three modals open/close correctly
- [ ] Escape key closes topmost open modal
- [ ] Clicking overlay backdrop closes modal
- [ ] Form modal correctly differentiates Add vs Edit (title, pre-fill)
- [ ] View modal share button works (tries navigator.share, falls back to clipboard)
- [ ] View modal delete shows confirm dialog before deleting

### Mobile / Responsive
- [ ] Grid goes to 1 column on narrow screens
- [ ] Toolbar wraps gracefully on small screens
- [ ] Modals scroll internally if content is tall
- [ ] Camera capture works on iOS Safari (getUserMedia prompt appears)
- [ ] Touch interactions work for cards, buttons, pills

---

## 10. Known Constraints & Gotchas

### Anthropic API — Direct Browser Access
The API rejects browser calls without `anthropic-dangerous-direct-browser-access: true` header. This must be included in every fetch call. Additionally, standard CORS preflight applies — the Anthropic API does support this header for personal/dev use, but it intentionally signals that the API key is exposed client-side.

### API Key Exposure
The Anthropic API key is visible in the page source. This is intentional for a single-user personal app. Do not deploy this app publicly on the internet. For any shared deployment, move AI calls to a server-side proxy (e.g., a Supabase Edge Function).

### Supabase RLS
The "allow all" policy bypasses authentication entirely. This is fine for a personal app on a private Supabase project. Ensure the Supabase project itself is not exposed to the public internet or add IP restrictions in the Supabase dashboard.

### SpeechRecognition Browser Support
`window.SpeechRecognition` is only available in Chromium-based browsers (Chrome, Edge). Firefox and Safari have partial or no support. Detect availability and show a fallback message ("Voice dictation requires Chrome or Edge. Try pasting text instead.") rather than a broken UI.

### Camera on iOS Safari
`getUserMedia` requires HTTPS. If serving locally over plain HTTP, camera access will be denied. Use a tunneling tool (ngrok, etc.) or serve via localhost (which is exempt). Also, iOS Safari requires `{ video: { facingMode: 'environment' } }` for the rear camera.

### HEIC/HEIF Images
iPhones save photos as HEIC by default. The Claude API does not accept HEIC; it only accepts JPEG, PNG, GIF, and WebP. Detect `file.type === 'image/heic'` and show an error asking the user to convert the image. On iOS, the file input with `accept="image/*"` often returns JPEG regardless — test on real device.

### Image File Size
Claude API has a max image size. Convert large images to JPEG at reduced quality before sending to the API. Use a canvas element: draw the image, then `canvas.toBlob(callback, 'image/jpeg', 0.85)`. Aim for under 5MB for the base64 payload.

### Supabase Realtime + Local State
The Realtime subscription may fire for changes you yourself just made (the upsert you triggered). This causes a double-render (once from optimistic local update, once from realtime). Either: (a) skip realtime updates for records whose `id` matches `state.lastSavedId` for a 2-second window, or (b) accept the harmless double-render since `renderGrid()` is idempotent.

### `desc` as a Column Name
`desc` is a reserved word in SQL (used in `ORDER BY ... DESC`). Supabase/PostgREST handles this fine as a column name in JSON API calls, but be aware it may cause confusion in raw SQL queries. Wrap in quotes (`"desc"`) in any raw SQL if needed.

### Storage URL Extraction for Delete
When extracting the storage path from a `photo_url` for deletion, the URL format is:
```
https://<project>.supabase.co/storage/v1/object/public/recipe-photos/<path>
```
Split on `/recipe-photos/` and take index `[1]` to get the storage path.

### Modal Z-Index Stack
Three modals can theoretically overlap (e.g., capture modal → form modal). Keep a simple stack: close capture before opening form (`prefillForm` does this). Only one modal should be visible at a time. Use a single `z-index: 500` for all modals — they are shown/hidden via the `.open` class.

### `crypto.randomUUID()`
Available in modern browsers and on HTTPS. On HTTP (local dev without localhost), it may be unavailable. Fallback:
```js
function newId() {
  return crypto.randomUUID?.() ??
    'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
      const r = Math.random() * 16 | 0;
      return (c === 'x' ? r : (r & 0x3 | 0x8)).toString(16);
    });
}
```

### Debounce for Search
Without debouncing, every keystroke triggers `renderGrid()`. A simple 250ms debounce is sufficient and avoids jank on large recipe collections.

### Supabase CDN Version Lock
The CDN URL `@supabase/supabase-js@2` may pull the latest `2.x` minor version on each load. For stability, pin to a specific version (e.g., `@supabase/supabase-js@2.39.0`) once confirmed working.
