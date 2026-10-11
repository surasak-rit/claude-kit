# 05 — ผลค้นคว้า: วัตถุแทนโครงการวิจัยใน SAP S/4HANA (IO vs WBS vs GM+FM)

> **Input:** [04-architecture.md](04-architecture.md) หัวข้อ 4.3–4.6
> **ค้นคว้าเมื่อ:** 11 ต.ค. 2569
> **ขอบเขต:** เปรียบเทียบ 3 ทางเลือกสำหรับใช้แทนโครงการวิจัยใน S/4HANA (go-live ต.ค. 2570)
> 1. Internal Order (CO)
> 2. WBS Element (PS / Enterprise Projects)
> 3. Grants Management (PSM-GM) ร่วมกับ Funds Management (PSM-FM)

**วิธีอ่านเอกสารนี้**

| สัญลักษณ์ | ความหมาย |
|---|---|
| ข้อความที่มี `[Rn]` | **ข้อเท็จจริง** จากแหล่งอ้างอิงในหัวข้อ 9 |
| `[Rn]*` | ข้อเท็จจริงที่เห็นจาก search snippet หรือจากแหล่งรอง เพราะเปิดหน้าเต็มไม่ได้ (เช่น SAP Community ตอบ 403) ควรยืนยันซ้ำ |
| 💭 | **ความเห็นหรือการอนุมาน** ของผู้ค้นคว้า ไม่ใช่ข้อเท็จจริงจากแหล่ง |
| ❓ | ค้นแล้ว **หาไม่เจอ** หรือยืนยันไม่ได้ |

**ข้อจำกัดของการค้นคว้า**
- หน้ารายละเอียดของ SAP Business Accelerator Hub (api.sap.com) ต้อง login ชื่อ API ในเอกสารนี้จึงยืนยันจาก catalog feed สาธารณะ [R5] ซึ่งให้ชื่อและคำอธิบาย แต่ไม่ให้ field หรือ release ทุกตัว
- SAP Notes ส่วนใหญ่ต้อง login ระบบ S-user จึงอ้างอิงเฉพาะ KBA ที่เปิดอ่านได้สาธารณะ
- SAP Help ส่วนใหญ่ render ด้วย JavaScript บางข้อจึงยืนยันได้จาก snippet เท่านั้น
- ยัง **ไม่รู้ว่ามหาวิทยาลัยจะใช้ S/4HANA รุ่นใด** (on-premise / Private Cloud / Public Cloud) และย้ายแบบ brownfield (system conversion) หรือ greenfield (new implementation) สองข้อนี้เปลี่ยนคำตอบมาก (ดูหัวข้อ 6)

---

## 1. สรุปสำหรับผู้บริหาร

1. **ไม่มีทางเลือกใดที่ "คุมงบตามหมวดของแหล่งทุน" ได้ในตัว ยกเว้น GM**
   - IO และ WBS คุมงบ (availability control) ที่ระดับวัตถุ เป็นงบรวมหรืองบรายปี
   - ระดับ cost element ทำได้เพียง "ยกเว้นบาง cost element ไม่ให้ตรวจ" [R1][R2][R20]
   - GM คุมงบตาม sponsored class × sponsored program × budget validity period ได้โดยตรง [R42]
2. **Internal Order ไม่มีใน S/4HANA Cloud Public Edition** [R3] แต่ยังใช้ได้ใน on-premise / Private Cloud [R4]
3. **API มาตรฐานสำหรับ "สร้าง master" ไม่เท่ากัน**
   - WBS / Enterprise Project มี OData API สร้าง/แก้ได้ [R29][R31]
   - IO ไม่พบ OData API สร้าง/อ่าน master บน Business Accelerator Hub ❓ ต้องใช้ BAPI หรือพัฒนา OData เอง [R5][R12]
   - GM มี API อ่าน Grant แต่ไม่พบ API สร้าง Grant ❓ มีเพียง BAPI `BAPI_0035_CREATE` [R5][R49]
4. **ยอดใช้จริงอ่านได้ทุกทางเลือก** ผ่าน `Journal Entry Item - Read` (Universal Journal / ACDOCA) [R5][R6]
5. 💭 **ยอดผูกพันของการจัดซื้อ** จะไม่อยู่ใน SAP ไม่ว่าจะเลือกทางไหน เพราะจัดซื้อผ่าน WOLF → e-GP ไม่ผ่าน SAP MM
   - เว้นแต่จะบันทึก earmarked funds ใน FM (มี API `Earmarked Funds` [R5])
   - หรือ WOLF ส่ง PO เข้า SAP
6. 💭 **ข้อแนะนำ** (ดูหัวข้อ 7)
   - **go-live ต.ค. 2570:** ใช้วัตถุ "1 โครงการ = 1 วัตถุ" แบบเรียบง่าย ให้ RMS เป็นผู้คุมงบตามหมวดและทำรายงานตามแหล่งทุน
     - ถ้าเป็น **brownfield + on-prem/Private** → คง IO ได้ เสี่ยงต่ำสุด
     - ถ้าเป็น **greenfield** หรือ **Public Cloud** → ใช้ **WBS element ระดับเดียวต่อโครงการ** (Public Cloud ไม่มี IO)
   - **GM+FM:** พิจารณาเป็นระยะที่ 2 โดยเปิดใช้ต้นปีงบประมาณ (1 ต.ค. 2571 เป็นอย่างเร็ว)
     - **ยกเว้น** ทีม S/4HANA มีแผนเปิด FM อยู่แล้วตั้งแต่ go-live

---

## 2. ทางเลือก 1: Internal Order (CO)

### 2.1 แนวคิดและโครงสร้างข้อมูล

| องค์ประกอบ | รายละเอียด | อ้างอิง |
|---|---|---|
| Order type | กำหนด number range, budget profile, settlement profile | [R1] |
| Budget profile | กำหนดช่วงปี (past/future/start), งบรวม (overall) และ/หรืองบรายปี (annual) และการเปิด AVC (Activation Type 0/1/2) | [R2] |
| สถานะ | CRTD → REL → TECO → CLSD (ปิดแล้วลงบัญชีไม่ได้) | [R14] |
| ยอดใช้จริง | อยู่ใน ACDOCA (field order `AUFNR`) cost element กลายเป็น G/L account ประเภทต้นทุน | [R15]* [R16]* |

