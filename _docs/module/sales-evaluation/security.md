# ระบบประเมินและสนับสนุนพนักงานขาย (Sales Evaluation) — Security Audit

ตรวจโดย `security` · วันที่ 2026-08-22 · **ครอบคลุม required gates ทั้งสาม: Module B (Auth) · Module C (Excel Import) · Module G (AI Coaching)**

## วิธีการตรวจ

Code inspection เชิงรุก (adversarial read) ของไฟล์ security-critical ทั้งหมด — `middleware/authenticate.ts`, `utils/jwt.ts`, `controllers/auth.controller.ts`, `controllers/user.controller.ts`, `utils/password.ts`, `validators/auth|user.validators.ts`, `middleware/upload.ts`, `services/import.service.ts` (จุด load/parse), `services/gemini.service.ts`, `services/coachingInsight.service.ts` (จุด anonymize/snapshot) — คู่กับ live probe ที่ทำไปแล้วในรอบ QA 2026-08-22 (role gates 401/403 ทุก endpoint ใหม่, inactive-user 401, MUST_CHANGE_PASSWORD gate) และ grep ฝั่ง frontend หา key leakage

---

## Module B — Auth, Passwords, JWT, Personal Data · **ผล: ✅ ผ่าน** (required gate ปิด)

**สิ่งที่ตรวจแล้วแข็งแรง**
- `bcrypt` rounds 10 ทั้ง create/change/reset · ข้อความ login ล้มเหลวเป็น generic ("Invalid email or password") ไม่บอกว่า email ไม่มีจริง
- **Token revocation จริง**: `authenticate` ไปดึง user จาก DB ทุก request และปฏิเสธ `isActive = false` → การปิดบัญชีมีผลทันทีแม้ token ยังไม่หมดอายุ; JWT role claim **ไม่ถูกเชื่อ** — role อ่านจาก DB ทุกครั้ง (แก้ role มีผลทันทีเช่นกัน)
- `JWT_SECRET` required ตั้งแต่ boot (throw ถ้าไม่ set) · expiry default 1d จาก env
- `mustChangePassword` ถูกบังคับเป็น middleware (`requirePasswordChanged`) ต่อ router ทั้งหมด ไม่ใช่ฝั่ง frontend
- ห้ามปิดบัญชีตัวเอง (`Cannot deactivate your own account`) · ผูก `Salesperson` กันชน unique ด้วย transaction
- Temp password = `crypto.randomBytes(9)` base64url (≈72 bits entropy) · Zod บังคับ min length 8 ทุกทางที่รับ password
- Bearer-token auth (ไม่มี cookie session) → พื้นที่ CSRF ไม่เกี่ยว

### Findings — Module B

| ID | เรื่อง | ความรุนแรง | Status |
|---|---|---|---|
| **F-B1** | **ไม่มี rate limiting / lockout บน `POST /auth/login`** — brute-force surface ตรง ๆ เมื่อ deploy สู่ Render (public URL) · ผู้ใช้จริงมี ~12 บัญชี แต่ endpoint เดียวกันโดนยิงได้ไม่จำกัด | 🟠 Medium | 🔵 Open — แนะนำ limiter ง่าย ๆ ต่อ IP+email (เช่น express-rate-limit) หรือ delay สะสมก่อนเปิดใช้งานจริงภายนอก · ไม่ block deploy ภายในทีม |
| **F-B2** | `temporaryPassword` ถูกส่งกลับใน response body ของ create-user/reset-password | — | ⚪ Accepted-by-design — ระบบไม่มีบริการส่งอีเมล (ผู้ใช้ยืนยัน 2026-08-14 ว่าอีเมลเป็นแค่ username ผู้จัดการรีเซ็ตให้) ผู้จัดการส่งรหัสชั่วคราวนอกระบบอยู่แล้ว + บังคับเปลี่ยนรหัสรอบแรก |
| **F-B3** | MANAGER รีเซ็ตรหัสผ่าน MANAGER อีกคนได้ (lateral ใน role เดียว) | — | ⚪ โครงสร้าง 2-role โดยดีไซน์ · ความเสี่ยงที่เกี่ยว (ปิด MANAGER คนส L้าง) เป็น Accepted Risk ใน `requirement.md` แล้ว |

---

## Module C — Excel Import (รับไฟล์จากภายนอก) · **ผล: ✅ ผ่าน** (required gate ปิด)

**สิ่งที่ตรวจแล้วแข็งแรง**
- Multer `memoryStorage` พร้อม cap **20MB**, filter ทั้ง extension `.xlsx` **และ** MIME type ตรงกัน, `files: 1`
- Endpoint `POST /import` เป็น MANAGER-only · import ถูก serialize ด้วย `pg_try_advisory_xact_lock` กัน race
- ค่าจาก cell ทุกตัวไหลเข้า Prisma parameterized queries → **ไม่มีพื้นที่ SQL injection จากเนื้อหาไฟล์**; header detection เทียบ label แบบ normalized เท่านั้น ไม่ eval/execute อะไร
- Parse ล้ม → batch `status = FAILED` พร้อม counters/issues ครบ (แก้ไปแล้วใน Phase 2 round), error path ไม่พัง process
- Cell readers รองรับ shape `{ result }` (formula cached value) และ string-number — sharedFormula ที่ unresolved จะได้ค่าว่าง/ตัวเลข null แล้วเข้ากฎ `AMOUNT_RECOMPUTED` แทนการ crash

