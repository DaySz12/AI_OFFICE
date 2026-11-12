# AI Office — แผน Tech Stack และ Task

เป้าหมาย: หน้าออฟฟิศ pixel ที่แสดงสถานะ AI แต่ละตัวแบบเรียลไทม์ (ทำงาน / รอคนตอบ / ว่าง / ติดปัญหา / ออฟไลน์)
ต้นแบบหน้าตา: `prototype.html` (ไฟล์เดียว เปิดในเบราว์เซอร์ได้เลย ข้อมูลจำลอง)

## 1. ภาพรวมระบบ

```
[AI แต่ละตัว] --(HTTP POST /api/events)--> [.NET API] --(SignalR)--> [หน้าเว็บ Vue]
 Claude Code hooks                          คำนวณสถานะ                  วาดออฟฟิศ pixel
 bot ที่เขียนเอง                               |
 scheduled task                             [Supabase (Postgres)]
                                            เก็บ agent + event log
```

หลักคือ "AI ส่ง event มาบอก" แทนการไปถามทีละตัว ทุกตัวใช้รูปแบบ event เดียวกัน

## 2. Tech Stack

| ส่วน | เลือก | เหตุผล |
|---|---|---|
| Backend | ASP.NET Core (.NET 8 LTS ขึ้นไป) Minimal API | รับ event, คำนวณสถานะ |
| Real-time | SignalR | มากับ ASP.NET Core อยู่แล้ว ส่งสถานะไปหน้าเว็บทันที |
| งานเบื้องหลัง | BackgroundService | เช็ค "เงียบเกิน 60 วิ / 5 นาที" ทุก 10 วินาที |
| Database | Supabase (Postgres) | ต่อจาก .NET ด้วย Npgsql + EF Core ผ่าน connection string |
| หน้าเว็บ | Vue 3 + Vite + TypeScript + Pinia | Pinia เก็บสถานะ agent, client SignalR คือ `@microsoft/signalr` |
| ฉาก pixel | Phaser 3 (ฝังใน component ของ Vue) | รองรับ sprite sheet, animation, แผนที่จาก Tiled |
| ออกแบบแผนที่ | Tiled (ฟรี) | export JSON ให้ Phaser |

ทางเลือก: ใช้ Supabase Realtime แทน SignalR ได้ (Vue subscribe ตาราง `agents` ตรงๆ) แต่ต้องตั้ง Row Level Security ให้ดี
แนะนำ SignalR เพราะหน้าเว็บไม่ต้องแตะ database โดยตรง และ key ของ Supabase อยู่ฝั่ง server อย่างเดียว

## 3. โครงสร้างโฟลเดอร์

```
ai-office/
  server/
    AiOffice.Api/         # Minimal API + SignalR hub + BackgroundService
    AiOffice.Api.Tests/
  web/                    # Vue 3 + Vite + Phaser
  adapters/
    claude-code/          # สคริปต์ hook สำหรับ Claude Code
  assets/                 # sprite, tileset, แผนที่ Tiled
  prototype.html
  PLAN.md
```

ตารางใน Supabase:
- `agents` (id, name, kind, room, status, current_task, progress, last_event_at, done_today)
- `agent_events` (id, agent_id, type, task, progress, meta jsonb, created_at)

## 4. รูปแบบ Event (สัญญากลาง)

```csharp
// C# (server) — ฝั่ง Vue ทำ type เดียวกันใน TypeScript
// type: start | heartbeat | ask | error | done | offline
public record AgentEvent(string AgentId, string Type, string? Task, int? Progress, JsonElement? Meta, DateTimeOffset? At);
```

```ts
type EventType = "start" | "heartbeat" | "ask" | "error" | "done" | "offline";

interface AgentEvent {
  agentId: string;      // เช่น "claude-backend"
  type: EventType;
  task?: string;        // งานที่ทำอยู่ เช่น "แก้ bug API login"
  progress?: number;    // 0-100 ถ้ามี
  meta?: Record<string, unknown>;
  at?: string;          // ISO time ถ้าไม่ส่ง server ใส่ให้
}
```

กฎคำนวณสถานะ (ฝั่ง server):
- **กำลังทำงาน**: event ล่าสุดเป็น start/heartbeat และไม่เกิน 60 วินาที
- **รอคนตอบ**: event ล่าสุดเป็น ask
- **ติดปัญหา**: event ล่าสุดเป็น error
- **ว่างงาน**: event ล่าสุดเป็น done หรือเงียบเกิน 60 วินาที
- **ออฟไลน์**: ส่ง offline หรือเงียบเกิน 5 นาที

## 5. ต่อกับ Claude Code (ผ่าน hooks)