### 2.2 การคุมงบ
- **คุมงบได้ระดับ IO** เป็น overall หรือ annual และ carry forward งบข้ามปีได้ (ยกเว้น order ที่ปิดหรือติด deletion flag) [R1]
- **Tolerance:** ตั้งเป็น % หรือจำนวนเงิน ต่อ business transaction group ผลคือ warning / warning + mail / error [R1][R2]
- **คุมงบตาม cost element (หมวดงบ):** ❓ ไม่พบความสามารถมาตรฐาน
  - มีเพียง "exempt cost elements" (ยกเว้นบาง cost element จากการตรวจ) [R2]
  - SAP Community ระบุว่าคุมงบ IO ระดับ cost element ไม่ได้ [R10]*
- **Budget Availability Control แบบใหม่ (semantic tag):** มีใน Public Cloud สำหรับ cost center/project [R11] ❓ ไม่พบว่ารองรับ IO
- 💭 **วิธีทั่วไปถ้าจะคุมงบตามหมวดด้วย IO** มี 2 แบบ
  - แตก IO ย่อยต่อหมวด (1 โครงการ = หลาย IO) ซึ่งขัดข้อกำหนด "1 วัตถุต่อโครงการ"
  - ให้ระบบภายนอก (RMS) คุมแทน

### 2.3 รายงานตามแหล่งทุน, overhead, billing
- 💭 IO ไม่มีแนวคิด "แหล่งทุน" ในตัว ต้องใช้ฟิลด์ผู้ใช้หรือ order group / ช่วงเลข order (แบบ Boston University [R9]) แล้วทำรายงานใน RMS/BI
- Overhead คิดผ่าน costing sheet ของ CO ได้ (หลักการเดียวกับ WBS [R26])
- ❓ ไม่พบการเรียกเก็บเงินจากแหล่งทุนสำหรับ IO โดยตรง
  - 💭 resource-related billing (DP90) ออกแบบมาสำหรับ WBS / sales order

### 2.4 สถานะใน S/4HANA
- **on-premise / Private Cloud:** ยังรองรับ
  - มี Fiori app "Manage Internal Orders (Version 2)" F1604A สำหรับ "SAP S/4HANA, all versions" [R4]
  - ❓ ไม่พบ IO ใน Compatibility Scope Matrix (สำเนาบุคคลที่สาม [R13]) ซึ่ง 💭 หมายความว่าไม่ได้ถูกกำหนดวันหมดอายุ
- **Public Cloud:** **ไม่รองรับ** "Internal Order is not supported in SAP S/4HANA Cloud Public Edition" ลูกค้าใหม่ตั้งแต่ 1905 ให้ใช้ Project Control แทน [R3]

### 2.5 API ใน S/4HANA

| ความต้องการ | API / ช่องทาง | สถานะการยืนยัน |
|---|---|---|
| สร้าง / แก้ IO master | ❓ **ไม่พบ OData API บน Business Accelerator Hub** พบเพียง `Internal Order – Data Product` (sap-s4com-InternalOrder-v1) ซึ่งเป็น data product สำหรับอ่าน/delta [R5] | BAPI แบบเดิม `BAPI_INTERNALORDER_CREATE` อ้างใน SAP Community [R12]* ต้องเรียกผ่าน RFC หรือห่อเป็น custom OData |
| อ่านยอดใช้จริง | `Journal Entry Item - Read` (API_JOURNALENTRYITEMBASIC_SRV; on-prem: OP_API_JOURNALENTRYITEMBASIC_SRV_0001) [R5][R6] | catalog |
| ลงบัญชี | `Journal Entry - Post (Synchronous)` (SOAP) [R5] | catalog |
| ยอดผูกพัน | ❓ ไม่พบ API ยอดผูกพันเฉพาะ IO | — |
| งบ IO | ❓ ไม่พบ API | — |

### 2.6 การ migrate จาก ECC
- **brownfield (system conversion):** 💭 IO ติดไปพร้อมระบบ แทบไม่มีงาน migrate
- **greenfield:** มี migration object "CO – Internal Order (restricted)" ใน Migration Cockpit [R7] และ direct transfer จาก ECC [R17]*
- 💭 **ความเสี่ยงต่อ go-live ต่ำที่สุด** ใน 3 ทางเลือก

### 2.7 ตัวอย่างผู้ใช้
- **Boston University:** ใช้ IO เป็น "Sponsored Program" ภายใต้ Fund Center (FM) และกำหนดช่วงเลข IO ตามประเภททุน [R8][R9]
  - 💭 เป็นตัวอย่างของ IO ที่ใช้ร่วมกับ FM/GM ไม่ใช่ IO เดี่ยว

---

## 3. ทางเลือก 2: WBS Element (Project System / Enterprise Projects)

### 3.1 แนวคิดและโครงสร้างข้อมูล
- **Project definition → WBS element (เป็นลำดับชั้น)** แต่ละ WBS ทำหน้าที่เป็น cost object ได้ [R20]
- **Budget profile:** กำหนด overall/annual, ช่วงปี, สกุลเงิน และการเปิด AVC [R20]
  - AVC ตรวจ "assigned funds" (ยอดผูกพัน + ยอดใช้จริง) ที่ WBS ที่ถืองบและ WBS ลูก [R20]
- **ใน Public Cloud:** ใช้ "Enterprise Projects" (ไม่มี network) [R18]*
  - ส่วน PPM / Commercial Project Management มีเฉพาะ on-prem/Private [R18]*
- 💭 สำหรับ RMS ใช้แบบ **1 project definition + 1 WBS element ต่อ 1 โครงการ** ก็พอ ไม่จำเป็นต้องแตกลำดับชั้น

### 3.2 การคุมงบ
- **AVC แบบ classic (BPGE/BPJA):** คุมระดับ WBS เป็น overall/annual tolerance ตั้งต่อ activity group [R20]
  - ระดับ cost element ทำได้เพียง exempt cost elements [R19]*
  - 💭 จึง **ไม่ได้คุมตามหมวดของแหล่งทุนได้ดีกว่า IO** ข้อนี้ต่างจากที่เขียนไว้เดิมใน 04 หัวข้อ 4.5
- **Project Budget AVC แบบใหม่ (อิง ACDOCP + semantic tag):** อยู่ใน S/4HANA Cloud ตั้งแต่ 1808 / 1911 [R21]*[R22]*
  - Community รายงานว่า **ยังไม่รองรับใน on-prem / Private 2023** [R23]* ❓ ไม่พบข้อมูลรุ่น 2025
