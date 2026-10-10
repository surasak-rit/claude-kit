# 04 — สถาปัตยกรรมและเทคนิค

> **Input:** [03-requirements.md](03-requirements.md)
> **ข้อจำกัดหลัก:**
> - ทีม IT ขนาดเล็ก
> - SAP ECC 6.0 จะย้ายไป S/4HANA
> - แหล่งทุนภายนอกยังระบุไม่ได้
> - ทุกเรื่องต้องผ่านคณะ

## 1. ภาพรวมสถาปัตยกรรม

**รูปแบบที่แนะนำคือ Modular Monolith** คือแอปเดียว deploy ครั้งเดียว แต่แบ่งโมดูลภายในชัดเจน
เหตุผลคือทีมเล็กดูแลง่ายกว่า microservices มาก แต่ยังแยกบางโมดูลออกเป็น service ภายหลังได้
ส่วนที่แยกออกมาตั้งแต่แรกมีแค่ **Integration Layer** เพราะต้องเปลี่ยนตาม SAP

```mermaid
flowchart TB
  subgraph Users[ผู้ใช้]
    U1[นักวิจัย / คณะ / กองวิจัย<br/>การเงิน / ผู้บริหาร]
    U2[ผู้ทรงคุณวุฒิภายนอก]
  end

  subgraph RMS[RMS — Modular Monolith]
    WEB[Web App · Responsive SPA]
    API[REST API]
    subgraph Core[Core Modules]
      M1[Project & Contract]
      M2[Installment]
      M3[Expense & Budget Ledger]
      M4[Report & Review]
      M5[Change Request]
      M6[Closeout]
      M8[Funder Profile]
    end
    subgraph Platform[Platform Services]
      WF[Workflow Engine<br/>state machine + SLA]
      RULE[Rule Engine]
      DOC[Document Generator<br/>DOCX→PDF]
      NOTI[Notification]
      AUD[Audit Log]
      OUTBOX[(Outbox)]
    end
    DB[(PostgreSQL)]
    OBJ[(Object Storage<br/>ไฟล์แนบ/สัญญา)]
    BI[Dashboard / Reporting]
  end

  subgraph INT[Integration Layer]
    PORT[Canonical Finance API<br/>interface กลางของ RMS]
    ECC[ECC Adapter<br/>RFC/BAPI]
    S4[S/4HANA Adapter<br/>OData/SOAP API]
  end

  subgraph EXT[ระบบภายนอก]
    SAPECC[(SAP ECC 6.0)]
    SAPS4[(SAP S/4HANA)]
    HR[(ระบบ HR)]
    IDP[SSO / IdP]
    ESIGN[e-Signature / CA]
    MAIL[Email / LINE OA]
    FUND[แหล่งทุนภายนอก<br/>export / API]
  end

  U1 --> WEB
  U2 -->|ลิงก์ + OTP| WEB
  WEB --> API --> Core
  Core --> Platform
  Core --> DB
  DOC --> OBJ
  Platform --> DB
  BI --> DB
  OUTBOX --> PORT
  PORT --> ECC --> SAPECC
  PORT -.->|หลังอัปเกรด| S4 -.-> SAPS4
  API --> IDP
  Core --> HR
  DOC --> ESIGN
  NOTI --> MAIL
  M4 --> FUND
```

### หน้าที่ของแต่ละส่วน

