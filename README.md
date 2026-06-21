# Mileage Tracker — เอกสารโปรเจกต์

> Single-file PWA บันทึกระยะทางและน้ำมันรถยนต์ 13 ทะเบียน  
> อัปเดตล่าสุด: 2026-06-21

---

## ลิงก์สำคัญ

| | |
|--|--|
| **Live App** | https://atomnet16.github.io/mileage-tracker-/ |
| **GitHub** | https://github.com/atomnet16/mileage-tracker- |
| **Google Sheets** | เชื่อมผ่าน GAS Web App (ดู URL ในแอพ → ตั้งค่า) |

---

## โครงสร้างไฟล์

```
mileage-tracker/
├── index.html      ← แอพทั้งหมด (HTML + CSS + JS รวมไฟล์เดียว)
├── manifest.json   ← PWA manifest (ชื่อแอพ, icon, theme color)
├── sw.js           ← Service Worker (offline cache)
├── icon.png        ← App icon (AI-generated: road + GPS pins, teal bg)
└── README.md       ← ไฟล์นี้
```

---

## Deploy

```bash
git clone https://github.com/atomnet16/mileage-tracker-.git
git add .
git commit -m "message"
git push origin main
# GitHub Pages auto-deploy ~2 นาที
```

---

## Design System — v2 Teal

### CSS Variables
```css
:root {
  --bg:      #F2F6F6;
  --surface: #FFFFFF;
  --surf2:   #F3F7F7;
  --border:  #E2EAE9;
  --bord2:   #B2C8C6;
  --text:    #0F1F1E;
  --muted:   #5E7C7A;
  --muted2:  #9DB8B6;
  --primary: #0D9488;
  --plight:  #CCFBF1;
  --pdark:   #134E4A;
  --green:   #10B981;
  --amber:   #F59E0B;
  --red:     #EF4444;
}
```

### UI Sizes (ปัจจุบัน — หลัง scale up)

| Component | Size |
|-----------|------|
| Logo icon | 50×50px, font 26px |
| Logo title | 20px |
| Clock | 22px / 13px |
| Lang button | 14px, padding 7×14px |
| Card padding | 22px, radius 20px |
| Card title | 17px |
| Field label | 14px |
| Input | 18px, padding 15×18px |
| Select | 17px, padding 16×18px |
| Button primary | 19px, padding 20px |
| Tab icon (SVG) | 28×28px, circle 46px |
| Tab label | 13px |
| Record card padding | 18px |
| Record plate tag | 16px |
| Mini stat value | 18px |
| Summary stat value | 17px |
| Max-width | 960px (tablet/desktop) |

### Fonts
- IBM Plex Sans Thai — UI ทั่วไป
- IBM Plex Mono — ตัวเลข, code

---

## โครงสร้าง HTML

### 4 แท็บ

| # | ชื่อ | id | เนื้อหา |
|---|------|----|---------|
| 0 | บันทึก | `#tab0` | เลือกทะเบียน, กม.เริ่ม/จบ, เติมน้ำมัน |
| 1 | ประวัติ | `#tab1` | รายการทั้งหมด + ปุ่ม Sync |
| 2 | สรุป | `#tab2` | km รวม, น้ำมัน, km/L รายทะเบียน |
| 3 | ตั้งค่า | `#tab3` | **ซ่อน** `display:none!important` |

### Bottom Tab Bar (อยู่ก่อน `</body>`)
```html
<nav id="tabbar">
  <button class="tab-btn active" onclick="switchTab(0)" id="tabBtn0">
    <div class="tab-icon"><svg width="28" height="28"><!-- plus-circle --></svg></div>
    <span class="tab-label"></span>
  </button>
  <!-- tabBtn1=calendar, tabBtn2=bar-chart, tabBtn3=gear(hidden) -->
</nav>
```

### kmStart readonly — ใช้ CSS vars เสมอ
```html
style="background:var(--plight);color:var(--primary);
       cursor:not-allowed;border-color:var(--bord2);pointer-events:none;"
```

---

## JavaScript