- **Cost planning ตาม cost element:** ทำได้ (Easy Cost Planning เขียนลง ACDOCP ตั้งแต่ 2022) แต่เป็นแผน ไม่ใช่การคุมงบ [R24b]*
  - 💭 แปลว่าแสดงงบรายหมวดเทียบยอดจริงในรายงานได้ แต่ไม่ block การลงบัญชี
- **งบข้ามปี:** รองรับทั้ง overall (ตลอดอายุโครงการ) และ annual [R20]

### 3.3 รายงาน, overhead, billing
- **Overhead:** costing sheet ลงต้นทุนอ้อมเข้า WBS อัตโนมัติรายเดือน (ตัวอย่าง Warwick) [R26]
- **Billing แหล่งทุน:** resource-related billing (DP90/DP91 + DIP profile) ยังมีใน S/4HANA on-prem [R24][R25]
  - 💭 ต้องใช้ SD billing ด้วย มหาวิทยาลัยต้องประเมินว่าคุ้มหรือไม่
- 💭 **รายงานตามแหล่งทุน:** ไม่มีแนวคิด sponsor ในตัว ต้องใช้ฟิลด์ผู้ใช้ / ลำดับชั้น / RMS เช่นเดียวกับ IO

### 3.4 สถานะใน S/4HANA
- PS ยังเป็นส่วนหนึ่งของ S/4HANA on-prem / Private และยังพัฒนาต่อ เช่น
  - Project Schedule app
  - CDS view ใหม่ในปี 2025 [R27b]*
- **บางฟังก์ชันอยู่ใน Compatibility Scope**
  - เช่น Claim Management, Progress Tracking, Easy Cost Planning (SAP Note 2269324 เป็นรายการหลัก) [R27]*
  - S/4HANA 2023 ทำให้บางรายการกลายเป็น perpetual scope [R27]*
- **Transaction แบบเก่า:** CJ01/CJ11/CJ12 ไม่ใช่ target architecture ให้ใช้ CJ20N แทน (ตามบล็อกบุคคลที่สาม) [R28]
- **Public Cloud:** ใช้ Enterprise Projects แทน PS classic และ migration object "PS – WBS Element" ใช้ไม่ได้ ต้องใช้ Enterprise Project แทน [R35]
- ❓ ไม่พบ roadmap ทางการเรื่องวันสิ้นสุดของ PS

### 3.5 API ใน S/4HANA

| ความต้องการ | API / ช่องทาง | สถานะการยืนยัน |
|---|---|---|
| สร้าง/แก้ project + WBS (Cloud / Private) | `Enterprise Project` **API_ENTERPRISE_PROJECT_SRV_0002** (OData V2, CRUD ของ project และ WBS) [R29]* มีรายงานว่าเปิดใช้บน Private Edition ได้ [R30] | SAP Help snippet + แหล่งรอง |
| สร้าง/แก้ project + WBS (on-prem) | OData V4 path `/sap/opu/odata4/sap/api_project_v3/srvd_a2x/sap/project/0001/...` [R31]* และ BAPI `BAPI_BUS2054_CREATE_MULTI`, `BAPI_PROJECT_MAINTAIN` ที่ KBA ระบุว่าใช้ใน S/4HANA [R33]* | snippet ❓ ยังไม่ยืนยันชื่อบน api.sap.com |
| อ่านยอดใช้จริง | `Journal Entry Item - Read` [R5]; on-prem มี `OP_PROJECTACTUALFINANCIALITEM_0001` [R5] | catalog |
| อ่านยอดผูกพัน | `Project Commitments` (OP_PROJECTCOMMITMENTITEM_0001) [R5] | catalog |
| อ่านงบ | `Project Budget` (OP_PROJECTBUDGETITEM_0001) [R5] ส่วน Public Cloud **ไม่มี** standard "Project Available Budget" API ให้ใช้ CDS/analytical query [R32] | catalog + KBA |
| เปลี่ยนสถานะ (ปิดโครงการ) | 💭 น่าจะทำผ่าน Enterprise Project API / api_project_v3 (มี operation อ่านสถานะ WBS [R31]*) ❓ ยังไม่ยืนยัน operation ปิด | — |

### 3.6 การ migrate จาก IO ใน ECC
- ❓ **ไม่พบเครื่องมือมาตรฐานแปลง IO → WBS**
- Migration Cockpit มี object "PS – Project" และ "PS – WBS element" (แบบไฟล์) [R34] ❓ ไม่พบ object สำหรับงบ PS ในรายการ 1909
- โครงการที่โอนเข้ามาจะอยู่สถานะ Created ต้อง release (เช่น CNMASSSTATUS) [R36b]*
- 💭 **งานที่ต้องทำ:**
  - mapping IO → WBS
  - ย้ายยอดคงเหลือ / งบคงเหลือของโครงการที่ยังเปิด
  - ปรับ settlement rule, รายงาน, สิทธิ์ผู้ใช้ และฝึกอบรม
- 💭 ความเสี่ยงต่อ go-live: **ปานกลาง** ถ้าใช้ WBS ระดับเดียวและไม่ใช้ network / billing ในช่วงแรก

### 3.7 ตัวอย่างผู้ใช้
- **สหรัฐฯ**
  - Duke University: ใช้ WBS element เป็นรหัสโครงการที่ได้รับทุน [R36]
  - University of Kentucky: ใช้ WBS เป็น cost object และใช้ GM คู่กัน [R37]
  - Purdue: ใช้ Grant + Sponsored Program + WBSE [R53]
- **เยอรมนี:** ใช้ PSP-Element (= WBS) กับทุนภายนอก (Drittmittel) เช่น KIT [R38]
  - Uni Jena ใช้ PSP element ใหม่ร่วมกับ budget module ตั้งแต่ 1 ม.ค. 2024 [R39]
- **สหราชอาณาจักร:** University of Warwick ใช้ SAP project + costing sheet [R26]

---

## 4. ทางเลือก 3: Grants Management (PSM-GM) + Funds Management (PSM-FM)

### 4.1 แนวคิดและโครงสร้างข้อมูล

