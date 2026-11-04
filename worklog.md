# Crypt Loop — Worklog

บันทึกงานที่ทำในแต่ละ session เรียงใหม่สุดไว้บน

## 2026-11-04 — ลอง plugin parallax-threejs

**ผลสรุป:** bridge ของ `threejs-devtools-mcp` ใช้ได้ดี ไม่เจอ leak. `checkpoint` ผ่าน `deno task` ยังรันไม่ผ่านเพราะ Chromium ตัวเต็มเปิดไม่ขึ้นบนเครื่องนี้.

### ทำแล้ว
- ทดสอบ leak: วน `startRun()` 5 รอบแล้วอีก 5 รอบ (รวม 10) ค่าคงที่ที่ geometries 211, textures 10, programs 13 → `clearAll()` ไม่ leak. ค่า "orphaned" ที่ `dispose_check` รายงาน (geometry 135, texture 9) น่าจะเป็น `G` cache กับ render target ของ post-processing (ยังไม่ได้เทียบรายตัว)
- แก้ [crypt-loop.html](crypt-loop.html) ท้าย block `?dbg` (+5 บรรทัด): เพิ่ม `window.scene` (getter คืน `cur`) กับ `window.__renderer__`. ทดสอบแล้ว: town 170 children, หลัง `startRun()` 187 children, `__renderer__.render` เป็น function
- ลง Deno 2.9.8 (`winget install DenoLand.Deno`)
- รัน init: copy scaffold 25 ไฟล์ลง `lib/`, `scripts/`, `test/visual/` และสร้าง `deno.json` (7 tasks). `deno task` อ่านครบ 7 tasks
- ลง Playwright Chromium (`deno run -A npm:playwright install chromium`)

### แก้แล้ว (หลังเขียนส่วนบน)
- `deno task checkpoint` ใช้ได้แล้ว: แก้ `scripts/lib/live-scene.ts` บรรทัด 172 ให้อ่าน env `PARALLAX_CHROME_PATH` ก่อน (ถ้าไม่ตั้ง ใช้ `chromium.executablePath()` เหมือนเดิม). วิธีรัน (PowerShell):
  `$env:DEV_PORT='8771/crypt-loop.html?dbg'; $env:PARALLAX_CHROME_PATH='C:\Users\User\AppData\Local\ms-playwright\chromium_headless_shell-1248\chrome-headless-shell-win64\chrome-headless-shell.exe'; deno task checkpoint <label>`
- ผลรันแรก (label `baseline`): 228 objects, 83 geometries, 8 textures, screenshot เห็นหน้า title ปกติ (ไม่ดำ). ยังไม่ได้ทดสอบ checkpoint ตอนอยู่ใน run หรือ diff-checkpoints
- ไฟล์ `scripts/lib/live-scene.ts` ถูกแก้เอง plugin อัปเดต/init ใหม่อาจต้องแก้ซ้ำ

### ลองแล้วไม่ได้ผล: checkpoint ใน run + diff
- ใช้ headless shell แล้ว Chrome ไม่ค้างอยู่หลัง script จบ (port 9223 ไม่ listen หลังรัน) ทุกคำสั่งเลยเปิด page ใหม่ → state ของ run หาย. `S` เป็น null, `mode` เป็น `title`, uptime ของ page 1 วินาที
- ผลคือ checkpoint label `run` จริงๆ เป็นหน้า title (scene 228 objects เท่ากับ baseline) และ `diff-checkpoints run` เทียบหน้า title สองรอบ (pixelDiffRatio 0.0006 = noise) ไม่ได้บอกอะไรเกี่ยวกับ boss
- `spawnBoss()` ผ่าน driver ล้มด้วย `Cannot read properties of null (reading 'floor')` เพราะ `S` เป็น null (ไม่ใช่บั๊กของเกม)
- ยังไม่รู้สาเหตุที่ Chrome ตาย (สงสัย `browser.close()` หรือ process ตายตาม Deno บน Windows ยังไม่ได้ตรวจ)

