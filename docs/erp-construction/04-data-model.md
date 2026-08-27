# 04 — نموذج البيانات (Data Model)

## 1. العمود الفقري: دفتر الكلفة (Cost Ledger)

هذا أهم قرار تصميمي في النظام كله. **كل** حركة ذات أثر اقتصادي — في أي قسم — تنتج سطراً واحداً في دفتر الكلفة، بغض النظر عن مصدرها.

```
CostLedgerEntry
├── tenant_id, legal_entity_id
├── project_id  ·  wbs_node_id  ·  cost_code_id      ← أين
├── cost_type   (مواد | عمالة | معدات | مقاول ثانوي | مصاريف عامة)
├── entry_type  (BUDGET | REVISION | COMMITMENT | ACTUAL | ACCRUAL | FORECAST)
├── amount, quantity, uom, currency, fx_rate, base_amount
├── source_module, source_doc_type, source_doc_id, source_line_id  ← من أين
├── posting_date, document_date, period_id
├── journal_entry_id (nullable)                       ← الربط بالمحاسبة
├── is_reversed, reversal_of_id
└── created_by, created_at   (append-only — لا UPDATE ولا DELETE)
```

**قاعدة الأنواع الخمسة** لكل عقدة WBS في أي لحظة:

| النوع | المصدر | المعنى |
|-------|--------|--------|
| `BUDGET` | التقدير الفائز | الموازنة الأصلية |
| `REVISION` | الأوامر التغييرية المعتمدة | الموازنة المعدّلة |
| `COMMITMENT` | أوامر الشراء والعقود الفرعية | مال مُلتزم غير مصروف |
| `ACTUAL` | صرف مواد، ساعات عمالة، ساعات معدات، فواتير | كلفة واقعة |
| `FORECAST` | تقدير مراقب التكاليف أو حساب آلي | كلفة إكمال متبقية (ETC) |

ومنها تُشتق كل المؤشرات:

```
الموازنة المعتمدة = BUDGET + REVISION
المتاح للصرف     = الموازنة المعتمدة − COMMITMENT − ACTUAL
EAC              = ACTUAL + FORECAST
الانحراف         = الموازنة المعتمدة − EAC
CPI              = القيمة المكتسبة (EV) ÷ ACTUAL
SPI              = EV ÷ القيمة المخططة (PV)
```

> **لماذا هذا مهم:** الأنظمة الفاشلة تحسب كلفة المشروع من دفتر الأستاذ. دفتر الأستاذ يعرف "مصروف مواد 100 مليون" لكنه لا يعرف *أي جدار في أي طابق*. دفتر الكلفة يعرف.

## 2. الأبعاد الست لكل قيد محاسبي

كل `JournalLine` يحمل:

| البعد | مثال | الغرض |
|-------|------|-------|
| `account_id` | 5101 مصروف مواد | القائمة المالية |
| `cost_center_id` | إدارة التنفيذ | مسؤولية إدارية |
| `project_id` | مشروع برج الرافدين | ربحية المشروع |
| `cost_code_id` | 03.20.100 حديد تسليح | تفصيل الكلفة |
| `partner_id` | شركة الأمين للحديد | حساب الشريك |
| `segment_id` | قطاع الإنشاءات | تقارير القطاعات |

هذا يلغي الحاجة لدليل حسابات من 8000 حساب — 300 حساب × أبعاد = تحليل لا نهائي.

## 3. الكيانات الجوهرية ومخطط العلاقات المبسّط