| วัตถุ | ความหมาย | อ้างอิง |
|---|---|---|
| **Sponsor** | Business Partner ในบทบาทผู้ให้ทุน เก็บค่า default ที่ copy ไป grant | [R40] |
| **Grant** | สัญญาทุน มี grant type, sponsor, budget validity, อัตรา IDC, cost sharing, การตั้งค่า billing | [R40] |
| **Sponsored program** | มิติ "โปรแกรม/วัตถุประสงค์" ตามมุมของแหล่งทุน | [R40] |
| **Sponsored class** | หมวดค่าใช้จ่าย/รายได้ตามมุมแหล่งทุน (มี flag billable) ← **ตรงกับ "หมวดงบตามกติกาแหล่งทุน"** | [R40] |
| **Budget validity period** | ช่วงเวลาของงบ (time slice) ต้องครอบคลุมอายุ grant ทั้งหมด | [R40][R58] |
| **Fund (external/internal)** | external = เงินจากผู้ให้ทุน, internal = เงินสมทบของหน่วยงาน (cost sharing) | [R40] |
| FM account assignment | funds center, commitment item, fund, functional area, funded program | [R59] |

- **Lifecycle ของ grant:** draft → proposal → application → award → closing → closed / canceled กำหนดได้ว่าแต่ละสถานะลงบัญชีอะไรได้บ้าง [R40]
  - 💭 ตรงกับข้อกำหนด "ปิดวัตถุเมื่อปิดโครงการ"
- **ความสัมพันธ์กับ cost object:** grant ↔ WBS ได้ทั้ง 1:1, N:1, 1:N และ derive sponsored program/class อัตโนมัติ [R41][R60]
  - เอกสาร Cloud กล่าวถึง WBS และ cost center **ไม่กล่าวถึง IO** [R41]
  - Boston University (ยุค ECC) ใช้ IO ภายใต้ GM ได้ [R54]
  - 💭 ทางที่ SAP มุ่งไปคือ **WBS + GM + FM**

### 4.2 การคุมงบ
- **คุมงบตาม grant × sponsored program × sponsored class × budget validity** ผ่าน AVC profile + tolerance profile [R42]
  - 💭 นี่คือทางเลือกเดียวที่ **คุมงบตามหมวดของแหล่งทุนได้ในตัว SAP**
- งบภายนอกลงช่วง "OVERALL" (ทั้งอายุทุน) ส่วนงบ cost sharing ตั้งเป็นรายปีได้ [R42] รองรับทุนข้ามปีโดยตรง
- **ใน Cloud:** งบเก็บใน ACDOCP ยอดจริง/ผูกพันเก็บใน ACDOCA ไม่ต้องกระทบยอด [R41][R42]
  - ❓ on-prem: ไม่ยืนยันว่า GM/BCS ย้ายไป ACDOCP แล้ว KBA ยังกล่าวถึงตาราง FM เดิม (FMBDT/FMBDP) [R56]
- **Earmarked funds (FM):** บันทึกยอดผูกพันที่ไม่ได้มาจาก MM ได้ [R5]
  - 💭 ใช้รับยอดผูกพันจาก WOLF เข้า SAP ได้

### 4.3 Overhead, cost sharing, billing, รายงาน
- **Indirect cost (IDC):** คิดต่อ sponsored program × class มีอัตราและเพดานต่อช่วงเวลา [R43]
  - ลงค่าใช้จ่ายใน fund ภายนอก และรายได้ใน fund "IDC Recovery" [R43]
  - on-prem ใช้ transaction `GMIDCPOST` [R45]
- **Billing แหล่งทุน:** manual AR invoice, resource-related billing (ตาม expense หรือ payment), billing plan → Billing Document Request → SD [R44]
- **รายงานตามแหล่งทุน:** GM มี "sponsor view" ที่ map ปี บัญชี และสกุลเงินขององค์กรเป็นมุมมองของแหล่งทุน [R40]

### 4.4 สถานะใน S/4HANA
- **on-prem:** SAP Help ของ S/4HANA On-Premise ยังมีเอกสาร Grantee Management [R46]*
  - Purdue ใช้ GM บนระบบ S/4 Fiori [R53]
  - 💭 ถือว่ายังรองรับ (ไม่พบประกาศรุ่นที่ชัดเจน)
- **Public Cloud:** มี GM (scope item ต่างกันตามรุ่น) [R57] ขยายประเทศเป็นระยะ ❓ **ไม่พบว่ามีสำหรับประเทศไทย**
- **Simplification ที่เกี่ยวกับ FM** [R47]*
  - Former Budgeting System (FBS) ใช้ไม่ได้
  - BCS เป็นระบบงบเดียว [R48]
  - Fund accounting บน Special Ledger และ Average Daily Balance ถูกตัดออก
  - ❓ ไม่พบ simplification item เฉพาะ GM

### 4.5 API ใน S/4HANA

| ความต้องการ | API / ช่องทาง | สถานะการยืนยัน |
|---|---|---|
| อ่าน Grant | `Grant` (OP_API_GRANT_0001), `Grant Hierarchy` (OP_API_GRANTHIERARCHY_0001) [R5] | catalog (อ่านอย่างเดียว) |
| **สร้าง Grant** | ❓ **ไม่พบ OData API สร้าง Grant** มี BAPI `BAPI_0035_CREATE` "Create GM grant master data" [R49] | BAPI |
| Sponsored class | `Sponsored Class (A2X)` (CE_SPONSOREDCLASS_0001, prefix CE = Cloud) [R5] | catalog |
| Sponsored program | ❓ ไม่พบ API มีเพียง data product [R5] | — |
| FM master | `Fund`, `Funds Center`, `Commitment Item`, `Functional Area`, `Funded Program`, `Budget Period` (prefix OP_API_…_0001) [R5] | catalog (อ่าน) |
| งบ FM | `Funds Management Budget Structure (A2X)`, `Funds Management Budget Document Item (A2X)` (อ่าน) [R5] และ BAPI `BAPI_0050_CREATE` สำหรับเอกสารงบ BCS [R50] | catalog + BAPI |
| ยอดผูกพัน + ยอดจริง | `Funds Management Commitment and Actual items` (OP_API_FNDSMGMTCMTMTACTLITEM_0001) [R5] | catalog |
| บันทึกยอดผูกพัน (earmarked funds) | `Earmarked Funds` (OP_API_EARMARKEDFUNDS_SRV_0001, create/read/update) [R5] | catalog |
| ❓ API ลงงบ GM | ไม่พบ | — |

### 4.6 การ migrate จาก IO ใน ECC
- 💭 **เป็นการ implement ใหม่ ไม่ใช่การแปลง** ต้องออกแบบ
  - โครงสร้าง FM (fund center / commitment item / fund) และ derivation
  - master ของ GM
  - การ mapping หมวดงบของแต่ละแหล่งทุน → sponsored class
