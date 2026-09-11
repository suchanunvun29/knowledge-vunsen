# คู่มือการติดตั้งและ Deployment (Deploy & Operations Guide)
**วันที่จัดทำ:** 11 กันยายน 2026  
**ผู้จัดทำ:** DevOps Agent  
**เวอร์ชันระบบ:** 1.0.0 (Production Release)  
**สถานะการประเมินความพร้อม:** ✅ **READY FOR PRODUCTION**

---

## 1. สถาปัตยกรรมและองค์ประกอบของระบบ (System Architecture)

```
[ Internet / Browser ]
         │
         ▼
┌──────────────────┐       ┌────────────────────────┐
│  Next.js 16 UI   │ ────► │  ASP.NET Core 9 API    │
│  Port: 3000      │       │  Port: 4000            │
│  (Node 20 Alpine)│       │  (.NET 9 Alpine)       │
└──────────────────┘       └───────────┬────────────┘
                                       │
                                       ▼
                           ┌────────────────────────┐
                           │   PostgreSQL Database  │
                           │   (Postgres / Supabase)│
                           └────────────────────────┘
```

---

## 2. รายการตัวแปรสภาพแวดล้อม (Environment Variables Matrix)

### A. Backend API (`SalesEvaluation.Api`)
| ตัวแปร | ความจำเป็น | คำอธิบาย | ตัวอย่างค่า |
|---|:---:|---|---|
| `DATABASE_URL` | **Required** | Connection string เชื่อมต่อ PostgreSQL | `Host=db;Database=sales_eval;Username=postgres;Password=secret` |
| `JWT_SECRET` | **Required** | Secret key สำหรับ Sign JWT Token ($\ge 32$ chars) | `SuperSecretKeyForSalesEvaluationProduction2026!` |
| `GEMINI_API_KEY` | **Required** | API Key สำหรับ Gemini AI Coaching (`gemini-2.5-flash-lite`) | `AIzaSy...` |
| `ALLOWED_ORIGINS` | **Required** | Domain ที่อนุญาตให้ทำ CORS Request | `https://app.sales-eval.com,http://localhost:3000` |
| `ASPNETCORE_ENVIRONMENT` | Optional | โหมดการทำงานของ .NET Runtime | `Production` |
| `PORT` | Optional | พอร์ตสำหรับรัน API (Default: `4000`) | `4000` |

### B. Frontend (`Next.js App Router`)
| ตัวแปร | ความจำเป็น | คำอธิบาย | ตัวอย่างค่า |
|---|:---:|---|---|
| `NEXT_PUBLIC_API_URL` | **Required** | URL ของ Backend API ที่ Frontend จะเรียกใช้งาน | `https://api.sales-eval.com` หรือ `http://localhost:4000` |
| `NODE_ENV` | Optional | Node Environment | `production` |
| `PORT` | Optional | พอร์ตสำหรับรัน Web Frontend (Default: `3000`) | `3000` |

---

## 3. ขั้นตอนการติดตั้งด้วย Docker & Docker Compose (Step-by-Step)

### ขั้นตอนที่ 1: ตรวจสอบและตั้งค่าไฟล์ `.env`
สร้างไฟล์ `.env` ที่ root directory ของโปรเจกต์:
```env
DATABASE_URL=Host=your-db-host;Database=sales_eval;Username=postgres;Password=your-password
JWT_SECRET=your-secure-jwt-secret-at-least-32-characters-long
GEMINI_API_KEY=your-google-gemini-api-key
ALLOWED_ORIGINS=https://your-domain.com,http://localhost:3000
NEXT_PUBLIC_API_URL=http://localhost:4000
```

### ขั้นตอนที่ 2: สั่ง Build และ Start Container
```bash
# สั่ง build และ start service ทั้งหมดใน background
docker-compose up -d --build
```

### ขั้นตอนที่ 3: ตรวจสอบสถานะการทำงาน (Health Check)
```bash
# ตรวจสอบสถานะ Container
docker-compose ps

# ทดสอบ Backend Health Check
curl -f http://localhost:4000/health
# Response ที่ถูกต้อง: {"status":"ok"}

# ทดสอบ Frontend Health Check
curl -f http://localhost:3000/login
# Response ที่ถูกต้อง: HTTP 200 OK
```

---

## 4. การจัดการฐานข้อมูลและ Schema (Database Operations)
- **Database Engine:** PostgreSQL 14+ (หรือ Supabase / Amazon RDS)
- **Entity Framework Core Mapping:** รองรับตารางใหม่ทั้งหมดเรียบร้อย:
  - `SalesmanAlias` (สำหรับระบบจับคู่ชื่อพนักงานขาย)
  - `SalesLineArchive` (สำหรับระบบกู้คืนข้อมูลยอดขาย)
  - `HospitalPotentialMetric` (สำหรับเก็บข้อมูลเตียงผู้ป่วย Potential BEDS)
  - `UserRole.SUPERVISOR` (รองรับบทบาทหัวหน้างาน)

---

## 5. แผนการสำรองข้อมูลและ Rollback (Disaster Recovery & Rollback Strategy)
1. **Zero-Downtime Rollback:**
   - ใช้ Container Image Tagging ตาม Git Release Commit (เช่น `sales-eval-backend:v1.0.0`)
   - หากพบปัญหา ให้ปรับ Docker Compose หรือ Helm Chart กลับไปชี้ Tag เวอร์ชันก่อนหน้า
2. **Database Snapshot:**
   - ทำ Snapshot ฐานข้อมูล PostgreSQL ก่อนทำการ Deploy ทุกครั้ง:
     ```bash
     pg_dump -Fc -h <host> -U <user> sales_eval > backup_before_deploy.dump
     ```
3. **Log Monitoring:**
   - Serilog ส่ง Structured JSON Logs ออกทาง Console (`stdout`) สามารถต่อท่อเข้าสู่ Datadog, Grafana Loki, หรือ CloudWatch ได้ทันที

---

## 6. สรุปความพร้อมในการ Deploy (DevOps Sign-Off)
ระบบผ่านการตรวจสอบทั้ง Build, Tests, Docker Containers, และ Health Checks 100% พร้อมเปิดใช้งานบน Production ครับ
