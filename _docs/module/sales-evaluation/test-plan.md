# ระบบประเมินและสนับสนุนพนักงานขาย (Sales Evaluation & Enablement) — Test Strategy & Test Plan

**Has automated test framework:** yes — xUnit (.NET 9) ใน `tests/` (`SalesEvaluation.Api.Tests`, `SalesEvaluation.Application.Tests`, `SalesEvaluation.Domain.Tests`, `SalesEvaluation.IntegrationTests`), Typecheck (TypeScript) และ Manual Inspection / Accessibility E2E ใน `frontend/`

---

## 1. Test Strategy Overview

เอกสารนี้กำหนดกลยุทธ์และระดับการทดสอบ (Test Levels) สำหรับโมดูล **sales-evaluation** เพื่อให้มั่นใจว่าการพัฒนาฟีเจอร์ใหม่ใน **Phase 21 (New Business Features & UX Accessibility)** สอดคล้องกับสัญญาใน `design.md` และข้อกำหนดใน `requirement.md` โดยยึดตามหลักเกณฑ์ **Test Pyramid**:

1. **Unit Tests (Domain / Application)**:
   - ทดสอบ Business Logic, การคำนวณสูตร, กฎการกรองสิทธิ์ (`resolveViewerScope`), Name Normalization, และ Validation rules แยกเดี่ยว ไม่ต่อฐานข้อมูลภายนอก
   - ใช้ xUnit + FluentAssertions ใน `tests/SalesEvaluation.Domain.Tests` และ `tests/SalesEvaluation.Application.Tests`
2. **Integration Tests (Infrastructure / Database)**:
   - ทดสอบ EF Core Mapping, PostgreSQL Constraints, Database Transactions, และ Repository queries
   - ยืนยันว่า Entity ทุกตัวใช้ Primary Key และ Foreign Key เป็น `Int` (int identity) ตามคำสั่งสถาปัตยกรรมของผู้ใช้
   - ใช้ xUnit + Test Database ใน `tests/SalesEvaluation.IntegrationTests`
3. **API / Contract Tests (Presentation / Endpoints)**:
   - ทดสอบ HTTP Status Codes, Request Validation, Response Shapes, Role-Based Access Control (RBAC), และ File Streaming
   - ป้องกันช่องโหว่ด้านความปลอดภัย (Excel formula injection, Broken Object Level Authorization)
   - ใช้ `WebApplicationFactory` ใน `tests/SalesEvaluation.Api.Tests`
4. **E2E & UI Accessibility Tests (Frontend)**:
   - ตรวจสอบความถูกต้องของ UI Components, Modals, State Management (Zustand), และ Responsive Design
   - ตรวจสอบมาตรฐานการเข้าถึงและการใช้งานสำหรับทุกช่วงวัย (WCAG 2.2 AA): ขนาดฟอนต์ $\ge 16\text{px}$, Touch target $\ge 44 \times 44\text{px}$, Contrast ratio $\ge 4.5:1$, Keyboard navigation & Focus ring

---

## 2. Phase 21 Test Requirements & Scenarios

### REQ-021-01 — การขยาย UserRole รองรับ SUPERVISOR และการแบ่งกลุ่มสิทธิ์ (DES-031 / BE-101, BE-102, FE-101)
- **Levels:** unit, integration, api, e2e
- **คำอธิบาย:** ขยายบทบาทผู้ใช้ให้รองรับ `SUPERVISOR` เพื่อเป็นหัวหน้าระดับเขต/สายงาน พร้อมปรับ `resolveViewerScope` ให้กรองข้อมูลตามเขตที่รับผิดชอบ

#### Test Cases:
1. **Unit (Domain/Application)**:
   - `UserRole` Enum แปลงค่าสตริง `"SUPERVISOR"` ถูกต้อง ไม่เกิด Deserialization error
   - `resolveViewerScope(user)` เมื่อผู้ใช้มี Role เป็น `SUPERVISOR` จะคืนค่า `ViewerScope.Supervisor` พร้อมรายการ `TerritoryIds` ที่ได้รับมอบหมาย
   - `resolveViewerScope(user)` เมื่อผู้ใช้เป็น `MANAGER` คืนค่า `ViewerScope.Manager` (เห็นทุกเขต) และ `SALESPERSON` คืนค่า `ViewerScope.Salesperson` (เห็นเฉพาะตนเอง)
2. **Integration (Database/EF Core)**:
   - บันทึก `User` ที่มีบทบาท `SUPERVISOR` ลงฐานข้อมูลด้วย EF Core สำเร็จ โดย PK `id: Int identity`
   - ค้นหาผู้ใช้ตามบทบาท (`WHERE Role = UserRole.Supervisor`) คืนค่าครบถ้วนถูกต้อง