- **ถ้าเปิด FM หลังมีข้อมูลแล้ว:** ต้อง reconstruct เอกสาร (FMN0, FMN5N สำหรับ earmarked funds, FMAVCREINIT ฯลฯ) [R51]
  - ผู้เชี่ยวชาญแนะนำให้เปิด FM **ต้นปีงบประมาณ** [R52]*
  - 💭 ปีงบประมาณไทยเริ่ม 1 ต.ค. ซึ่งตรงกับ go-live ต.ค. 2570 พอดี
    - ถ้าจะเปิด FM ตั้งแต่ go-live ต้องตัดสินใจและออกแบบให้เสร็จในช่วง blueprint ปัจจุบัน
    - ถ้าไม่ทัน รอบถัดไปที่เหมาะคือ 1 ต.ค. 2571
- ❓ ไม่พบระยะเวลา implement มาตรฐานจาก SAP
- 💭 **ความเสี่ยงต่อ go-live: สูงที่สุด** โดยเฉพาะกับทีม IT ขนาดเล็ก และเมื่อแหล่งทุนภายนอกยังระบุไม่ครบ (04 ข้อจำกัดหลัก)

### 4.7 ตัวอย่างผู้ใช้
- **สหรัฐฯ**
  - Purdue University: Grant / Sponsored Program / WBSE [R53]
  - Boston University: grant → sponsored program / IO → sponsored class [R54]
  - University of Kentucky: GM + WBS [R37]
- **อินเดีย:** Indian Institute of Science ใช้ S/4HANA FM (เฉพาะ FM ไม่ใช้ GM) คุมงบตาม commitment item + fund center และใช้ earmarked funds [R55]
- **ไทย:** ❓ ไม่พบมหาวิทยาลัยไทยที่ใช้ SAP GM จากแหล่งเปิด

---

## 5. ตารางเปรียบเทียบ

| เกณฑ์ (ตามข้อกำหนด RMS) | IO | WBS (PS / Enterprise Project) | GM + FM |
|---|---|---|---|
| 1 วัตถุต่อ 1 โครงการ | ✅ | ✅ (WBS ระดับเดียว) | ✅ grant (+ WBS/IO เป็น cost object) |
| สร้างผ่าน API มาตรฐาน | ⚠️ ไม่พบ OData ใช้ BAPI [R5][R12] | ✅ Enterprise Project / api_project_v3 [R29][R31] | ⚠️ อ่านได้ สร้างผ่าน BAPI [R5][R49] |
| ยอดใช้จริงแยกหมวด (อ่าน) | ✅ ผ่าน G/L account ใน ACDOCA [R6] | ✅ เหมือน IO + Project APIs [R5] | ✅ ตาม sponsored class / commitment item [R5] |
| ยอดผูกพัน (จัดซื้อนอก SAP) | ❌ 💭 ไม่มีที่บันทึก | ❌ 💭 ไม่มีที่บันทึก | ✅ earmarked funds [R5] |
| **คุมงบตามหมวดของแหล่งทุน** | ❌ ระดับ IO เท่านั้น [R1][R10] | ❌ ระดับ WBS เท่านั้น [R20] | ✅ sponsored class [R42] |
| งบข้ามปี | ✅ overall/annual [R1] | ✅ overall/annual [R20] | ✅ budget validity / OVERALL [R42] |
| Overhead | costing sheet | costing sheet [R26] | IDC ตามกติกาแหล่งทุน [R43] |
| Billing แหล่งทุน | ❓ | RRB (DP90) [R24] | GM billing [R44] |
| สถานะ Public Cloud | ❌ ไม่รองรับ [R3] | ✅ Enterprise Projects [R35] | ✅ (❓ ประเทศไทย) [R57] |
| สถานะ on-prem/Private | ✅ [R4] | ✅ บางส่วนเป็น compatibility scope [R27] | ✅ [R46] |
| Migrate จาก ECC IO | 💭 ง่ายที่สุด [R7] | 💭 ปานกลาง ไม่มีตัวแปลง [R34] | 💭 ยาก เป็น implement ใหม่ [R51] |
| ความเสี่ยงต่อ go-live ต.ค. 2570 | 💭 ต่ำ | 💭 ปานกลาง | 💭 สูง |
| ภาระทีม IT เล็ก | 💭 ต่ำ (แต่ต้องดูแล BAPI/RFC) | 💭 ปานกลาง | 💭 สูง |

---

## 6. เกณฑ์การตัดสินใจ

ตอบคำถามตามลำดับ คำถามแรก ๆ ตัดตัวเลือกได้ทันที

1. **S/4HANA รุ่นใด?**
   - Public Cloud → IO ใช้ไม่ได้ [R3] เหลือ Enterprise Project หรือ GM
   - on-prem / Private → ใช้ได้ทั้ง 3 ทาง
2. **ย้ายแบบใด?**
   - brownfield → IO ติดมาด้วย ต้นทุนเปลี่ยนต่ำ
   - greenfield → ทุกทางเลือกต้องสร้างใหม่ ต้นทุนส่วนต่างระหว่าง IO กับ WBS จะแคบลง 💭
3. **ทีม S/4HANA มีแผนเปิด FM (BCS) ทั้งมหาวิทยาลัยหรือไม่?** (หน่วยงานรัฐมักคุมงบแผ่นดินด้วย FM 💭)
   - ถ้ามี → การเพิ่ม GM ทีหลังมีต้นทุนต่ำลงมาก
   - ถ้าไม่มี → GM ต้องลาก FM มาทั้งก้อน
4. **ต้องการให้ SAP block การเบิกเกินงบรายหมวดหรือไม่ หรือให้ RMS คุมก่อนส่งเข้า SAP ก็พอ?**
   - ถ้า RMS คุมได้ → IO/WBS พอ
   - ถ้าต้อง block ใน SAP (เช่น กติกาผู้ตรวจสอบ) → ต้องใช้ GM
5. **สัดส่วนทุนภายนอกที่ต้องรายงาน/เรียกเก็บตามรูปแบบของแหล่งทุน** มาก → GM คุ้มขึ้น น้อย → RMS ทำรายงานได้
6. **ต้องการ billing แหล่งทุนจาก SAP หรือไม่** (แหล่งทุนไทยส่วนใหญ่จ่ายเป็นงวดตามสัญญา 💭 ทำให้ billing ใน SAP อาจไม่จำเป็น)
7. **ทีม IT รับภาระ integration แบบไหนได้**
   - OData มาตรฐาน (WBS) ดูแลง่ายกว่า BAPI/RFC หรือ custom OData (IO, GM create) 💭

