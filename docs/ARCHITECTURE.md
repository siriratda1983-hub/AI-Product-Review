# ARCHITECTURE: สถาปัตยกรรมระบบ AI Product Review

> ที่มา: `docs/spec/Build_Spec_v0.1.docx` + สิ่งที่ Producer ตอบเพิ่มเติม (2026-09-26)
> ถ้าเอกสารนี้ขัดกับสเปก ให้ยึดเอกสารนี้ (มันคือสเปกที่ปรับตามสภาพจริงแล้ว) และบันทึกเหตุผลใน Decision Log

## 1. ภาพรวม

```
ผู้ใช้ (เว็บ / มือถือ)
   │  https
   ▼
Cloudflare Tunnel
   │
   ▼  เครื่อง local prod ที่บ้าน (Docker Compose)
┌──────────────────────────────────────────────────────────────┐
│  Web UI + AI แชท (ดัดแปลงจาก ai-animation-studio)          │
│        │ เรียกผ่าน HTTP เท่านั้น                              │
│        ▼                                                     │
│  API (/v1/...)  ──►  Orchestrator  ──►  PostgreSQL (ความจริง) │
│                           │                                  │
│                    Scheduler / Admission                     │
│                           │                                  │
│                         Redis (broker)                       │
│   ┌──────────┬──────────┬─┴────────┬──────────┬──────────┐   │
│   research   script     asset_cpu  audio_api  compose_cpu …   │
│   workers    workers    workers    workers    workers        │
│                                                              │
│   asset_gpu / video_gpu workers ──► ComfyUIAdapter ─────────┼──┐
└──────────────────────────────────────────────────────────────┘  │
                                                                  ▼
                               dev:  ComfyUI บน Windows (RTX 3060)
                               prod: ComfyUI บน RunPod
```

- **Web UI / AI แชท ไม่รู้จักคิวหรือ RunPod** มันแค่เรียก API แล้วแสดงผล
- **worker GPU ในเครื่องไม่ได้ใช้ GPU เอง** มันส่ง workflow ไป ComfyUI แล้วรอผล จำนวนงาน GPU ที่รันพร้อมกันถูกจำกัดด้วย slot ของ backend (3060 = 1 slot)
- **MCP (ภายหลัง, ไม่บังคับ):** ห่อ API ชุดเดียวกันให้ Claude สั่งงานได้โดยตรง

## 2. บทบาทของ AI แชท (พนักงานรับออเดอร์)

สเปก v0.1 ไม่ได้กำหนดแชทบอทไว้ จึงกำหนดดังนี้:

1. คุยเพื่อเก็บข้อมูลสินค้า (= stage 1 Product Intake): ชื่อสินค้า รูป/วิดีโอ ลิงก์ จุดเด่น ความยาว ภาษา แพลตฟอร์ม
2. โชว์การ์ดสรุป + ค่าใช้จ่ายประมาณการ **AI ห้ามเริ่มงานที่เสียเงินเอง** ผู้ใช้ต้องกดยืนยัน (หลักการเดียวกับ `agent.py` เดิม)
3. ยืนยันแล้วเรียก `POST /v1/product-review/jobs` → ได้ `job_id`
4. รายงานสถานะจาก `GET /v1/jobs/{id}` (เช่น "รอคิว GPU ลำดับที่ 3") และส่งคำขอแก้ผ่าน `/revise`

แชทบอทอยู่ **หน้า** ระบบคิว ไม่ได้อยู่ในคิว ถ้าแชทบอทพังหรือเปลี่ยนตัว ระบบผลิตยังทำงานได้

## 3. เทคโนโลยี

| ส่วน | ใช้ | หมายเหตุ |
|---|---|---|
| ภาษา | Python 3.12 | |
| API | FastAPI | แยกจาก Django UI เดิม |
| สถานะงาน | PostgreSQL 16 | `SELECT … FOR UPDATE SKIP LOCKED` สำหรับ claim |
| Broker | Redis 7 | |
| Worker | Celery 5.6 | รันใน Linux container (Celery ไม่รองรับ Windows) |
| ประกอบคลิป | FFmpeg | |
| ไฟล์ | StorageAdapter: dev = โฟลเดอร์ local / prod = S3-compatible | อ้างอิงด้วย `artifact_id` |
| สร้างภาพ/วิดีโอ | ComfyUIAdapter: `local` / `runpod` | สลับด้วย `COMFY_BACKEND` ใน `.env` |
| เปิดสู่ภายนอก | Cloudflare Tunnel | |

## 4. Stage และคิว