| ส่วน | หน้าที่ | หมายเหตุ |
|---|---|---|
| Workflow Engine | ทุกเรื่องที่ต้องอนุมัติใช้ engine เดียว: กำหนดขั้น, ผู้ถือเรื่อง, SLA, ผู้ปฏิบัติแทน, ประวัติ | กำหนดเส้นทางด้วย config ในตาราง (ไม่ hard-code) เพื่อปรับตามระเบียบได้ |
| Rule Engine | ตรวจเงื่อนไขงวด, budget check, เส้นทางอนุมัติตามวงเงิน/ประเภท | กฎเก็บเป็น JSON (เช่น JSONLogic) ADMIN แก้ได้ |
| Funder Profile | เก็บกติกาของแต่ละแหล่งทุน: หมวดงบ, งวด, แบบฟอร์ม, คืนเงิน, export | รองรับแหล่งทุนที่ยังระบุไม่ได้ด้วยการเพิ่ม config ไม่ต้องแก้โค้ด |
| Budget Ledger | บัญชีย่อยต่อโครงการ ต่อหมวด: งบ / ผูกพัน / ใช้จริง | MVP: RMS เป็นผู้บันทึก · เฟส 1.5+: ยอดใช้จริงมาจาก SAP |
| Outbox | เก็บ event ที่จะส่งไป SAP ใน transaction เดียวกับข้อมูล แล้วค่อยส่งแบบ async + retry | กันข้อมูลหายหรือส่งซ้ำ |
| Integration Layer | แปลง canonical model ↔ SAP | แยก deploy ได้ เปลี่ยน adapter ตอนย้ายไป S/4HANA |

---

## 2. Data Model หลัก

```mermaid
erDiagram
  FUNDER ||--o{ FUNDER_PROFILE : "มีกติกา (versioned)"
  FUNDER_PROFILE ||--o{ PROJECT : ใช้
  ORG_UNIT ||--o{ PROJECT : สังกัด
  PERSON ||--o{ PROJECT_MEMBER : เป็น
  PROJECT ||--o{ PROJECT_MEMBER : มี
  PROJECT ||--|| CONTRACT : มี
  CONTRACT ||--o{ CONTRACT_AMENDMENT : แก้ไข
  PROJECT ||--o{ BUDGET_LINE : "งบตามหมวด"
  PROJECT ||--o{ INSTALLMENT : แบ่งงวด
  INSTALLMENT ||--o| DISBURSEMENT_REQUEST : เบิก
  PROJECT ||--o{ EXPENSE_REQUEST : "จัดซื้อ/ยืม"
  EXPENSE_REQUEST ||--o{ EXPENSE_ITEM : รายการ
  EXPENSE_REQUEST ||--o| ADVANCE_CLEARANCE : ล้างหนี้
  BUDGET_LINE ||--o{ LEDGER_ENTRY : เคลื่อนไหว
  PROJECT ||--o{ REPORT : ส่ง
  REPORT ||--o{ REPORT_VERSION : "ฉบับแก้"
  REPORT ||--o{ REVIEW : ประเมิน
  PROJECT ||--o{ OUTPUT : ผลผลิต
  PROJECT ||--o{ CHANGE_REQUEST : ขอเปลี่ยน
  PROJECT ||--o| CLOSEOUT : ปิด
  WORKFLOW_INSTANCE ||--o{ WORKFLOW_TASK : ขั้น
  PERSON ||--o{ WORKFLOW_TASK : ถือเรื่อง
  DOCUMENT }o--|| PROJECT : แนบ
  SAP_LINK }o--|| PROJECT : อ้างอิง

  PROJECT {
    uuid id PK
    string code "รหัสโครงการ"
    string title_th
    string title_en
    uuid funder_profile_id FK
    uuid org_unit_id FK
    date start_date
    date end_date
    decimal total_budget
    enum status "draft|contracting|active|closing|closed|terminated"
    bool has_obligation "ค้างภาระ"
  }
  CONTRACT {
    uuid id PK
    string contract_no
    uuid template_id
    bool is_standard "แก้จาก template หรือไม่"
    enum status
    date effective_date
    uuid signed_pdf_id
  }
  BUDGET_LINE {
    uuid id PK
    string category "หมวดงบตาม funder"
    decimal planned
    decimal committed
    decimal actual
  }
  INSTALLMENT {
    uuid id PK
    int seq
    decimal amount
    json condition "เช่น report:progress:1:approved"
    date due_date
    enum status "pending|eligible|requested|paid"
  }
  DISBURSEMENT_REQUEST {
    uuid id PK
    enum status
    json rule_results
    string sap_doc_no
  }
  EXPENSE_REQUEST {
    uuid id PK
    enum type "procurement|advance|reimbursement"
    decimal amount
    string budget_category
    date clear_due_date
    enum status
    string sap_doc_no
  }
  LEDGER_ENTRY {
    uuid id PK
    enum kind "commit|release|actual|adjust"
    decimal amount
    string source "rms|sap"
    string ref
  }
  REPORT {
    uuid id PK
    enum type "progress|final|financial"
    int period
    date due_date
    enum status
  }
  OUTPUT {
    uuid id PK
    enum type "publication|ip|prototype|utilization|other"
    string identifier "DOI/เลขคำขอ"
    bool committed "ผูกพันตามสัญญา"
    date due_date
  }
  WORKFLOW_TASK {
    uuid id PK
    string step
    uuid assignee_id
    uuid delegate_id
    timestamp due_at "SLA"
    enum result "approve|reject|return"
  }
  SAP_LINK {
    uuid id PK
    string object_type "WBS|FUND|GRANT|IO|DOC"
    string sap_key
    string system "ECC|S4"
  }
```