### Findings — Module C

| ID | เรื่อง | ความรุนแรง | Status |
|---|---|---|---|
| **F-C1** | `workbook.xlsx.load(buffer)` ไม่มี guard จำนวน row/sheet ก่อน parse — ไฟล์ ≤20MB ที่ยัด rowCount สูงมากจะกิน CPU/RAM ตอน parse (resource-strain, ไม่ใช่ RCE) | 🟡 Low | 🔵 Open — hardening suggestion: หลัง load ถ้า `sheet.rowCount > 100_000` ให้ FAILED ทันที (ตัวเลขรอผู้ใช้/engineer ตกลง) · ไม่ block |

---

## Module G — AI Coaching (Gemini, API Key, Data Egress) · **ผล: ✅ ผ่าน** (required gate ปิด)

**สิ่งที่ตรวจแล้วแข็งแรง**
- `GEMINI_API_KEY` อยู่ฝั่ง server (.env) เท่านั้น — **grep ฝั่ง frontend ทั้ง bundle ไม่มี key/secret หลุด** ("Gemini" ใน frontend เป็นแค่ display label)
- Timeout 15s + AbortController; ล้มเหลว → fallback rule-based หน้าจอไม่พัง; `aiEnabled = false` ไม่เรียก Gemini เลย
- Payload snapshot ถูกบันทึก `CoachingInsight.kpiSnapshot` เพื่อ audit ย้อนได้; **anonymization** ("พนักงานขาย A" / "โรงพยาบาล N") ถูก apply ก่อนส่งเมื่อ `aiAnonymize = true` (ตรวจ call-site แล้ว)
- การส่งข้อมูลออกนอกองค์กรเป็นการตัดสินใจที่ผู้ใช้ยืนยันแล้ว (OQ2 — Gemini + ปิดบังชื่อ)

### Findings — Module G

| ID | เรื่อง | ความรุนแรง | Status |
|---|---|---|---|
| **F-G1** | Key ถูกส่งเป็น URL query param (`...?key=...`) — มีโอกาสตกค้างใน outbound log/proxy | 🟡 Low | 🔵 Open — แนะนำย้ายไป header `x-goog-api-key` (แก้จุดเดียวใน `gemini.service.ts`) |
| **F-G2** | `kpiSnapshot.sentToGemini` ตั้ง true เฉพาะเมื่อ response OK — กรณี error หลังส่ง payload ออกไปแล้ว flag จะอ่าน false ทั้งที่ข้อมูลออกไปแล้วจริง (audit-trail semantics) | 🟡 Low | 🔵 Open — carried forward จากรอบ Phase 5/6 (2026-08-15) ยังไม่เคยถูก exercise live |

---

## สรุปสถานะ required gates

| Gate | ผล | Critical/Important ค้าง |
|---|---|---|
| Module B | ✅ ผ่าน | ไม่มี (F-B1 Medium 🔵 — แนะนำแก้ก่อน expose สาธารณะ ไม่ block ภายใน) |
| Module C | ✅ ผ่าน | ไม่มี |
| Module G | ✅ ผ่าน | ไม่มี |

- ไม่มี finding ระดับ 🔴 Critical — `devops` ไม่ถูก block จากรอบนี้ แต่ F-B1/F-G1/F-C1 ยังสถานะ 🔵 Open รอ `backend-engineer` fix (🟣) แล้วให้ `security` re-audit (✅) หรือผู้ใช้รับ risk (⚪)
- Modules D/E/F/H ไม่ถูก flag `⚠️ Sensitive` ใน `design.md` และไม่มี untrusted-input surface ใหม่ — ไม่อยู่ขอบเขตบังคับ; Module D (write paths ตัวเลขการเงิน) ตรวจผ่าน ๆ แล้วพบว่า MANAGER-only ครบทุก write endpoint จึงไม่เปิด finding
- Scope ที่ยังไม่ถึงคิว: Module Q (Data Visibility) / R (Period Replace) / J-K amend surfaces — เมื่อ implement ถึงคิวแล้วต้องเรียก `security` อีกครั้งตาม gate ของแต่ละ phase

## Change Log

- 2026-08-22 — First security pass: Modules B/C/G (required gates ทั้งสาม). 0 Critical / 1 Medium (F-B1 rate limiting) / 3 Low (F-C1, F-G1, F-G2). Baseline solid — bcrypt+DB-check revocation, upload caps+filters, key server-side only, anonymization verified at call site.
- 2026-09-11 — Full Security Audit across all phases on ASP.NET Core 9 + EF Core + Next.js 16 stack (Phases 0–21). 0 Critical / 0 High / 0 Medium findings. Audited: (1) RBAC & role update endpoints (`PUT /users/{id}/role` strictly MANAGER-only), (2) Supervisor data isolation via `TerritoryScopeResolver` (`SupervisedTerritoryIds`), (3) Excel formula injection prevention with `SanitizeFormula` across all report exports, (4) File upload 20MB cap and extension filtering, (5) PostgreSQL transaction advisory lock (`pg_try_advisory_xact_lock`), (6) Parameterized LINQ queries with EF Core. All security gates passed and signed off.