3. **API (Endpoints & RBAC)**:
   - `GET /api/users`: คืนรายชื่อผู้ใช้พร้อมฟิลด์ `role` และสถานะการผูกพนักงานขาย (`isLinkedToSalesperson`)
   - `PUT /api/users/{id}/role`:
     - เมื่อเรียกโดย `MANAGER` สามารถเปลี่ยน Role เป็น `SUPERVISOR` ได้สำเร็จ (`200 OK`)
     - เมื่อเรียกโดยผู้ใช้ที่ไม่ได้เป็น `MANAGER` (เช่น `SUPERVISOR` หรือ `SALESPERSON`) ต้องถูกปฏิเสธด้วย `403 Forbidden`
4. **E2E & Frontend (UI / Usability)**:
   - หน้า `/users` แสดง Badge แสดงบทบาท 3 สีชัดเจน พร้อมข้อความและไอคอนกำกับ
   - แสดงกล่องแจ้งเตือน (Alert Banner) สีเหลือง สำหรับบัญชีผู้ใช้ที่ยังไม่ได้ผูกกับพนักงานขายในระบบ
   - การสลับสิทธิ์หรือแก้ไขผู้ใช้มีขนาดปุ่มไม่เล็กกว่า $44 \times 44\text{px}$

---

### REQ-021-02 — การตรวจสอบและคัดกรองชื่อพนักงานขายตอนนำเข้ายอดขาย Dry-Run & Alias Resolution (DES-032 / BE-103, BE-104, BE-105, FE-102, FE-103)
- **Levels:** unit, integration, api, e2e
- **คำอธิบาย:** ระบบนำเข้ายอดขายแบบ Dry-Run ตรวจหาชื่อพนักงานขายที่ไม่มีในระบบ และอนุญาตให้ผู้ใช้เลือก 3 ทางเลือก (`AUTO_CREATE`, `MAP_EXISTING`, `SKIP`) โดยบันทึก Alias และนำเข้าใน Transaction เดียว

#### Test Cases:
1. **Unit (Domain/Import Service)**:
   - ฟังก์ชันจับคู่ชื่อเทียบกับ `Salesperson`, `SalesmanAlias`, และ `SalesmanNameRule`:
     - ชื่อที่ตรงกัน (Exact Match หรือ Normalized Match) ถูกจัดว่าผ่าน
     - ชื่อที่ไม่ตรงและไม่มีกฎ ถูกรวบรวมเป็น `UnverifiedSalesmanItem` พร้อมนับจำนวนแถว
   - ตรวจสอบ Payload ของ `salesmanDecisions`:
     - `AUTO_CREATE`: สร้าง `Salesperson` ใหม่ (PK `Int`)
     - `MAP_EXISTING`: ต้องระบุ `targetSalespersonId: Int` ที่มีอยู่จริงในระบบ
     - `SKIP`: แถวที่มีชื่อดังกล่าวจะถูกข้าม ไม่นับเป็น Error และไม่ถูกบันทึก
2. **Integration (EF Core / Persistence & Transaction)**:
   - Entity `SalesmanAlias` บันทึกสำเร็จด้วย PK `id: Int identity`, `normalizedKey: string`, `salespersonId: Int`, `decidedById: Int?`
   - การ Execute Import ทั้งหมดทำงานใน Database Transaction เดียวกัน: หากแถวใดแถวหนึ่งเกิด Unhandled Exception ทุกการเปลี่ยนแปลง (รวมถึงการสร้าง Salesperson และ Alias ใหม่) ต้อง Rollback สมบูรณ์
3. **API (Endpoints)**:
   - `POST /api/sales/import/dry-run`:
     - ส่งไฟล์ Excel ที่มีชื่อที่ไม่รู้จัก -> คืนค่า `200 OK` พร้อม `hasUnverifiedSalesmen = true` และรายการชื่อที่ต้องตัดสินใจ โดย**ไม่มีการเขียนข้อมูลลงฐานข้อมูลจริง**
   - `POST /api/sales/import/execute`:
     - ส่งไฟล์พร้อมอาร์เรย์ `salesmanDecisions` -> นำเข้ายอดขายสำเร็จ คืนสรุปจำนวนแถว (`insertedRows`, `skippedRows`)
     - นำเข้าไฟล์ถัดไปที่มีชื่อเดิมซ้ำ -> ระบบจำค่าจาก `SalesmanAlias` ที่บันทึกไว้ได้อัตโนมัติ โดยไม่แจ้งเตือนซ้ำ