---

## 7. ข้อแนะนำสำหรับมหาวิทยาลัยนี้ 💭

> ทั้งหัวข้อนี้เป็นความเห็นจากข้อมูลข้างต้น ต้องยืนยันกับทีม S/4HANA และกองคลัง

**ระยะที่ 1: go-live ต.ค. 2570**
- **ใช้วัตถุแบบเรียบง่าย 1 โครงการ = 1 วัตถุ** และให้ **RMS เป็นผู้คุมงบตามหมวดและทำรายงานตามแหล่งทุน** (ตรงกับสถาปัตยกรรมใน 04 อยู่แล้ว)
  - **ค่าแนะนำหลัก: WBS element ระดับเดียวต่อโครงการ**
    - มี OData API สำหรับสร้าง/แก้ project [R29][R31] ตรงกับข้อกำหนด "สร้างผ่าน API"
    - มี API อ่าน commitment / budget / actual ต่อ project [R5]
    - ใช้ได้ทุกรุ่นของ S/4HANA
    - เป็นฐานที่ GM ใน S/4HANA ผูกด้วย [R41]
  - **ยกเว้น:** ถ้าเป็น **brownfield บน on-prem/Private** และทีม S/4HANA ไม่มีกำลังทำ redesign → **คง IO**
    - ยอมรับว่าต้องสร้าง IO ผ่าน BAPI/RFC หรือ custom OData
    - หรือให้การเงินสร้าง IO ด้วยมือแล้วกรอกเลขกลับใน RMS ช่วงแรก
- **ยอดผูกพันจากการจัดซื้อ:** ให้ RMS ดึงจาก WOLF โดยตรง (`getProcurementStatus`) ไม่คาดหวังจาก SAP
- **ห้ามเลื่อน go-live ของ RMS เพื่อรอเรื่องนี้:** RMS ออกแบบ `SAP_LINK.object_type` (IO / WBS / GRANT / FUND) และตาราง mapping หมวดงบไว้แล้ว เปลี่ยนเฉพาะ adapter

**ระยะที่ 2: ไม่เร็วกว่าปีงบประมาณ 2572 (1 ต.ค. 2571)**
- ประเมิน **GM + FM** เมื่อ
  - S/4HANA นิ่งแล้ว
  - แหล่งทุนภายนอกระบุได้ครบ
  - มีข้อมูลจาก RMS ว่าต้องคุม/รายงานตามแหล่งทุนกี่แบบ
- ถ้าเลือก WBS ในระยะที่ 1 จะต่อยอดเป็น WBS + GM ได้โดยไม่ต้องเปลี่ยน cost object [R41]
- RMS จะย้ายการคุมงบตามหมวดไปให้ SAP (sponsored class) และเหลือเป็น pre-check

**ข้อยกเว้นที่ควรพิจารณา GM ตั้งแต่ go-live**
- เฉพาะเมื่อทีม S/4HANA **มีแผนเปิด FM ตั้งแต่ 1 ต.ค. 2570 อยู่แล้ว** และมีที่ปรึกษาที่เคยทำ GM
  - เพราะการเปิด FM ต้นปีงบประมาณง่ายกว่าเปิดกลางปี [R52]*
- แม้กรณีนี้ ก็ควรทำแค่ FM + GM master + budget ส่วน IDC / billing ไว้ทีหลัง

---

## 8. คำถามสำหรับทีม S/4HANA / ที่ปรึกษา SAP

**รุ่นและแนวทาง**
1. S/4HANA ที่จะใช้เป็นรุ่นใด (on-prem / RISE Private Cloud / Public Cloud) และ release ใด (เช่น 2023 / 2025)
2. ย้ายแบบ brownfield, greenfield หรือ selective data transition
3. แผนปัจจุบันใน blueprint ใช้วัตถุใดแทน IO สำหรับงานอื่น (งบลงทุน, โครงการบริการวิชาการ) ถ้ามีมาตรฐานกลางแล้วควรใช้ตาม
4. มีแผนเปิด **Funds Management (BCS)** หรือ **Grants Management** หรือไม่ ใน wave ใด

**การคุมงบ**
5. ใน release ที่จะใช้ Project Budget AVC แบบอิง ACDOCP รองรับ WBS บน on-prem/Private แล้วหรือยัง [R23]*
6. มีวิธีคุมงบตามกลุ่ม G/L (หมวดงบ) ต่อ WBS/IO โดยไม่ต้องใช้ FM หรือไม่ (เช่น semantic tag [R11][R22]*)
7. ถ้าใช้ FM: commitment item จะ map กับหมวดงบของแหล่งทุนอย่างไร ใช้ derivation แบบใด

**API และ integration**
8. ยืนยันชื่อและการเปิดใช้ API ใน release จริง
   - `API_ENTERPRISE_PROJECT_SRV_0002` / `api_project_v3`
   - `OP_PROJECTCOMMITMENTITEM_0001`
   - `OP_PROJECTBUDGETITEM_0001`
   - `OP_API_JOURNALENTRYITEMBASIC_SRV_0001`
   - `Journal Entry - Post (Synchronous)`
   - `Earmarked Funds`
   - `Grant`
9. ถ้าคง IO: จะให้ RMS สร้าง IO ผ่านช่องทางใด (BAPI ผ่าน RFC / custom OData / BTP) และใครเป็นเจ้าของ
10. ใช้ middleware ใด (BTP Integration Suite, PI/PO ต่อ) และ RMS เรียกตรงได้หรือไม่
11. ช่องทางปิดวัตถุ (TECO/CLSD หรือสถานะ grant) ผ่าน API ได้หรือไม่

**Migration**
12. โครงการวิจัยที่ยังเปิดอยู่ ณ ก.ย. 2570 จะย้ายอย่างไร (ยอดคงเหลือ งบคงเหลือ เงินยืมค้าง)
13. เลข IO เดิมจะคงไว้หรือ map ไปเลขใหม่ (RMS ต้องเก็บทั้งสอง)
14. WOLF จะส่งข้อมูลจัดซื้อ (PO / ยอดผูกพัน) เข้า S/4HANA หรือไม่ ถ้าส่งจะลงเป็น earmarked funds หรือ PO

**การตัดสินใจ**
15. ใครมีอำนาจตัดสินเรื่องวัตถุโครงการวิจัย และกำหนดวันตัดสินเมื่อใด (ข้อเสนอ: ไม่เกิน ธ.ค. 2569 ตาม 04 หัวข้อ 4.5)

