# ระบบประเมินและคาดการณ์ผลกระทบจากภัยแล้งต่อพืชเศรษฐกิจของประเทศไทย

**GOTTALOMLOK แล้ง** — ดัชนีเสี่ยงภัยแล้งรายอำเภอ ครอบคลุม 928 อำเภอทั่วประเทศ
ข้าว · อ้อย · มันสำปะหลัง · ข้าวโพด

🔗 **เปิดใช้งานระบบ:** https://fancy-breeze-807e.suranajkrua165.workers.dev
📄 **รายงานฉบับเต็ม:** [รายงาน_Lab3_ระบบประเมินภัยแล้ง.docx](./รายงาน_Lab3_ระบบประเมินภัยแล้ง.docx)
📐 **System Design:** [docs/SYSTEM_DESIGN.md](./docs/SYSTEM_DESIGN.md)

นายสุรนาจ เครือวาท · เลขทะเบียน 6606614847

---

## ระบบนี้ทำอะไร

เลือกพืชเศรษฐกิจ 1 ใน 4 ชนิด แล้วดูระดับความเสี่ยงภัยแล้งของทุกอำเภอทั่วประเทศบนแผนที่เดียว
คลิกที่อำเภอเพื่อดูค่าดัชนีรายพืช ตัวชี้วัดเบื้องหลัง และแนวโน้มย้อนหลัง 12 รอบ
วาดขอบเขตพื้นที่เพื่อสรุปผลหลายอำเภอพร้อมกัน และส่งออกข้อมูลเป็นไฟล์ GIS ได้ 4 รูปแบบ

ผู้ใช้เป้าหมาย: เกษตรกร · เกษตรอำเภอและเกษตรจังหวัด · ผู้ประกอบการโรงสีและโรงงานแปรรูป · หน่วยงานด้านน้ำ

---

## สถานะข้อมูล (อ่านก่อนใช้ตัวเลข)

| ชั้นข้อมูล | สถานะ |
|---|---|
| ขอบเขตอำเภอ 928 แห่ง และจังหวัด 77 แห่ง | **ข้อมูลจริง** จากชั้นข้อมูลกรมการปกครอง |
| โครงสร้างฐานข้อมูล API และการส่งออกไฟล์ | **ทำงานได้จริง** ผ่านการทดสอบอัตโนมัติ 15 รายการ |
| VHI · SPI-3 · ความชื้นดินผิว | อยู่ระหว่างเชื่อมต่อ Google Earth Engine (ดู `scripts/gee_indices.py`) |
| พื้นที่ปลูกรายพืช และสัดส่วนเขตชลประทาน | **ค่าสังเคราะห์** ต้องขอข้อมูลจริงจากหน่วยงานที่รับผิดชอบ |
| โมเดลพยากรณ์ 30/60 วัน | ยังไม่ได้ฝึกและยังไม่ผ่านการตรวจสอบความถูกต้อง |

ระบบติดแถบแจ้งเตือนสถานะข้อมูลไว้บนหน้าเว็บ เพื่อไม่ให้นำตัวเลขไปอ้างอิงเชิงนโยบายโดยเข้าใจผิด

---

## วิธีรัน

### ดูเฉพาะหน้าเว็บ (ไม่ต้องติดตั้งฐานข้อมูล)

```bash
python3 -m http.server 5500
```
เปิด http://localhost:5500/frontend/index.html

### รันระบบเต็ม (PostGIS + FastAPI + หน้าเว็บ)

```bash
cp .env.example .env
docker compose up -d                      # PostGIS 16 พร้อมสคีมาอัตโนมัติ

python -m venv .venv && source .venv/bin/activate
pip install -r backend/requirements.txt

python scripts/load_boundaries.py         # 77 จังหวัด + 928 อำเภอ (ข้อมูลจริง)
python scripts/seed_demo_risk.py

cd backend && uvicorn app.main:app --reload --port 8010
```

- เอกสาร API อัตโนมัติ: http://localhost:8010/docs
- ตรวจสถานะระบบ: http://localhost:8010/api/health
- เชื่อมหน้าเว็บเข้ากับ API: แก้ `const API_BASE = "http://localhost:8010";` ใน `frontend/index.html`

### ทดสอบ

