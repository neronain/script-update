<div align="center">

# script-update

**คลังพักของ controller ที่ "รันผ่านจริงแล้ว" ก่อนเลื่อนขั้นเข้าคลังกลางของ LMDS**

ความรู้ที่แลกมาด้วยการ debug บนเครื่องหนึ่ง ควรกลายเป็นของทั้งกอง —
ไม่ใช่หายไปแล้วให้เครื่องถัดไปเจอปัญหาเดิมซ้ำ

[![tier](https://img.shields.io/badge/tier-candidates-8a5300)](#สองชั้น)
[![consumed by](https://img.shields.io/badge/consumed%20by-lmds%20recipes-1f5fbf)](https://github.com/neronain/AutoDeployDGXProject)
[![read](https://img.shields.io/badge/scan-header--only%20·%20never%20run-17703f)](#ความปลอดภัย)
[![promotes to](https://img.shields.io/badge/promotes%20to-dgx--spark--all--controllers-555)](https://github.com/neronain/dgx-spark-all-controllers)

</div>

---

## รีโปนี้คืออะไร

เมื่อ LMDS deploy โมเดลบนเครื่องจนได้ค่าที่ **รันผ่าน + ทดสอบแล้ว** (parser ตรง, image ถูกรุ่น,
mmproj ครบ, quant ที่ start ขึ้นจริง) คำสั่ง `lmds recipes --publish <slug>` จะส่ง controller
ตัวนั้นมากองที่นี่ · คนดูแลรีวิว แล้วเลื่อนตัวที่ดีเข้าคลังกลาง (`dgx-spark-all-controllers`)
ที่ทุกเครื่องในกอง `--sync` ไปใช้

รีโปนี้เป็น **ด่านกรอง** ระหว่างเครื่องที่เพิ่งทดสอบเสร็จ กับคลังกลางที่ต้องสะอาดเสมอ

## สองชั้น

| ชั้น | รีโป | ใครเขียน | ใครอ่าน |
|---|---|---|---|
| **canonical** (คลังกลาง) | [`dgx-spark-all-controllers`](https://github.com/neronain/dgx-spark-all-controllers) | คนดูแล หลัง review | ทุกเครื่อง `lmds recipes --sync` |
| **candidates** (คลังนี้) | `script-update` | `lmds recipes --publish` (เครื่องที่เทสต์ผ่าน) | คนดูแล มา review → promote |

```
  เครื่องที่เทสต์ผ่าน ──lmds recipes --publish──▶  script-update  ──review + promote──▶  dgx-spark-all-controllers
                                                    (คลังนี้)                              (canonical, public)
                                                                                                    │
       เครื่องทั้งกอง  ◀────────────────────────── lmds recipes --sync ───────────────────────────────┘
```

ทำไมต้องแยก: ระบบนี้ไม่ได้มีแค่ทีมเราใช้ ลูกค้าเอาไปรันบนเครื่องของเขาด้วย ถ้าปล่อยให้ทุกเครื่อง
push เข้าคลังกลางตรง ๆ คลังกลางจะปนเปื้อนค่าที่จูนเฉพาะเครื่อง — candidates กันไว้ตรงนี้

## สิ่งที่เก็บ — และสิ่งที่ **ไม่** เก็บ

controller ที่ publish มาถือเฉพาะ **"ค่าของโมเดล"** (จริงเหมือนกันทุกเครื่อง) ·
**"ค่าของเครื่อง"** อยู่ใน `bundle.env` คนละไฟล์ ไม่ตามขึ้นมา และฝั่ง sync ก็ตัด context ทิ้ง
อยู่แล้ว — เครื่องปลายทางจึงเอาไป **fit ใหม่ตามหน่วยความจำของตัวเอง** ไม่ใช่รับค่าเครื่องเก่ามาทั้งดุ้น

| ✅ เก็บ (ของโมเดล) | ❌ ไม่เก็บ (ของเครื่อง) |
|---|---|
| `ENGINE`, `image` (pinned digest) | `port` — ตั้งเองตอน deploy (ชนกันทุกเครื่อง) |
| `tool_parser` / `reasoning_parser` | `context` cap — dedicated 262K แต่ shared 131K |
| ไฟล์ `mmproj`, chat-template override | `slots`, `gpu_util` |
| native context + สูตร KV | |
| `MODEL_FEATURES` (ความสามารถที่วัดได้จริง) | |
| `SOURCE` (repo@revision + gguf), `VALIDATED_ON` | |

## โครงสร้าง

```
script-update/
├── README.md
└── controllers/
    └── <model-slug>/
        ├── <model-slug>-single.sh    # controller · header เป็นบล็อก KEY="value" ที่ LMDS อ่าน
        └── PROFILE.yaml              # provenance: measured features, validated_on, source, engine
```

LMDS อ่านเฉพาะ **header** ของ `*-single.sh` (ตัวแปรบนสุดรูปแบบ `KEY="ค่า"` / `KEY="${KEY:-ค่า}"`)
· header ต้องมีอย่างน้อย `ENGINE` · `SOURCE` · `VALIDATED_ON` ถึงจะเข้าแคตตาล็อกได้

## ใช้งาน

```bash
# ดึงจากคลังนี้ (candidates) มาที่ hub
lmds recipes --sync --repo https://github.com/neronain/script-update

# ดึงจากคลังกลาง (ค่าเริ่มต้น)
lmds recipes --sync

# ส่ง controller ที่เทสต์ผ่านขึ้นคลัง (ระบุความสามารถที่วัดได้จริง)
lmds recipes --publish <slug> --features tools,vision,reasoning
```

ปลายทางของ `--publish` ตั้งใน config (`recipes.publish_repo`) · **ว่าง = local store
ในเครื่อง** (ค่าเริ่มต้นที่ปลอดภัยสำหรับลูกค้า: fleet แชร์กันเองโดยไม่แตะรีโปเรา) ·
ทีมเราตั้งเป็นรีโปนี้เพื่อ push ขึ้นไป review

## ความปลอดภัย

- **ไม่มีความลับในนี้** — `bundle.env` ตัด API key ออกโดยดีไซน์ controller ที่ publish จึงไม่มี
  token · เปิด public ได้อย่างปลอดภัย
- **เครื่องลูกค้าไม่ push เข้ารีโปเรา** — publish target ของลูกค้าเป็น local/repo ส่วนตัวเขา
- controller ถูกอ่านแบบ **header-only ด้วย regex ไม่เคยถูกรัน** ตอน sync

---

<div align="center">

ส่วนหนึ่งของ **LMDS · Local Model Deploy Studio** — จัดการผ่าน `lmds recipes`

**neronain** · [facebook.com/neronain.minidev](https://www.facebook.com/neronain.minidev)

</div>