---

## 9. แหล่งอ้างอิง

> `*` = ยืนยันจาก search snippet หรือแหล่งรอง ควรเปิดตรวจอีกครั้งด้วยบัญชี SAP

**Internal Order**
- [R1] SAP Learning — Budgeting and Availability Control on Internal Orders: https://learning.sap.com/courses/cost-center-and-internal-order-accounting-in-sap-s-4hana/budgeting-and-availability-control-on-internal-orders
- [R2] SAP PRESS — How to configure budgeting for internal orders in SAP S/4HANA: https://blog.sap-press.com/how-to-configure-budgeting-for-internal-orders-in-sap-s4hana
- [R3] SAP KBA 2853554 — Internal Order is not supported in SAP S/4HANA Cloud Public Edition: https://userapps.support.sap.com/sap/support/knowledge/en/2853554
- [R4] SAP KBA 3768617 — Manage Internal Orders (Version 2) F1604A: https://userapps.support.sap.com/sap/support/knowledge/en/3768617
- [R5] SAP Business Accelerator Hub — public API catalog feed: https://api.sap.com/odata/1.0/catalog.svc/APIContent.APIs
- [R6] SAP KBA 2821106 — Journal Entry Item - Read vs Accounting Document - Read: https://userapps.support.sap.com/sap/support/knowledge/en/2821106
- [R7] SAP KBA 3764865 — migration object "CO – Internal Order (restricted)": https://userapps.support.sap.com/sap/support/knowledge/en/3764865
- [R8] Boston University — Funds Management (IO = Sponsored Program): https://www.bu.edu/sph/faculty-staff/staff-handbook/xi-finance/funds-management
- [R9] Boston University — BUworks Internal Order Types: https://www.bu.edu/tech/support/buworks/help/finance/buworks-internal-order-types
- [R10]* SAP Community Q&A — Cost element wise budget control for internal orders: https://answers.sap.com/t5/enterprise-resource-planning-q-a/cost-element-wise-budget-control-for-internal-orders/qaq-p/12816178
- [R11] SAP KBA 2914209 — Budget Availability Control by semantic tag: https://userapps.support.sap.com/sap/support/knowledge/en/2914209
- [R12]* SAP Community archive — BAPI_INTERNALORDER_CREATE: https://archive.sap.com/discussions/thread/1324978
- [R13] SAP S/4HANA Compatibility Scope Matrix (third-party copy): https://www.innflow.com/images/dokumente/Kompatibilitaetsliste-Landingpage-SAP_S4HANA.pdf
- [R14] SAP Help (4.6C) — order status / system status: https://help.sap.com/saphelp_46c/helpdata/EN/a9/ab749c414111d182b10000e829fbfe/content.htm
- [R15]* ACDOCA field reference (AUFNR): https://queryviz.io/tables/ACDOCA
- [R16]* SAP PRESS — Primary and secondary costs in SAP S/4HANA: https://blog.sap-press.com/primary-and-secondary-costs-in-sap-s4hana
- [R17]* SAP Community — data migration lift and shift (CO Internal Orders via direct transfer): https://groups.community.sap.com/t5/enterprise-resource-planning-blogs-by-members/s4finance-to-s-4hana-sap-finance-data-migration-lift-and-shift/ba-p/13538501

**WBS / Project System**
- [R18]* SAP Community — The two project control possibilities in SAP S/4HANA Cloud: https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/the-two-project-control-possibilities-in-sap-s-4hana-cloud-enterprise/ba-p/13577297
- [R19]* SAP Help (4.6C) — PS availability control / exempt cost elements: https://help.sap.com/saphelp_46c/helpdata/en/b9/3ea440417c11d189400000e829fbbd/content.htm
- [R20] SAP Learning — Configuring Budgeting Functions (Project Financials Control in SAP S/4HANA): https://learning.sap.com/courses/project-financials-control-in-sap-s-4hana/configuring-budgeting-functions
- [R21]* SAP Blog — SAP S/4HANA Cloud 1808 for EPPM: https://blogs.sap.com/2018/08/17/intelligent-erp-update-sap-s4hana-cloud-1808-for-enterprise-portfolio-and-project-management/
- [R22]* SAP Blog — Project budget availability control profile: budget-relevant costs: https://blogs.sap.com/2020/04/29/project-budget-availability-control-profile-project-budget-relevant-costs/
- [R23]* SAP Community Q&A — budget availability control for ACDOCA/ACDOCP for projects (on-prem): https://community.sap.com/t5/enterprise-resource-planning-q-a/can-we-activate-budget-availability-control-for-acdocp-table-for-projects/qaq-p/13734960
- [R24] SAP KBA 2482717 — resource-related billing (DP90) in S/4HANA on-prem: https://userapps.support.sap.com/sap/support/knowledge/en/2482717
- [R24b]* SAP Community — PS Easy Cost Planning integration with S/4HANA financial planning: https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/ps-easy-cost-planning-integration-with-s-4hana-financial-planning/ba-p/13553744
- [R25] SAP Learning — Performing Resource-Related Billing: https://learning.sap.com/courses/project-financials-control-in-sap-s-4hana/performing-resource-related-billing
- [R26] University of Warwick — Research Finance handbook, handover and setup: https://warwick.ac.uk/services/finance/research_finance/post-award-rf-handbook/handover-and-setup/
- [R27]* SAP Community — An update on the compatibility scope for EPPM: https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/an-update-on-the-compatibility-scope-for-enterprise-portfolio-and-project/ba-p/13577186
- [R27b]* SAP Community — EPPM in SAP S/4HANA Cloud Private Edition (2025): https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/enterprise-portfolio-and-project-management-in-sap-s-4hana-cloud-private/ba-p/14349843
- [R28] Serio Consulting — S/4HANA changes for Project Systems: https://serioconsulting.com/blog/s-4hana-changes-for-project-systems-ps/
- [R29]* SAP Help — Enterprise Project API (API_ENTERPRISE_PROJECT_SRV_0002): https://help.sap.com/docs/SAP_S4HANA_CLOUD/988903b47d7040f6ac4ec02e44bb58e4/b467d86283be4a56869f1e6784e47b64.html
- [R30] Payhawk — Enabling API services on SAP S/4HANA Cloud Private Edition: https://payhawk.com/help/enabling-api-services-on-sap-s-4hana-cloud-private-edition
- [R31]* SAP Help (On-Premise) — Read WBS Element Status (api_project_v3): https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/04f8d809ad314e3ea009ad2dc88b3d7f/d893824102cb41ba99cd536fc610ae73.html
- [R32] SAP KBA 3710740 — no standard Project Available Budget API (Public Cloud): https://userapps.support.sap.com/sap/support/knowledge/en/3710740
- [R33]* SAP KBA 2906133 — BAPI_BUS2054_CREATE_MULTI: https://userapps.support.sap.com/sap/support/knowledge/en/2906133
- [R34] SAP — Migration Objects for SAP S/4HANA (1909 on-premise): https://help.sap.com/doc/69dd0f1ef0034261ab6e617e0b2beb21/1909.latest/en-US/MigrationObjects_OP_EN.pdf
- [R35] SAP KBA 2655507 — PS – WBS Element migration not feasible in Public Cloud: https://userapps.support.sap.com/sap/support/knowledge/en/2655507
- [R36] Duke University — WBS elements / fund codes for sponsored projects: https://myresearchpath.duke.edu/wbs-elementsfund-codes-sponsored-projects
- [R36b]* SAP Blog — Data migration approach for investment management: https://blogs.sap.com/2022/08/03/data-migration-approach-for-investment-management/
- [R37] University of Kentucky — chart of accounts / cost object structure: https://ufs.uky.edu/bpm/e-17-1-chart-accounts-cost-object-structure
- [R38] KIT — PSP-Stammdatenformular (Drittmittel): https://www.fima.kit.edu//Drittmittelverwaltung/PSP-Stammdatenformular.pdf
- [R39] Universität Jena — Budcon training 2024: https://wiki.uni-jena.de/download/attachments/144114421/2024_03_22_virtuelle_Schulung_Budcon.pdf?api=v2