```bash
pytest -v        # 15 รายการ ทดสอบกับ PostGIS จริงและตรวจไฟล์ส่งออกด้วย ogrinfo
```

---

## API

| Method | Endpoint | หน้าที่ |
|---|---|---|
| GET | `/api/risk` | ชั้นข้อมูลความเสี่ยงเป็น GeoJSON เลือกพืชและระดับพื้นที่ได้ |
| GET | `/api/risk/point` | ค่าความเสี่ยง ณ พิกัดเดียว (ST_Contains) |
| POST | `/api/query/draw` | สรุปค่าในขอบเขตที่ผู้ใช้วาด (ST_Intersects) |
| GET | `/api/stats/national` | สรุปจำนวนอำเภอและไร่ แยกตามชั้นความเสี่ยง |
| GET | `/api/export` | ส่งออก Shapefile (.zip) / GeoPackage / CSV / GeoJSON |
| GET | `/api/tiles/{z}/{x}/{y}.pbf` | Vector tile จาก ST_AsMVT |
| GET | `/api/raster/tileurl` | XYZ tile URL ของชั้นภาพดัชนีจาก Google Earth Engine |
| GET | `/api/health` | สถานะระบบและจำนวนข้อมูล |

---

## โครงสร้างโปรเจกต์

```
Lab3/
├─ frontend/index.html          หน้าเว็บทั้งหมดในไฟล์เดียว (MapLibre GL)
├─ frontend/data/               ชั้นข้อมูลสำหรับโหมดที่ไม่ต้องใช้ backend
├─ backend/app/routers/         risk · export · tiles · raster
├─ backend/tests/               ชุดทดสอบ 15 รายการ
├─ db/01_schema.sql             ตาราง ดัชนี และ view ของ PostGIS
├─ scripts/
│  ├─ load_boundaries.py        นำเข้าขอบเขตการปกครองจริง
│  ├─ gee_indices.py            คำนวณ VHI / SPI-3 / SSM จาก Google Earth Engine
│  ├─ compute_risk.py           Risk = Hazard × Exposure × Vulnerability
│  ├─ fetch_ftw_parcels.py      นำเข้าขอบเขตแปลงเกษตรจาก Fields of The World
│  └─ seed_demo_risk.py         ใส่ค่าตัวอย่างเพื่อทดสอบระบบ
└─ docs/                        SYSTEM_DESIGN · DATA_SOURCES · REAL_DATA_RECIPES
```

---

## ข้อจำกัดที่ต้องทราบ

1. ภัยแล้งเป็น slow-onset hazard รอบการอัปเดตที่เป็นไปได้จริงคือ 8–10 วัน ไม่ใช่เรียลไทม์
2. เรดาร์ SAR ให้ค่าความชื้นเฉพาะผิวดินระดับ 5 เซนติเมตร ไม่ใช่ความชื้นในเขตราก
3. การแสดงผลระดับอำเภอซ่อนความแตกต่างภายในพื้นที่ ควรพัฒนาสู่ระดับตำบลและระดับแปลง
4. ประกาศเขตภัยพิบัติของหน่วยงานราชการมี administrative bias จึงใช้เป็น ground truth เดี่ยว ๆ ไม่ได้
5. Backend และฐานข้อมูลรันบนเครื่องผู้พัฒนา ลิงก์สาธารณะจึงแสดงผลจากชั้นข้อมูลที่เตรียมไว้
   ฟังก์ชันส่งออก Shapefile/GeoPackage และชั้นภาพจากดาวเทียมทำงานเมื่อรันระบบเต็มตามขั้นตอนข้างต้น

---

## แหล่งข้อมูลและสัญญาอนุญาต

ขอบเขตการปกครอง: กรมการปกครอง ผ่านชุดข้อมูล OpenGISData-Thailand ·
ภาพถ่ายดาวเทียมพื้นหลัง: Esri, Maxar, Earthstar Geographics ·
แผนที่ฐาน: © OpenStreetMap contributors (ODbL) ·
ดัชนีจากภาพดาวเทียม: MODIS (NASA LP DAAC), CHIRPS (UCSB Climate Hazards Center),
Sentinel-1 (ESA Copernicus) ประมวลผลผ่าน Google Earth Engine

รายละเอียดทั้งหมดอยู่ใน [docs/DATA_SOURCES.md](./docs/DATA_SOURCES.md)