```
LegalEntity 1──* Project *──1 Contract
                    │             │
                    │             ├──* VariationOrder
                    │             ├──* PaymentCertificate ──* IpcLine
                    │             └──* Guarantee / Retention
                    │
                    ├──* WbsNode (شجرة) ──* Activity (جدول زمني)
                    │        │                   │
                    │        │                   └──* ActivityRelation (FS/SS/FF/SF)
                    │        │
                    │        └──* BoqItem ──* ProgressMeasurement
                    │
                    ├──* CostLedgerEntry ◀── كل شيء
                    ├──* PurchaseRequisition ──* PurchaseOrder ──* GoodsReceipt
                    │                                    │             │
                    │                                    └──── ThreeWayMatch ────┐
                    │                                                  │        │
                    ├──* MaterialIssue                          VendorInvoice ──┘
                    ├──* Subcontract ──* SubcontractIPC
                    ├──* EquipmentAllocation ──* EquipmentLog
                    ├──* TimesheetLine (من HCM)
                    ├──* InspectionRequest / Ncr / PunchItem
                    ├──* WorkPermit / Incident
                    ├──* Rfi / Submittal / Transmittal / Drawing
                    └──* ProjectRisk / ProjectIssue / DailyReport
```

## 4. جداول مرجعية أساسية بتفصيل الحقول

### `Project`
```
id, tenant_id, legal_entity_id, code (فريد), name_ar, name_en,
business_line (CONSTRUCTION|DEVELOPMENT|FM|CONSULTANCY),
client_partner_id, consultant_partner_id,
contract_id, project_manager_id, branch_id, division_id,
type, delivery_method (LUMP_SUM|UNIT_PRICE|COST_PLUS|EPC|DESIGN_BUILD),
status (PLANNED|MOBILIZING|ACTIVE|SUSPENDED|SUBSTANTIALLY_COMPLETE|CLOSED),
planned_start, planned_finish, actual_start, actual_finish,
contract_value, revised_contract_value, currency,
completion_percent, health_score,
location_geo (PostGIS Point/Polygon), address,
retention_percent, advance_percent, defect_liability_months,
created_at, updated_at, deleted_at
```

### `WbsNode`
```
id, project_id, parent_id, path (ltree), level, code, name,
type (PHASE|PACKAGE|WORK_ITEM), unit, budget_qty,
weight_percent (وزن في منحنى الإنجاز),
is_cost_collecting (هل تُجمع عليه الكلفة مباشرة),
responsible_id, planned_start, planned_finish
```
> استخدام امتداد `ltree` في PostgreSQL يجعل استعلامات الشجرة (كل أبناء عقدة، مجموع كلفة فرع) فورية بدل التكرار العودي.

### `Contract` و `PaymentCertificate`
```
Contract: id, project_id, type (MAIN|SUBCONTRACT|SUPPLY|CONSULTANCY),
  party_partner_id, value, currency, signing_date, start_date, duration_days,
  retention_percent, advance_percent, advance_recovery_method,
  payment_terms_days, ld_rate_per_day, ld_cap_percent,
  price_adjustment_formula, governing_law, dispute_resolution

PaymentCertificate (IPC): id, contract_id, sequence_no, period_from, period_to,
  work_done_to_date, previous_certified, current_gross,
  variations_amount, materials_on_site, advance_recovery,
  retention_deducted, ld_deducted, other_deductions, tax_amount,
  net_payable, status (DRAFT|SUBMITTED|UNDER_REVIEW|CERTIFIED|PAID|DISPUTED),
  submitted_at, certified_at, certified_by, ar_invoice_id
```

### `Unit` (التطوير العقاري)
```
id, phase_id, building_id, floor, unit_no, unit_type_id,
area_gross, area_net, area_balcony, bedrooms, bathrooms, view_type,
base_price, price_per_sqm, current_price,
status (PLANNED|UNDER_CONSTRUCTION|AVAILABLE|RESERVED|SOLD|HANDED_OVER|REGISTERED),
reservation_id, sales_contract_id, owner_partner_id,
construction_progress_percent, handover_date, title_deed_no
```