**หลักการของ data model**
- `FUNDER_PROFILE` เก็บเป็น version: แก้กติกาแล้วไม่กระทบโครงการเดิมที่ผูกกับ version เก่า
- `LEDGER_ENTRY` เป็นแบบ append-only: ยอด `committed` / `actual` ใน `BUDGET_LINE` คำนวณจาก entry จึงตรวจสอบย้อนหลังได้
- `SAP_LINK` แยกออกมาเป็นตารางของตัวเอง: ตอนย้ายไป S/4HANA เพิ่ม record ใหม่ (`system=S4`) โดยไม่ต้องแก้ schema หลัก

---

## 3. State Machines

### สัญญา (Contract)

```mermaid
stateDiagram-v2
  [*] --> Draft: ระบบสร้างจาก template
  Draft --> LegalReview: แก้จาก template / ทุนภายนอก
  Draft --> PendingPI: มาตรฐาน
  LegalReview --> PendingPI: ผ่าน
  LegalReview --> Draft: ขอแก้
  PendingPI --> PendingFaculty: PI e-Sign
  PendingPI --> Draft: PI ขอแก้
  PendingFaculty --> PendingApprover: คณะอนุมัติ
  PendingFaculty --> PendingPI: ส่งกลับ
  PendingApprover --> Active: ลงนาม
  PendingApprover --> PendingFaculty: ส่งกลับ
  Active --> Amended: Change Request อนุมัติ
  Amended --> Active
  Active --> Closed: ปิดโครงการ
  Active --> Terminated: ยุติ
```

### คำขอเบิกงวด (Disbursement Request)

```mermaid
stateDiagram-v2
  [*] --> Eligible: เงื่อนไขงวดครบ (event)
  Eligible --> PendingPI: ระบบสร้างคำขอ
  PendingPI --> PendingFaculty: PI ยืนยัน
  PendingFaculty --> RuleCheck: คณะอนุมัติ
  RuleCheck --> PendingFinance: ผ่านกฎ
  RuleCheck --> Exception: ไม่ผ่านกฎ
  Exception --> PendingFinance: RO อนุมัติข้ามกฎ (มีเหตุผล)
  Exception --> PendingPI: ส่งกลับ
  PendingFinance --> PendingApprover: การเงินตรวจ
  PendingApprover --> Approved
  Approved --> Paid: SAP จ่ายแล้ว (sync / แจ้งด้วยมือใน MVP)
  Paid --> [*]
```

### ยืมเงิน → ล้างหนี้ (Advance)

```mermaid
stateDiagram-v2
  [*] --> Draft
  Draft --> PendingFaculty: ส่ง (ผ่าน budget check)
  PendingFaculty --> PendingFinance: อนุมัติ · ตั้งยอดผูกพัน
  PendingFinance --> Disbursed: จ่ายเงินยืม
  Disbursed --> ClearingSubmitted: ส่งหลักฐาน
  Disbursed --> Overdue: เลยกำหนดล้างหนี้
  Overdue --> ClearingSubmitted
  ClearingSubmitted --> Cleared: การเงินตรวจผ่าน · ผูกพัน → ใช้จริง
  ClearingSubmitted --> Disbursed: ตีกลับ
  Cleared --> [*]
```

