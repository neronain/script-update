# script-update — คลัง controller ที่ "รันผ่านจริง" ของ LMDS (candidates)

รีโปนี้คือ **ที่พักของ controller ที่ผ่านการทดสอบแล้ว** ก่อนจะถูกเลื่อนขั้นเข้าคลังกลาง
เมื่อ LMDS deploy โมเดลหนึ่งตัวจน "ได้ค่าที่ถูกและดีที่สุด" แล้ว (ผ่าน capability test,
declared ตรงกับ measured) มันจะ **publish controller ตัวนั้นมาที่นี่อัตโนมัติ** เพื่อให้
เครื่องอื่นในกองดึงไปใช้ได้เลย ไม่ต้องมานั่งเดา parser / context / flag ใหม่ทุกครั้ง

> **สั้น ๆ:** ความรู้ที่แลกมาด้วยการ debug บนเครื่องหนึ่ง ควรกลายเป็นของทั้งกอง
> ไม่ใช่หายไปแล้วให้เครื่องถัดไปเจอปัญหาเดิมซ้ำ

---

## รีโปนี้อยู่ตรงไหนในภาพใหญ่

LMDS มีคลัง controller สองชั้น:

| ชั้น | รีโป | ใครเขียน | ใครอ่าน |
|---|---|---|---|
| **canonical** (คลังกลาง) | [`dgx-spark-all-controllers`](https://github.com/neronain/dgx-spark-all-controllers) | คนดูแลหลัง review เท่านั้น | ทุกเครื่อง `--sync` |
| **candidates** (คลังนี้) | `script-update` | LMDS publish อัตโนมัติหลังเทสต์ผ่าน | คนดูแลมา review → promote |

ทำไมต้องแยกสองชั้น: ระบบนี้ไม่ได้มีแค่เราใช้ ลูกค้าเอาไปรันบนเครื่องของเขาด้วย
ถ้าปล่อยให้ทุกเครื่อง push เข้าคลังกลางตรง ๆ คลังกลางจะปนเปื้อนค่าที่จูนกับเครื่องเฉพาะตัว
**candidates เป็นด่านกรอง** — ของเข้ามากองที่นี่ก่อน คนดูแลค่อยเลือกตัวที่ดีเลื่อนเข้า canonical

```
เครื่องที่เทสต์ผ่าน ──publish──▶  script-update (candidates)  ──review+promote──▶  dgx-spark-all-controllers (canonical)
                                                                                          │
   เครื่องทุกตัวในกอง  ◀──────────────────────── lmds recipes --sync ─────────────────────┘
```

---

## สิ่งที่เก็บที่นี่ (และสิ่งที่ **ไม่** เก็บ)

controller ที่ publish มาเก็บเฉพาะ **"ความจริงของโมเดล"** — ค่าที่เป็นจริงเหมือนกันทุกเครื่อง
ส่วนค่าที่ขึ้นกับ **เครื่อง** ไม่เก็บที่นี่ เพราะมันต่างกันทุกเครื่อง ถ้าเอามาแชร์จะทำให้เครื่อง
ถัดไปดึงค่าที่ผิดกับตัวเองไปใช้

| ✅ เก็บ (ของโมเดล) | ❌ ไม่เก็บ (ของเครื่อง — อยู่ที่ `bundle.env`/`lmds set`) |
|---|---|
| `engine` (llama.cpp / vllm / sglang) | `port` — ชนกันทุกเครื่อง ตั้งเองตอน deploy |
| `image` (pinned digest) | `context` cap — dedicated 262K แต่ shared 131K |
| `tool_parser` / `reasoning_parser` | `slots`, `gpu_util` |
| ไฟล์ `mmproj`, chat-template ที่ต้อง override | |
| **native context** + สูตร KV (ให้ fit คำนวณ cap ตามเครื่องเอง) | |
| quant ที่ start ขึ้นจริง | |
| **measured capabilities** (chat/vision/tools/reasoning) | |
| `validated_on` (วันที่ + เครื่องที่เทสต์), `source` (repo@revision + ชื่อ gguf) | |

> เคสจริงที่ทำให้กฎนี้จำเป็น: โมเดลตัวเดียวกันอยากได้ context 262K บนเครื่องเดี่ยว แต่ 131K
> บนเครื่องที่รันหลายตัว · และทุกเครื่อง port ต้องไม่ชนกัน — พวกนี้เป็นค่าของ **เครื่อง**
> ไม่ใช่ของโมเดล จึงห้าม publish

---

## โครงสร้างรีโป

```
script-update/
├── README.md
└── controllers/
    └── <model-slug>/
        ├── <model-slug>-single.sh    # controller · header เป็นบล็อก KEY="value" ที่ LMDS อ่าน
        └── PROFILE.yaml              # provenance: measured caps, validated_on, source, sha256
```

LMDS อ่านเฉพาะ **ส่วน header** ของ `*-single.sh` (บล็อกตัวแปรบนสุดรูปแบบ `KEY="ค่า"` หรือ
`KEY="${KEY:-ค่า}"`) — **อ่านอย่างเดียว ไม่รันสคริปต์** เพราะการรัน controller คือการรัน
โค้ด Bash จากอินเทอร์เน็ตบนเครื่อง hub

### ฟิลด์ที่ header ต้องมี

controller จะเข้าแคตตาล็อกได้ต้องมีอย่างน้อย:

- `ENGINE` — llama.cpp / vllm / sglang
- `SOURCE` — ที่มาของ weight (`org/repo@revision` + ชื่อไฟล์ gguf)
- `VALIDATED_ON` — วันที่ + เครื่องที่ทดสอบผ่าน (เช่น `2026-08-16 · msi-3`)

ที่เหลือ (image, parser, mmproj, ฯลฯ) มีเท่าที่โมเดลนั้นต้องใช้จริง

---

## วิธีใช้งาน (consume)

ดึงสูตรจากคลังนี้ (หรือ canonical) มาที่ hub:

```bash
lmds recipes --sync                                  # จาก canonical (ค่าเริ่มต้น)
lmds recipes --sync --repo https://github.com/neronain/script-update   # จาก candidates
lmds recipes <model>                                 # ดูค่าที่รันผ่านของโมเดลนั้น
```

เมื่อ `lmds deploy` แบบไม่มี LLM provider มันจะหยิบค่าจากแคตตาล็อกแทนการเดา —
ได้ image ที่ถูกรุ่น, parser ที่ตรง, และข้อบังคับของ quant ที่ถ้าไม่ตั้งแล้ว start ไม่ขึ้น

---

## วงจรชีวิตของ controller หนึ่งตัว

1. **deploy + จูน** — บนเครื่องหนึ่งจน start ขึ้น เสิร์ฟได้ ผ่าน capability test (declared ตรง measured, drift = 0)
2. **publish** — `lmds recipes --publish <slug>` (หรืออัตโนมัติหลัง test ผ่าน) → push controller + PROFILE มาที่รีโปนี้ พร้อม `validated_on`
3. **review** — คนดูแลดูว่าค่าถูกต้อง ไม่มีของเฉพาะเครื่องหลุดมา
4. **promote** — เลื่อนเข้า `dgx-spark-all-controllers` (canonical)
5. **pull** — เครื่องทั้งกอง `lmds recipes --sync` ได้ค่าที่พิสูจน์แล้วไปใช้

---

## เรื่องความปลอดภัย

- **ไม่มีความลับในนี้** — `bundle.env` ตัด API key ออกโดยดีไซน์อยู่แล้ว controller ที่ publish
  จึงไม่มี token/รหัส · รีโปนี้เปิด public ได้อย่างปลอดภัย
- **เครื่องลูกค้าไม่ push เข้ารีโปของเรา** — deployment ของลูกค้าตั้ง publish target เป็น
  local (หรือ repo ส่วนตัวของเขา) เขาได้ประโยชน์จากการ pull canonical ฟรี และสร้าง cache
  ของ fleet ตัวเองได้ แต่ไม่มีทางเขียนกลับมาที่คลังของเรา
- controller ถูกอ่านแบบ **header-only ด้วย regex** ไม่เคยถูกรันตอน sync

---

*รีโปนี้เป็นส่วนหนึ่งของ LMDS (Local Model Deploy Studio) · จัดการ controller ผ่าน
`lmds recipes` — ดู `lmds recipes --help`*