### `WorkOrder` (الصيانة)
```
id, tenant_id, contract_id, asset_id, location_id, request_id,
type (CORRECTIVE|PREVENTIVE|EMERGENCY|INSPECTION),
priority (P1..P4), sla_id, reported_at, sla_response_due, sla_resolve_due,
assigned_team_id, assigned_technician_id,
status (NEW|ASSIGNED|IN_PROGRESS|ON_HOLD|AWAITING_PARTS|COMPLETED|VERIFIED|CLOSED),
started_at, completed_at, response_breached, resolve_breached,
labor_hours, parts_cost, total_cost, billable, invoice_id,
root_cause, resolution_notes, customer_rating, signature_url
```

## 5. الأنماط المشتركة في كل جدول

كل جدول في النظام يحمل هذه الحقول إلزامياً:

```sql
tenant_id        UUID NOT NULL           -- عزل RLS
legal_entity_id  UUID                     -- الكيان القانوني
created_at       TIMESTAMPTZ NOT NULL
created_by       UUID NOT NULL
updated_at       TIMESTAMPTZ
updated_by       UUID
deleted_at       TIMESTAMPTZ              -- حذف منطقي فقط
version          INTEGER NOT NULL         -- تفاؤلي (Optimistic Locking)
external_ref     JSONB                    -- مراجع الأنظمة الخارجية
```

وللمستندات المعاملاتية إضافةً:
```sql
doc_no           TEXT NOT NULL            -- ترقيم من NumberSeries
doc_date         DATE NOT NULL
status           TEXT NOT NULL
workflow_id      UUID
posted_at        TIMESTAMPTZ              -- لحظة الترحيل (لا يُعدّل بعدها)
```

## 6. الجداول المؤرَّخة (Temporal / Slowly Changing)

هذه الكيانات **لا تُحدَّث** — يُضاف سجل جديد بتاريخ سريان:

`ExchangeRate` · `PriceList` · `SalaryStructure` · `InternalHireRate` · `ApprovalMatrix` · `OrgStructure` · `TaxRate` · `Budget`

نمط موحّد: `valid_from`, `valid_to`, `is_current` مع فهرس `EXCLUDE USING gist` لمنع التداخل الزمني.

## 7. الحجوم المتوقعة وقرارات التقسيم (Partitioning)

| الجدول | الحجم السنوي التقديري | القرار |
|--------|------------------------|--------|
| `CostLedgerEntry` | 5–50 مليون سطر | تقسيم بالنطاق على `posting_date` (شهري) |
| `JournalLine` | 3–30 مليون | تقسيم شهري |
| `Attendance` | 1000 موظف × 365 = 0.4 مليون | تقسيم سنوي |
| `AuditLog` | 20–100 مليون | تقسيم شهري + أرشفة بعد 24 شهراً |
| `MeterReading` / IoT | يحتمل الملايين | جدول زمني منفصل (TimescaleDB أو ClickHouse) |
| `Notification` | ملايين | تقسيم + حذف بعد 90 يوماً |

## 8. مخزن الملفات

- الملفات **لا تُخزَّن في قاعدة البيانات** — فقط بيانات وصفية (`Document`) + مفتاح كائن في S3.
- كل ملف: `checksum` (منع التكرار)، `mime`، `size`، `virus_scan_status`، `ocr_text` (مفهرس في OpenSearch)، `encryption_key_id`.
- روابط موقّعة قصيرة الأجل (Pre-signed URLs) — لا وصول مباشر.

## قرارات مفتوحة

1. **الترقيم:** هل يجب أن يكون رقم المستند متسلسلاً بلا فجوات (متطلب ضريبي في بعض الدول)؟ هذا يفرض قفلاً متسلسلاً يؤثر على الأداء — نحتاج تأكيداً.
2. **العملة الأساسية:** دينار عراقي؟ دولار؟ عملتان (محاسبية وتقريرية)؟
3. **السنة المالية:** ميلادية كاملة أم مختلفة؟
4. **البُعد السابع:** هل نحتاج بُعد "الممول/المصدر التمويلي" (مهم في المشاريع الحكومية والمنح)؟