> ช่วงที่อยู่ในสถานะ `Overdue` → โครงการติดธง `has_obligation` และ rule engine บล็อกการเบิกงวดถัดไป

### รายงาน (Report)

```mermaid
stateDiagram-v2
  [*] --> Upcoming: สร้างจากกำหนดในสัญญา
  Upcoming --> Draft: ใกล้ถึงกำหนด (เตือน)
  Draft --> PendingFaculty: PI ส่ง (validate ครบ)
  Draft --> Overdue: เลยกำหนด
  Overdue --> PendingFaculty
  PendingFaculty --> InReview: ทุนนี้ต้องมีผู้ทรงฯ
  PendingFaculty --> Accepted: ไม่ต้องมีผู้ทรงฯ
  PendingFaculty --> Draft: ส่งกลับ
  InReview --> Revision: ให้แก้
  Revision --> InReview: ส่งฉบับแก้
  InReview --> Accepted
  Accepted --> [*]: emit event → งวดถัดไป Eligible
```

---

## 4. การเชื่อมต่อ SAP (ECC 6.0 → S/4HANA)

### 4.1 หลักการ

1. **RMS ไม่รู้จัก SAP โดยตรง:** core ของ RMS เรียกผ่าน **Canonical Finance API** ที่ RMS นิยามเอง ส่วน adapter แต่ละตัวแปลงเป็นภาษาของ SAP แต่ละรุ่น
2. **แบ่งเจ้าของข้อมูลชัดเจน:** RMS เป็นเจ้าของวงจรชีวิตโครงการ, สัญญา, workflow และเอกสาร ส่วน SAP เป็นเจ้าของการบันทึกบัญชี, การจ่ายเงิน และเงินยืม
3. **ทุก transaction มี idempotency key (เลขอ้างอิง RMS):** ส่งซ้ำแล้ว SAP ไม่บันทึกซ้ำ
4. **กระทบยอดรายวัน (reconciliation job):** เทียบยอดใน RMS กับ SAP แล้วแจ้งเตือนถ้าไม่ตรง

### 4.2 Canonical Finance API

| Operation | ทิศทาง | ใช้เมื่อ | เฟส |
|---|---|---|---|
| `getActuals(projectRef, period)` | SAP → RMS | sync ยอดใช้จริงต่อหมวด | 1.5 |
| `getPaymentStatus(rmsRef)` | SAP → RMS | สถานะการจ่ายงวด / เงินยืม | 1.5 |
| `getOpenItems(projectRef)` | SAP → RMS | เงินยืมและใบสำคัญค้าง สำหรับ checklist ปิดโครงการ | 1.5 |
| `createProjectMaster(project)` | RMS → SAP | สัญญามีผล | 2 |
| `parkDisbursement(request)` | RMS → SAP | คำขอเบิกงวดผ่านการอนุมัติ | 2 |
| `createPurchaseRequisition(req)` | RMS → SAP | คำขอจัดซื้อผ่านการอนุมัติ | 2 |
| `createAdvance(req)` / `clearAdvance(req)` | RMS → SAP | ยืมเงิน / ล้างหนี้ | 2 |

### 4.3 Adapter: ECC vs S/4HANA 🟡

> ตารางนี้เป็นแนวทางเบื้องต้น ต้องยืนยันกับทีม SAP ของมหาวิทยาลัยว่าใช้ module ใดและมี middleware อะไรอยู่แล้ว