4. **E2E & Frontend (UI / Interaction)**:
   - Modal ตรวจสอบชื่อพนักงานขายแสดงขึ้นมาอัตโนมัติหลังกดอัปโหลดและ Dry-Run พบชื่อใหม่
   - ตัวเลือก 3 ปุ่ม (`เพิ่มเป็นพนักงานใหม่`, `จับคู่กับคนเดิม`, `ข้ามรายการนี้`) มีขนาดพื้นที่กด $\ge 44 \times 44\text{px}$
   - ฟิลด์ Autocomplete ค้นหาพนักงานขายเดิมตอบสนองรวดเร็ว มี Debounce 300ms
   - แสดง Modal ยืนยันสรุปภาพรวม (เช่น "จะเพิ่ม 1 คน, แมป 2 คน, ข้าม 5 แถว") ก่อนกดยืนยัน Execute จริง

---

### REQ-021-03 — การส่งออกรายงานรายบุคคลของทุกคนเป็น Multi-sheet Excel (DES-033 / BE-106, FE-104)
- **Levels:** unit, api, e2e
- **คำอธิบาย:** สร้างและดาวน์โหลดไฟล์ Excel รายงานรายบุคคลของพนักงานทุกคนรวมใน 1 ไฟล์ แยก Sheet ละคน ด้วย ClosedXML พร้อมคุมสิทธิ์ตาม `viewerScope`

#### Test Cases:
1. **Unit (Reporting / ClosedXML Service)**:
   - ClosedXML Workbook สร้าง Sheet ภาพรวม (Overview) 1 Sheet และ Sheet รายบุคคลตามรายชื่อพนักงานที่ได้รับสิทธิ์
   - การตั้งชื่อ Sheet: Sanitization อักขระพิเศษ (ห้ามมี `[]:*?/\\` และความยาวไม่เกิน 31 ตัวอักษร)
   - ป้องกัน Excel Formula Injection: ตรวจสอบเซลล์ข้อความ หากขึ้นต้นด้วย `=`, `+`, `-`, `@` ให้เติม single quote (`'`) ข้างหน้า
2. **API (Endpoint & Permission)**:
   - `GET /api/reports/individual/export-all`:
     - คืน Content-Type `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` พร้อม Header `Content-Disposition` ถูกต้อง
     - **สิทธิ์ MANAGER**: Workbook มี Sheet ครบทุกคนในองค์กร
     - **สิทธิ์ SUPERVISOR**: Workbook มีเฉพาะ Sheet ของพนักงานขายในเขตที่ตนเองดูแล
     - **สิทธิ์ SALESPERSON**: ได้รับ `403 Forbidden` หรือคืนเฉพาะ Sheet ของตนเองเท่านั้น
3. **E2E & Frontend (UI / Export Interaction)**:
   - ปุ่ม "Export รายงานพนักงานทุกคน (Excel)" บนหน้ารายงานภาพรวมทีม แสดงสถานะ Loading / Spinner ขณะระบบกำลังสร้างไฟล์
   - ปุ่มถูกซ่อนหรือ Disable สำหรับผู้ใช้ที่ไม่มีสิทธิ์ส่งออกรายงานของทีม
   - ตรวจสอบไฟล์ `.xlsx` ที่ดาวน์โหลดมา สามารถเปิดอ่านใน Microsoft Excel / LibreOffice ได้อย่างถูกต้อง ฟอร์แมตตัวเลขและตารางชัดเจน

---

### REQ-021-04 — มาตรฐาน UX Accessibility & All-Ages Usability (WCAG 2.2 AA) (DES-034 / FE-105, FE-106, FE-107)
- **Levels:** e2e, unit
- **คำอธิบาย:** ยกระดับประสบการณ์การใช้งานของระบบทั้งหมดให้รองรับผู้ใช้ทุกช่วงวัยตามมาตรฐาน WCAG 2.2 AA

#### Test Cases:
1. **Unit (Frontend Utilities)**:
   - ฟังก์ชันตรวจสอบ Contrast Ratio ให้ผลลัพธ์ $\ge 4.5:1$ สำหรับคู่สีพื้นหลังและตัวอักษรปกติ
   - Custom hook `useDebounce` ทำงานหน่วงเวลา 300ms ถูกต้อง ไม่เกิด Memory leak