### ได้ผล: run + boss ใน process เดียว
- script ใน scratchpad (`boss-diff.ts`) attach ครั้งเดียวแล้วทำ startRun → tick(60) → สรุป scene → spawnBoss → tick(30) → สรุป scene. ใช้ `getSceneSummary` ของ plugin
- ผล: 209 → 237 objects (+28: Mesh 198→222, Group 4→8), enemies 2→4, `S.boss` false→true. PointLight 4 เท่าเดิม
- screenshot หลัง spawn เห็น BONE KING bar กับข้อความ boss ในจอ → `window.scene` getter ตาม `cur` ถูกต้อง และ boss ที่ bridge เคยมองไม่เห็นมาจากการจับ town scene ผิดตัว (สรุปจากหลักฐานนี้ ยังไม่ได้รัน bridge ซ้ำ)
- ข้อจำกัด: `checkpoint`/`diff-checkpoints` ของ plugin ยังใช้ข้ามคำสั่งไม่ได้ เพราะ Chrome ตายทุกครั้ง

### ปัญหาที่ยังค้าง
- `chrome.exe` ตัวเต็ม (`chromium-1248`) เปิดไม่ขึ้น, `os error 14001` (side-by-side configuration). สาเหตุจริงยังไม่รู้ (เลี่ยงด้วย headless shell ข้างบน)
- scene_diff ผ่าน bridge รอบแรกไม่เห็น boss ที่ spawn (216 objects ก่อนและหลัง). น่าจะเป็นเพราะ bridge จับ town scene. ยังไม่ได้รัน diff ซ้ำหลังเพิ่ม `window.scene`
- screenshot ผ่าน bridge ได้ภาพดำ (ยังไม่ยืนยันสาเหตุ สงสัย canvas ไม่มี `preserveDrawingBuffer` หรือหน้าไม่ได้ foreground)
- MCP `spector` เชื่อมต่อไม่ได้ (`CONNECTION_CLOSED`)

### ข้อควรรู้
- script ของ plugin เปิดแค่ `http://localhost:<DEV_PORT>` ไม่มี path. ใช้ `DEV_PORT='8771/crypt-loop.html?dbg'` ได้ (ยังไม่ได้ยืนยันเพราะ script ล้มก่อน)
- ไม่ได้สร้าง `.mcp.json`: plugin เปิด MCP servers ชุดเดียวกันให้แล้ว และ `${CLAUDE_PLUGIN_ROOT}` ของ spector ไม่ resolve นอก plugin
- bridge ของ `threejs-devtools-mcp`: `set_dev_port` ให้ตรง server แล้วเปิด `http://localhost:65345/crypt-loop.html?dbg` ใน Playwright

### สถานะ repo (ยังไม่ commit)
- แก้: `crypt-loop.html`
- ไฟล์ใหม่ยังไม่ track: `deno.json`, `deno.lock`, `lib/`, `scripts/`, `test/`, `worklog.md`
- ค้าง: `.playwright-mcp/` (log ของ Playwright, ควรลบ) และ `.parallax/` จะโผล่เมื่อรัน checkpoint สำเร็จ (ควรใส่ `.gitignore`)
- artifact ที่ publish ไว้ไม่ต้อง republish: โค้ดที่เพิ่มทำงานเฉพาะเมื่อมี `?dbg`

### ต่อไป
- [ ] ตัดสินใจ: แก้ `scripts/lib/live-scene.ts` ให้ใช้ headless shell, หรือซ่อม `chrome.exe` ที่เปิดไม่ขึ้น, หรือเลิกใช้ `checkpoint` แล้วใช้ bridge อย่างเดียว
- [ ] รัน scene_diff ซ้ำหลังเพิ่ม `window.scene` ว่าเห็น boss ไหม
- [ ] ลบ `.playwright-mcp/` และเพิ่ม `.gitignore` สำหรับ `.parallax/`
- [ ] commit (แยก commit: `window.scene` hook กับ tooling ของ parallax)
