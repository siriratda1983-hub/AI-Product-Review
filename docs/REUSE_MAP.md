# REUSE_MAP: ใช้อะไรจาก ai-animation-studio ได้บ้าง

> สำรวจจาก [Telesale-Team/ai-animation-studio](https://github.com/Telesale-Team/ai-animation-studio) commit `294d3f8` (v1.16.7)
> ยังเป็น **ร่างจากการอ่านโค้ด** ยังไม่ได้ทดสอบจริง ก่อนใช้โมดูลใดให้ยืนยันด้วยการรัน (สเปกหัวข้อ 19 ข้อ 10)

- **reuse**: เรียกใช้ได้เกือบตรง ๆ ผ่าน adapter บาง ๆ
- **wrap**: ใช้แกนเดิม แต่ต้องห่อ/ตัดส่วนที่ผูกกับ Stickman หรือไฟล์ JSON ออก
- **rewrite**: แนวคิดใช้ได้ แต่ต้องเขียนใหม่
- **do-not-use**: ผูกกับ Stickman หรือเป็นคอขวดที่สเปกห้าม

| ไฟล์ | สถานะ | เหตุผล / สิ่งที่ต้องทำ |
|---|---|---|
| `studio/jobs.py` | **do-not-use** | คิวเดียวในหน่วยความจำ + worker thread เดียว + ประวัติเป็น `output/jobs.json` = คอขวดที่สเปกห้าม |
| `studio/pipeline.py` | rewrite | ผูกกับ `episode.json` / genre / cast; ใช้แนวคิด "ถาม AI จนได้บทที่ผ่านการตรวจ" (`_ask_until_valid`) และ version ของบทได้ |
| `studio/render.py` | **wrap บางส่วน** | `render_episode` วาดเฟรม Stickman แล้ว pipe เข้า ffmpeg ในขั้นเดียว แยก compose ตรง ๆ ไม่ได้ ใช้ได้เฉพาะ `wrap` / `_words` / `_clusters` (ตัดบรรทัดซับไตเติลภาษาไทย) และวิธี pipe rawvideo เข้า ffmpeg เป็นตัวอย่าง |
| `studio/config.py` → `ffmpeg_exe()` | reuse | หา ffmpeg ที่มี libx264 |
| `studio/audio.py` | wrap | TTS (Edge / Gemini / ElevenLabs) + cache + `mix_into` / `write_wav` ใช้ได้; `voice_for` ผูกกับตัวละคร ต้องเปลี่ยนเป็น voice profile ของ presenter |
| `studio/llm.py` | wrap | `complete()` หลายเจ้า + สลับรุ่นอัตโนมัติ + `extract_json` ใช้ได้; ต้องต่อ rate limit/semaphore ต่อ provider ตามสเปกหัวข้อ 13 |
| `studio/costs.py` | wrap | ledger + เพดานงบ ใช้แนวคิดได้ แต่ต้องย้ายที่เก็บจากไฟล์เป็นตาราง `provider_usage` และแยกตาม user |
| `studio/publish.py` | wrap (Phase หลัง) | YouTube adapter ใช้ได้; ต้องเพิ่ม idempotency key และย้ายสถานะเข้า PostgreSQL |
| `studio/settings.py` | rewrite | settings อยู่ในไฟล์ JSON; ระบบใหม่ใช้ `.env` + ตาราง config |
| `studio/agent.py`, `studio/chat.py`, `studio/prompts/agent_th.md` | wrap (ทีม UI) | แนวคิด "AI ตอบ JSON {say, choices, action}, ห้ามเริ่มงานเสียเงินเอง" ใช้ต่อได้; เปลี่ยน action ให้เรียก API ใหม่ และเก็บข้อมูลสินค้าแทนหัวข้อคลิป |
| `studio/chats.py` | wrap (ทีม UI) | เก็บประวัติแชตต่อ user |
| `studio/web/*`, `studio/webapp/*` | wrap (ทีม UI) | หน้าตา + ระบบ login ใช้ต่อ; เพิ่มหน้ารีวิวสินค้าที่เรียก `/v1/...` |
| `studio/telegram_bot.py` | wrap (Phase หลัง) | เปลี่ยนจาก `JobQueue` เป็นเรียก API |
| `studio/thumbnail.py` | rewrite | ผูกกับ cast/episode |
| `studio/stats.py` | wrap (Phase หลัง) | ดึงยอดวิว YouTube |
| `studio/rig.py`, `studio/episode.py`, `studio/background.py`, `studio/fx.py`, `studio/characters.py`, `studio/series.py`, `studio/topics.py`, `studio/demo*.{py,js}` | **do-not-use** | เฉพาะ Stickman / สัญญา `episode.json` / ซีรีส์การ์ตูน |

## วิธีดึงโค้ดเดิมมาใช้ (ยังต้องตัดสินใจ)

1. **คัดลอกเฉพาะฟังก์ชันที่ต้องใช้** มาไว้ใน `app/legacy/` พร้อมบันทึก commit ต้นทาง (ง่ายสุด ไม่ผูกกัน แต่ต้อง sync เอง) — **แนะนำสำหรับ MVP**
2. ติดตั้ง ai-animation-studio เป็น package (ต้องเพิ่ม `pyproject.toml` ใน repo เดิม)
3. git submodule