### ตัวแปรหลัก
```js
const STORAGE_KEY    = 'mileage_records_v1'
const WEBAPP_URL_KEY = 'mileage_webapp_url'
const LANG_KEY       = 'mileage_lang'
const DEFAULT_WEBAPP_URL = 'https://script.google.com/macros/s/AKfycbwBw0jhD1wzmno6btxX69uOX3EqGhiWNfV0dd0LB4Jg0VZejESHjtkpHOHPnaPl5Lw/exec'
```

### applyLang() — ต้องอัป `.tab-label` span เท่านั้น
```js
// ✅ ถูก
['0','1','2','3'].forEach((i) => {
  const btn = document.getElementById('tabBtn' + i);
  const lbl = btn && btn.querySelector('.tab-label');
  if (lbl) lbl.textContent = tx.tabs[i];
});
// ❌ ผิด — ลบ SVG icon ออก
document.getElementById('tabBtn0').textContent = 'บันทึก';
```

### switchTab(i)
```js
document.querySelectorAll('.tab-btn').forEach((b,j) => b.classList.toggle('active', i===j));
document.querySelectorAll('.tab-panel').forEach((p,j) => p.classList.toggle('active', i===j));
```

---

## Data Model

```js
{
  id:         Date.now(),
  plate:      "2AP-7433",
  kmStart:    12345,
  kmEnd:      12500,
  fuelLiters: 30.5,
  fuelKm:     12400,
  ts:         1718323200000,
  synced:     false
}
```

---

## ทะเบียนรถ (13 คัน — hardcoded)

```
2AC-9758  2AP-7433  2AP-5911  2AP-7622
3D-5161   2AE-4783  2H-2330   2AP-5733
2AP-8522  2AP-6499  2AP-7933  2AP-8621
2AP-6955
```

> แก้ใน `<select id="selPlate">` และ `ALL_PLATES` array ใน JS

---

## Logic สำคัญ

### Auto-fill kmStart
```
วันนี้มี kmStart แล้ว  →  readonly แสดงค่าเดิม
ยังไม่มีวันนี้          →  ดึง kmEnd ล่าสุดของเมื่อวาน
ไม่มีประวัติเลย         →  กรอกเองครั้งแรก
```
`window._todayRecId` — เก็บ id ของ record วันนี้

### Save
```
มี _todayRecId + kmEnd  →  UPDATE record เดิม
ไม่มี                   →  CREATE ใหม่ (unshift)
```

---

## Google Apps Script

| action | payload | response |
|--------|---------|----------|
| `sync` | `{action:'sync', records:[...]}` | `{success, added, updated}` |
| `getAll` | `{action:'getAll'}` | `{success, records:[...]}` |
| `delete` | `{action:'delete', id}` | `{success}` |
| GET ping | _(ไม่มี body)_ | `{status:'ok'}` |

Auto-connect on load: Ping → getAll → merge → sync pending

---

## PWA

### manifest.json
```json
{
  "name": "Mileage Tracker",
  "start_url": "/mileage-tracker-/",
  "display": "standalone",
  "theme_color": "#0D9488",
  "icons": [
    { "src": "icon.png", "sizes": "192x192", "purpose": "any" },
    { "src": "icon.png", "sizes": "512x512", "purpose": "maskable" }
  ]
}
```

### sw.js — Cache Strategy
| Request | Strategy |
|---------|----------|
| `script.google.com` (GAS) | Network only → offline fallback JSON |
| `fonts.googleapis.com` | Cache first |
| App assets | Cache first → network fallback |

Cache name: `mileage-v1` — เปลี่ยน version เมื่ออัปเดตไฟล์

### Registration (index.html)
```js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('sw.js').catch(() => {});
  });
}
```

---

## ประวัติการเปลี่ยนแปลง

| วันที่ | เปลี่ยน |
|--------|--------|
| 2026-06-16 | สร้างแอพ + Limestone theme |
| 2026-06-19 | v2 Teal UI + bottom tab bar + SVG icons |
| 2026-06-19 | เพิ่ม PWA: manifest.json + sw.js + icon.png |
| 2026-06-21 | Scale up UI ~1.4x สำหรับ mobile readability |