**Grants Management + Funds Management**
- [R40] SAP Learning — Understand Grants Management master data: https://learning.sap.com/courses/implementing-grants-management-in-sap-s-4hana-cloud-public-edition-public-sector-higher-education/understand-grants-management-master-data_d76620eb-e45d-4c2f-b237-cfd85ef0bc90
- [R41] SAP Learning — Integrating SAP Grants Management and SAP S/4HANA Cloud: https://learning.sap.com/courses/implementing-grants-management-in-sap-s-4hana-cloud-public-edition-public-sector-higher-education/integrating-sap-grants-management-and-sap-s-4hana-cloud_a220248a-2dce-4c5a-a573-b0e33ac75a1c
- [R42] SAP Learning — Budgeting in SAP Grants Management overview: https://learning.sap.com/courses/implementing-grants-management-in-sap-s-4hana-cloud-public-edition-public-sector-higher-education/budgeting-in-sap-grants-management-overview_da9c44fd-0b79-4565-869f-f7b4ade23944
- [R43] SAP Learning — Calculating indirect costs: https://learning.sap.com/courses/implementing-grants-management-in-sap-s-4hana-cloud-public-edition-public-sector-higher-education/calculating-indirect-costs_da08c64d-af20-4344-b796-070287b39870
- [R44] SAP Learning — Running billing processes: https://learning.sap.com/courses/implementing-grants-management-in-sap-s-4hana-cloud-public-edition-public-sector-higher-education/running-billing-processes_f5554b05-3bb4-43d7-bc3f-bd7d20bd6c32
- [R45] se80.co.uk — transaction GMIDCPOST: https://www.se80.co.uk/saptcodes/g/gmid/gmidcpost.htm
- [R46]* SAP Help (On-Premise) — Grantee Management, Maintain Grant: https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/6d5c8dacefc54f6b879285c366aec1b1/58fecc53a8b77214e10000000a174cb4.html
- [R47]* SAP Community — Conversion to SAP S/4HANA with Funds Management active in ECC: https://groups.community.sap.com/t5/enterprise-resource-planning-blogs-by-sap/conversion-to-sap-s-4hana-with-funds-management-module-active-in-sap-ecc/ba-p/13565123
- [R48] SAP PRESS — Customizing budget management settings for the public sector in SAP S/4HANA: https://blog.sap-press.com/customizing-budget-management-settings-for-the-public-sector-in-sap-s4hana
- [R49] sapdatasheet — BAPI_0035_CREATE (Create GM grant master data): https://www.sapdatasheet.org/abap/func/bapi_0035_create.html
- [R50] se80.co.uk — BAPI_0050_REVERSE (FM budget entry document): https://www.se80.co.uk/sapfms/b/bapi/bapi_0050_reverse.htm
- [R51] SAP Help — FM reconstruction / subsequent activation tools: https://help.sap.com/doc/baf2cc53a8b77214e10000000a174cb4/3.6/en-US/62eade51a1661c6be10000000a441470.html
- [R52]* SAP Community archive — activating FM (start of fiscal year): https://scn.sap.com/thread/1961889
- [R53] Purdue University — SPS Grants Management training: https://www.purdue.edu/sps/Training/grantsmgmt.html
- [R54] Boston University — GM reference materials: https://www.bu.edu/tech/support/buworks/help/grants-management-help-guides/gm-reference-materials/
- [R55] Indian Institute of Science — S/4HANA FM overall concept: https://digits.iisc.ac.in/SAP-Training-Material/Finance%20&%20Controlling/Funds%20Management/1.%20IISc_SAP_S4HANA_FM_Overall_Concept_Presentation_V%201.0.pdf
- [R56] SAP KBA 2431477 — FM budget tables in S/4HANA: https://userapps.support.sap.com/sap/support/knowledge/en/2431477
- [R57] SAP Learning — Using SAP Grants Management: basic overview: https://learning.sap.com/courses/implementing-grants-management-in-sap-s-4hana-cloud-public-edition-public-sector-higher-education/using-sap-grants-management-basic-overview_bc0aa03c-2891-406b-b415-957a4ae03a9a
- [R58] SAP error message GMMASTERDATA008 (budget validity): https://www.michaelmanagement.com/sap-error-messages/en/gmmasterdata/gmmasterdata008
- [R59] SAP Training IPS910 (FM account assignment elements): https://cdn.training.sap.com/cdn/course-pdf/IPS910_EN_Col11_CO_A4.pdf/G/EN/IPS910/011
- [R60] SAP Learning — Explaining account assignment derivation: https://learning.sap.com/learning-journeys/implementing-budget-management-process-in-sap-s-4hana-cloud-public-edition-for-public-sector/explaining-account-assignment-derivation_a640e0d9-136b-4791-a7fb-4275583d84b6