| # | Stage | คิว |
|---|---|---|
| 1 | product_intake | qc_cpu |
| 2 | product_research | research_api |
| 3 | review_script | script_api |
| 4 | storyboard | script_api |
| 5 | asset_resolution | asset_cpu |
| 6 | visual_generation (ต่อ shot) | asset_gpu |
| 7 | voice_audio | audio_api |
| 8 | video_generation (ต่อ shot) | video_gpu |
| 9 | compose | compose_cpu |
| 10 | qc | qc_cpu |
| 11 | preview_approval | (รอผู้ใช้ ไม่ใช้ worker) |
| 12 | publish (optional) | publish_api |

State machine ของ job และ stage, field ขั้นต่ำของ stage และตารางฐานข้อมูล: ตามสเปกหัวข้อ 5 และ 12 (จะเขียนเป็นโค้ดใน `docs/CONTRACTS.md` + migration ใน Phase 1)

## 5. สภาพแวดล้อม

| | dev (Windows 11 ของ Producer) | prod (local prod ที่บ้าน) |
|---|---|---|
| รัน | Docker Desktop + WSL2 | Docker Compose |
| ComfyUI | Windows ตรง ๆ (ComfyUI Portable) `http://host.docker.internal:8188` | RunPod |
| GPU slot | 1 (RTX 3060) | ตามจำนวน RunPod worker |
| เข้าจากภายนอก | ไม่จำเป็น | Cloudflare Tunnel |

## 6. ลำดับการสร้าง (MVP)

1. **รอบที่ 1 (Phase 1–2):** ระบบคิวครบด้วย stage จำลอง (stub, ไม่เสียเงิน) จนผ่าน Test A–G ในสเปกหัวข้อ 17
2. **รอบที่ 2 (Phase 3–4):** เปลี่ยน stub เป็นของจริงทีละ stage (LLM, TTS, ComfyUI, FFmpeg)
3. **MVP เสร็จ** = ข้อมูลสินค้า 1 ชิ้น → preview MP4 จริงครบทุก stage + ผ่าน Test A–G

## 7. การตัดสินใจที่รับมาจาก ai-animation-studio (MASTER_PLAN v1.5)

| อ้างอิง | สิ่งที่ใช้ต่อในระบบนี้ |
|---|---|
| D6 / D19 | เครื่อง Producer = RTX 3060 **12GB** มี ComfyUI + SDXL อยู่แล้ว (API `127.0.0.1:8188`) → ใช้เป็น backend `local` ตอน dev |
| D13 | **ห้ามฝังค่าตั้งค่าในโค้ด** ค่าที่ Producer อาจเปลี่ยนต้องแก้ได้จากหน้าเว็บ ความลับอยู่ใน `.env` |
| D17 | เปิดให้ลูกค้าภายนอกใช้ → แยกข้อมูลตามผู้ใช้/ทีม (`tenant_id`) ตั้งแต่ schema แรก |
| D18 | **มีเพดานงบค่า API เสมอ** หยุดเรียกก่อนเกินงบ → ใน Scheduler เป็น budget guard ต่อผู้ใช้และทั้งระบบ |
| D27 | AI คุยตอบเป็น JSON ที่โปรแกรมตรวจก่อนทำ และห้ามเริ่มงานเสียเงินเองโดยไม่ผ่านการ์ดสรุป + ค่าใช้จ่าย |
| D29 | LLM สลับโมเดลเองเมื่อโมเดลแรกล่ม ไม่ฝังชื่อโมเดลในโค้ด |
| MASTER_PLAN §14 | ทำงานกับ AI หลายเจ้าโดยให้ความจำอยู่ใน repo (`AGENTS.md` + `PROGRESS.md`) |
| D11 | Blueprint 60 agent ใช้เป็น "แผนที่หน้าที่" ไม่ได้สร้างเป็น agent จริง 60 ตัว → ในระบบนี้แต่ละหน้าที่คือ stage + worker |

## 8. Decision Log

| # | วันที่ | การตัดสินใจ | เหตุผล |
|---|---|---|---|
| A1 | 2026-09-26 | repo ใหม่แยกจาก ai-animation-studio | ไม่เสี่ยงทำระบบ Stickman พัง (สเปกหัวข้อ 14) |
| A2 | 2026-09-26 | ทุก service รันใน Docker, ComfyUI dev รันบน Windows | Celery ต้องใช้ Linux; GPU ใช้บน Windows ตรง ๆ ง่ายกว่า |
| A3 | 2026-09-26 | ComfyUIAdapter ตัวเดียว สองปลายทาง (local / runpod) | dev ใช้ 3060 ฟรี, prod ใช้ RunPod โดยไม่แก้โค้ด |
| A4 | 2026-09-26 | AI แชทเป็นผู้รับออเดอร์ เรียก API เท่านั้น | แยก UI ออกจากระบบผลิต; AI ไม่เริ่มงานเสียเงินเอง |
| A5 | 2026-09-26 | API ใหม่ใช้ FastAPI | async, แยกจาก Django UI เดิม — **ยังเปลี่ยนได้ ถ้า Producer/ทีมต้องการ Django** |