2. **E2E & Visual Inspection (Accessibility Guidelines)**:
   - **Typography**: ตัวอักษรเนื้อหา (Body text) ทั้งระบบมีขนาด $\ge 16\text{px}$ (`text-base`), หัวข้อขนาดใหญ่อ่านง่าย
   - **Touch Targets & Spacing**: ทุกปุ่ม, ลิงก์, และฟิลด์กรอกข้อมูล มีขนาดพื้นที่คลิกไม่ต่ำกว่า $44 \times 44\text{px}$ พร้อมระยะห่าง (Margin/Gap) อย่างน้อย $8\text{px}$
   - **Color Independence**: ไม่มีจุดใดที่สื่อความหมายด้วยสีเพียงอย่างเดียว (เช่น สถานะ KPI หรือความผิดพลาด มีข้อความกำกับและไอคอนร่วมด้วยเสมอ)
   - **Keyboard Navigation & Focus**:
     - ปุ่มและอินพุตทั้งหมดมี Focus Ring ชัดเจน (`ring-2 ring-blue-500 ring-offset-2`) เมื่อกดปุ่ม Tab
     - Modal ทุกตัวรองรับการปิดด้วยปุ่ม `Escape` และกัก Focus ไม่ให้หลุดออกนอก Modal
   - **Action-Oriented Dashboard**:
     - KPI สรุปภาพรวมและกล่องแจ้งเตือนงานที่ต้องทำ (Actionable Items) แสดงผลในครึ่งบนของหน้าจอ (Above the fold) โดยไม่ต้องเลื่อนหน้าจอ

---

## 3. Test Execution Matrix

| Test ID | Requirement | Level | Test Project / Tool | Target Criteria |
|---|---|---|---|---|
| **TC-021-001** | REQ-021-01 (SUPERVISOR Enum & Scope) | Unit | `SalesEvaluation.Domain.Tests` | Logic คืนสิทธิ์และเขตถูกต้อง 100% |
| **TC-021-002** | REQ-021-01 (User Entity EF Core PK Int) | Integration | `SalesEvaluation.IntegrationTests` | บันทึกและดึงข้อมูลด้วย PK `Int identity` สำเร็จ |
| **TC-021-003** | REQ-021-01 (User Endpoints RBAC) | API | `SalesEvaluation.Api.Tests` | `MANAGER` ได้ 200, role อื่นได้ 403 ตามสิทธิ์ |
| **TC-021-004** | REQ-021-02 (Salesman Matching Logic) | Unit | `SalesEvaluation.Application.Tests` | แยกชื่อตรง/ชื่อใหม่ ถูกต้องตาม Rule |
| **TC-021-005** | REQ-021-02 (SalesmanAlias Persistence) | Integration | `SalesEvaluation.IntegrationTests` | บันทึก Alias สำเร็จ, Rollback transaction ได้ |
| **TC-021-006** | REQ-021-02 (Dry-run & Execute API) | API | `SalesEvaluation.Api.Tests` | Dry-run ไม่เซฟ, Execute บันทึกครบถ้วน |
| **TC-021-007** | REQ-021-02 (Dry-run Modal Interaction) | E2E | Frontend Inspection / Cypress | Target $\ge 44\text{px}$, Autocomplete ลื่นไหล |
| **TC-021-008** | REQ-021-03 (ClosedXML Multi-sheet Stream) | Unit | `SalesEvaluation.Application.Tests` | โครงสร้าง Sheets ถูกต้อง, ป้องกัน Formula Injection |
| **TC-021-009** | REQ-021-03 (Export API Permission Scope) | API | `SalesEvaluation.Api.Tests` | สตรีมไฟล์ `.xlsx` ได้, กรองข้อมูลตามเขตของ User |
| **TC-021-010** | REQ-021-04 (WCAG 2.2 AA Contrast & Size) | Visual/E2E | Lighthouse / Axe Core / Inspection | Contrast $\ge 4.5:1$, Font $\ge 16\text{px}$, Touch $\ge 44\text{px}$ |
| **TC-021-011** | REQ-021-04 (Keyboard Nav & Focus Ring) | E2E | Manual Tab Inspection | เข้าถึงทุกฟิลด์ได้ด้วยคีย์บอร์ด, Focus Ring เด่นชัด |

---

## 4. Unresolved Open Questions

1. **การจำลองข้อมูลทดสอบ (Test Data Seeding) สำหรับ Multi-sheet Export**:
   - ในการทดสอบ Unit/Integration ของ ClosedXML จะใช้ In-memory Mock Data ชุดเล็ก (3 พนักงาน, คนละ 5 รายการขาย) เพื่อให้การรันเทสเสร็จสิ้นรวดเร็ว ไม่กระทบต่อความเร็วของ CI Pipeline

---

## Change Log

- 2026-09-11 — **initial (test-planner, ตาม requirement.md, design.md, plan.md รอบ 2026-09-11)**: จัดทำกลยุทธ์การทดสอบ (Test Plan) ฉบับแรกสำหรับ **Phase 21: New Business Features & UX Accessibility** ครอบคลุม 4 Requirements หลัก (`REQ-021-01` ถึง `REQ-021-04`), 11 Test Cases (`TC-021-001` ถึง `TC-021-011`) ในระดับ Unit, Integration, API, และ E2E Accessibility ตามมาตรฐาน Test Pyramid และ WCAG 2.2 AA