| ประเด็น | ECC 6.0 Adapter | S/4HANA Adapter |
|---|---|---|
| Protocol | RFC/BAPI (ผ่าน SAP JCo/NCo) หรือ SOAP ผ่าน SAP PI/PO | OData / SOAP APIs มาตรฐาน (ดูรายการใน SAP Business Accelerator Hub) ผ่าน SAP BTP Integration Suite หรือเรียกตรง |
| Master โครงการ | Fund / Grant (PSM-FM/GM), WBS (PS) หรือ Internal Order (CO) ตามที่ใช้อยู่ | ควรตัดสินใจโครงสร้างใหม่ให้เสร็จก่อน (FM/GM ยังมีใน S/4HANA) |
| บันทึกเอกสารบัญชี | เช่น `BAPI_ACC_DOCUMENT_POST` / park ด้วย custom RFC | Journal Entry API |
| ใบขอซื้อ | `BAPI_PR_CREATE` | Purchase Requisition API |
| ยอดใช้จริง | custom RFC หรือ extract จากตาราง FM/CO | CDS view / OData ของ Universal Journal (ACDOCA) |
| ความเสี่ยง | custom RFC ต้องทิ้งเมื่ออัปเกรด | API บางตัวต้องเปิดใช้งาน / ต้องมี BTP |

### 4.4 Pattern การส่งข้อมูล

```mermaid
sequenceDiagram
  participant Core as RMS Core
  participant DB as PostgreSQL (Outbox)
  participant Worker as Integration Worker
  participant Adp as Adapter (ECC/S4)
  participant SAP
  Core->>DB: อนุมัติคำขอ + insert outbox event (transaction เดียว)
  Worker->>DB: poll event สถานะ pending
  Worker->>Adp: parkDisbursement(rmsRef, payload)
  Adp->>SAP: BAPI / OData call
  SAP-->>Adp: sap_doc_no / error
  alt สำเร็จ
    Adp-->>Worker: ok + sap_doc_no
    Worker->>DB: บันทึก SAP_LINK + event done
  else ล้มเหลวชั่วคราว
    Worker->>DB: retry ด้วย exponential backoff
  else ล้มเหลวถาวร
    Worker->>DB: dead-letter + แจ้ง FIN/ADMIN
  end
```

---

## 5. ทางเลือก Tech Stack

| | **ทางเลือก A: TypeScript Full-stack** | **ทางเลือก B: .NET** |
|---|---|---|
| Frontend | React + Next.js, UI kit (เช่น MUI / shadcn) | React (หรือ Blazor ถ้าทีมไม่ถนัด JS) |
| Backend | NestJS (Node.js) | ASP.NET Core 8 |
| Database | PostgreSQL | PostgreSQL หรือ SQL Server (ถ้ามี license อยู่แล้ว) |
| ORM | Prisma / TypeORM | Entity Framework Core |
| Workflow | state machine ในตารางของ RMS เอง (ถ้าซับซ้อนใช้ Temporal) | state machine ในตารางของ RMS เอง (ถ้าซับซ้อนใช้ Elsa Workflows) |
| สร้างเอกสาร | docx-templates + LibreOffice headless (DOCX→PDF) | OpenXML SDK / DocX + LibreOffice headless |
| ต่อ SAP ECC | ผ่าน middleware (PI/PO, BTP) เป็นหลัก | **SAP NCo** (connector ทางการสำหรับ RFC) หรือผ่าน middleware |
| ข้อดี | ภาษาเดียวทั้ง frontend/backend, หาคนง่าย, ecosystem ใหญ่ | type-safe, เครื่องมือ enterprise ครบ, ต่อ RFC ตรงได้ด้วย connector ทางการ, เหมาะกับองค์กรที่ใช้ Microsoft |
| ข้อเสีย | ต่อ RFC ตรงทำได้ยากกว่า (ต้องพึ่ง middleware) | ถ้าใช้ SQL Server / Windows มีค่า license, คนที่ถนัด .NET + React อาจหายากกว่า |

**คำแนะนำ:** ให้เลือกตาม **ทักษะของทีม IT ที่มีอยู่** เป็นหลัก
- ถ้ามหาวิทยาลัยมี middleware ต่อ SAP อยู่แล้ว (PI/PO หรือ BTP) → **A** หรือ **B** ก็ได้
- ถ้าต้องต่อ ECC ตรงแบบ RFC → **B** เสี่ยงน้อยกว่า