ตั้งใน `.claude/settings.json` ให้แต่ละ hook เรียกสคริปต์ที่ POST event ไป server:

| Hook ของ Claude Code | ส่ง event |
|---|---|
| SessionStart / UserPromptSubmit | start |
| PreToolUse / PostToolUse | heartbeat (task = ชื่อเครื่องมือที่ใช้) |
| Notification (ขออนุญาต / รอ input) | ask |
| Stop | done |
| SessionEnd | offline |

ชื่อ agent ใช้จากชื่อโฟลเดอร์โปรเจกต์ หรือกำหนดเองผ่าน env เช่น `AI_OFFICE_AGENT=claude-backend`
ตรวจชื่อ hook และรูปแบบ input กับเอกสาร Claude Code เวอร์ชันที่ใช้อยู่อีกครั้งก่อนเขียนจริง

## 6. Task แบ่งเป็นช่วง

### ช่วง 1: โครงพื้นฐาน (ทำให้ข้อมูลไหลได้)
- [ ] สร้างโปรเจกต์ Supabase + ตาราง `agents`, `agent_events` (ผ่าน EF Core migration)
- [ ] `server/AiOffice.Api`: `POST /api/events`, `GET /api/agents`, SignalR hub `/hubs/office`
- [ ] ฟังก์ชันคำนวณสถานะ + unit test
- [ ] BackgroundService เช็ค "เงียบเกิน 60 วิ / 5 นาที" ทุก 10 วินาที
- [ ] สคริปต์ยิง event ปลอม (แทน simulation ใน prototype) เช่น `.http` file หรือ console app เล็กๆ

### ช่วง 2: หน้าเว็บ
- [ ] `web` ด้วย Vue 3 + Vite + TypeScript + Pinia + Phaser
- [ ] ย้ายฉากจาก `prototype.html` มาเป็น Phaser scene (ใช้หุ่นยนต์แบบเดิมไปก่อน)
- [ ] ต่อ SignalR แล้วให้ตัวละครเดินไปโต๊ะ/มุมพักตามสถานะ
- [ ] แถบสรุปสถานะ + ตัวกรอง + แผงรายละเอียดและ log

### ช่วง 3: ต่อ AI ตัวจริง
- [ ] `adapters/claude-code`: สคริปต์ hook + ตัวอย่าง settings.json
- [ ] ทดสอบกับ Claude Code session จริง 1 ตัว แล้วขยายเป็นหลายตัว
- [ ] เอกสารสั้นๆ ให้ bot อื่นยิง `POST /api/events` ได้ (ตัวอย่าง curl)
- [ ] แสดงจำนวนงานที่เสร็จวันนี้จาก `agent_events`

### ช่วง 4: ทำให้สวย
- [ ] เลือก asset pack (เช็ค license) และทำแผนที่ใน Tiled
- [ ] sprite ตัวละคร: เดิน 4 ทิศ, นั่งพิมพ์, หลับ, ยกมือถาม
- [ ] เอฟเฟกต์ลูกโป่ง ? / ! / zZ, เสียงแจ้งเตือนเมื่อมีตัวติดปัญหา (เลือกเปิดได้)

### ช่วง 5: ใช้งานจริง
- [ ] ตั้ง API key ให้ `POST /api/events` และเก็บ connection string ของ Supabase ใน user-secrets / env (กันคนอื่นยิงมั่ว)
- [ ] Dockerfile (.NET + build Vue เป็น static ให้ .NET เสิร์ฟ) + วิธีรันบนเครื่อง/server
- [ ] (ทางเลือก) แจ้งเตือนไป LINE / Slack เมื่อมีตัวรอคนนานเกิน X นาที

## 7. เริ่มใน VS Code

1. สร้างโฟลเดอร์ `ai-office` แล้ววาง `PLAN.md` กับ `prototype.html` ไว้ข้างใน
2. เปิด Claude Code ใน VS Code แล้วพิมพ์ประมาณนี้:

> อ่าน PLAN.md กับ prototype.html แล้วทำ "ช่วง 1: โครงพื้นฐาน" ให้หน่อย ใช้ stack ตามแผน (.NET + Vue + Supabase) ทำเสร็จแล้วให้ยิง event ปลอมทดสอบได้

3. ทำทีละช่วง เช็คผลแล้วค่อยไปช่วงถัดไป

## คำถามที่ยังเปิดอยู่
- จะดู AI ตัวไหนบ้าง (Claude Code กี่ session, bot อะไรบ้าง)
- รันบนเครื่องตัวเองอย่างเดียว หรือต้องเปิดให้คนอื่นในทีมดูด้วย
