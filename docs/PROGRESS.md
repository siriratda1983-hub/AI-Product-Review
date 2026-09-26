# PROGRESS: สถานะงานล่าสุด

> Agent ทุกตัว: อ่าน **"สถานะตอนนี้"** และ **"กระดานงาน"** ก่อนเริ่ม ใส่ชื่อตัวเองในช่อง "ใครถือ" ก่อนลงมือ และ**เพิ่มบันทึกใน Log ทุกครั้งที่จบงาน** (ใหม่สุดอยู่บน)

## สถานะตอนนี้

- มีเอกสารตั้งต้นครบ: `AGENTS.md`, `docs/ARCHITECTURE.md`, `docs/REUSE_MAP.md`, สเปกต้นฉบับใน `docs/spec/`
- **ยังไม่มีโค้ด**
- repo นี้จะย้ายไปอยู่ใน org `Telesale-Team` (รอสิทธิ์)

## คำถามที่ยังรอ Producer

1. ทีม Agent เดิมมีกี่ตัว เป็น AI อะไรบ้าง (จะใช้แบ่งช่อง "ใครถือ")
2. RTX 3060 เป็นรุ่น 12GB (เดสก์ท็อป) หรือ 6GB (โน้ตบุ๊ก)
3. RunPod: Serverless หรือ Pod และมี workflow ComfyUI สำหรับรีวิวสินค้าแล้วหรือยัง
4. วิธีดึงโค้ดเดิม (`docs/REUSE_MAP.md` ท้ายไฟล์) — แนะนำ "คัดลอกเฉพาะที่ใช้"

## กระดานงาน

| # | งาน | Phase | ใครถือ | สถานะ |
|---|---|---|---|---|
| T1 | `docs/CONTRACTS.md`: schema ฐานข้อมูล, API `/v1`, interface ของ adapter (**สัญญากลาง ต้องเสร็จก่อน T2–T6**) | 1 | Claude (session นี้) | ยังไม่เริ่ม |
| T2 | `docker-compose.yml` (postgres, redis, api, workers) + `.env.example` ที่รันบน Windows 11 + Docker Desktop | 1 | ว่าง | ยังไม่เริ่ม |
| T3 | migration ฐานข้อมูล + state machine ของ job/stage | 1 | ว่าง | ยังไม่เริ่ม |
| T4 | API ขั้นต่ำ (สร้างงาน, ดูสถานะ, ดู stage, cancel) | 1 | ว่าง | ยังไม่เริ่ม |
| T5 | Celery routing ตามคิว + worker registry/heartbeat + stage จำลอง (stub) ครบ 12 stage | 1 | ว่าง | ยังไม่เริ่ม |
| T6 | scheduler: fairness, admission control, lease, retry/backoff, idempotency, dead-letter | 2 | ว่าง | ยังไม่เริ่ม |
| T7 | ชุดทดสอบ Test A–G (สเปกหัวข้อ 17) | 2 | ว่าง | ยังไม่เริ่ม |
| T8 | ComfyUIAdapter (`local` / `runpod`) + workflow ทดสอบบน 3060 | 3 | ว่าง (เหมาะกับทีมเดิม) | ยังไม่เริ่ม |
| T9 | Web UI + AI แชทรับออเดอร์ เรียก API `/v1` (ดัดแปลงจาก ai-animation-studio) | 3 | ว่าง (เหมาะกับทีมเดิม) | รอ T1 |

## Log

### 2026-09-26 · Claude Code (session: adoring-galileo)
- อ่านสเปก v0.1 และสำรวจ ai-animation-studio (commit `294d3f8`)
- สร้าง `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `docs/ARCHITECTURE.md`, `docs/REUSE_MAP.md`, `docs/PROGRESS.md`
- ข้อค้นพบสำคัญ: `jobs.py` = คิวเดียว worker เดียว (ต้องเขียนใหม่); `render.py` วาดเฟรม Stickman กับ ffmpeg ในขั้นเดียว แยก compose ตรง ๆ ไม่ได้; `audio.py` / `llm.py` / `costs.py` / `ffmpeg_exe()` ใช้ต่อได้ผ่าน adapter
- งานถัดไป: T1 (สัญญากลาง)