**ทางเลือกที่ไม่แนะนำสำหรับเฟสนี้**
- **Microservices:** ทีมเล็กแบกภาระ operation ไม่ไหว
- **Low-code (เช่น Power Apps):** ทำ prototype ได้เร็ว แต่จะติดข้อจำกัดเรื่อง budget ledger, rule engine และการต่อ SAP

### Infrastructure

| ส่วน | แนะนำ |
|---|---|
| Deploy | Docker containers บน VM หรือ Kubernetes ของมหาวิทยาลัย (on-prem หรือ private cloud) |
| Object storage | MinIO (on-prem, S3-compatible) |
| Search | PostgreSQL full-text ก่อน แล้วค่อยใช้ OpenSearch เมื่อจำเป็น |
| Dashboard | หน้า dashboard ในตัว RMS สำหรับผู้ใช้ทั่วไป และ Metabase / Power BI ต่อ read replica สำหรับนักวิเคราะห์ |
| CI/CD | Git + pipeline (GitLab CI / GitHub Actions / Azure DevOps): test → scan → deploy staging → production |
| Monitoring | log กลาง, metrics และ alert เมื่อ integration ล้มเหลว |

---

## 6. Dashboard และรายงานผู้บริหาร

| กลุ่ม | ตัวชี้วัด | กรองตาม |
|---|---|---|
| ภาพรวมทุน | จำนวนโครงการ active, งบรวม, งบตามแหล่งทุน | ปีงบประมาณ, คณะ, แหล่งทุน, ประเภททุน |
| การเบิกจ่าย | % เบิกจ่ายเทียบแผน, ยอดผูกพัน, ยอดคงเหลือ, เงินยืมค้าง/เลยกำหนด | เดือน, คณะ |
| ความก้าวหน้า | โครงการตามสถานะ, รายงานค้างส่ง/เลยกำหนด, โครงการที่ขอขยายเวลา | คณะ, แหล่งทุน |
| ผลผลิต | ผลผลิตตามประเภท, ผลผลิตที่ผูกพันแต่ยังไม่ส่ง | ปี, คณะ |
| ประสิทธิภาพกระบวนการ | lead time เฉลี่ยต่อขั้น, % เรื่องที่อยู่ใน SLA, หน่วยงานที่ค้างมากที่สุด | ประเภทเรื่อง, หน่วยงาน |
| ความเสี่ยง | นักวิจัยค้างภาระ, โครงการใกล้หมดเวลาแต่ใช้งบน้อย | — |

---

## 7. ความปลอดภัยและ PDPA (สรุปเชิงเทคนิค)

- **Authentication:** SSO (OIDC/SAML) และ MFA สำหรับบทบาทการเงิน/ผู้อนุมัติ ส่วนผู้ทรงฯ ภายนอกใช้ magic link + OTP ที่มีวันหมดอายุ
- **Authorization:** RBAC + policy ตามหน่วยงานและโครงการ ตรวจซ้ำที่ API ทุกครั้ง (ไม่พึ่งการซ่อนปุ่มใน UI)
- **การปกป้องข้อมูล:** เข้ารหัสข้อมูลอ่อนไหว (เลขบัตรประชาชน, เลขบัญชีธนาคาร) ระดับ column และปิดบังบางส่วนเมื่อแสดงผล
- **ไฟล์แนบ:** สแกนไวรัส, จำกัดชนิดไฟล์, ดาวน์โหลดผ่าน signed URL ที่หมดอายุ
- **Audit log:** append-only แยกตาราง และสำรองแยกจากข้อมูลหลัก
- **PDPA:** ทำ data inventory และ retention policy ต่อประเภทข้อมูล ทำ privacy notice สำหรับผู้ทรงฯ ภายนอก และมีช่องทางรับคำขอใช้สิทธิของเจ้าของข้อมูล
