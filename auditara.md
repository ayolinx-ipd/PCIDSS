# Product Requirements Document (PRD)

## ISO Assessment & Audit Readiness Platform

**Status:** Draft v1.0\
**Tanggal:** 25 September 2026\
**Fokus awal:** ISO/IEC 27001:2022\
**Target MVP/NPP:** Validasi end-to-end assessment engine dengan satu
user berhak penuh, sambil menyiapkan fondasi RBAC untuk implementasi
multi-role.

------------------------------------------------------------------------

## 1. Ringkasan Produk

Produk ini adalah platform **ISO Assessment & Audit Readiness** untuk
membantu perusahaan, internal auditor, dan konsultan audit mengumpulkan
dokumen, memetakan evidence terhadap requirement ISO, melakukan
assessment berbantuan AI, menemukan gap/ketidaksesuaian, memberi
proposed finding (Conformity/Observation/Minor/Major), memberikan
rekomendasi perbaikan, dan menyimpan hasil assessment sebagai
snapshot/version yang dapat dibandingkan dari waktu ke waktu.

Prinsip utama sistem:

> **Master Reference menentukan apa yang harus dipenuhi → dokumen
> perusahaan menyediakan evidence → Qdrant menemukan evidence kandidat →
> LLM menilai coverage → rule/scoring engine menghitung hasil → auditor
> memvalidasi → hasil diarsipkan sebagai audit trail.**

Sistem **bukan pengganti keputusan auditor**. AI menghasilkan evidence
mapping dan proposed assessment; hasil dapat direview, dikomentari, dan
di-override oleh internal auditor/konsultan.

------------------------------------------------------------------------

## 2. Tujuan Produk

1.  Memusatkan Master Reference ISO yang dikelola oleh
    consultant/internal auditor.
2.  Memusatkan dokumen/evidence perusahaan lintas departemen.
3.  Mengurangi pekerjaan manual pencarian evidence.
4.  Menguji setiap clause/control terhadap dokumen perusahaan secara
    konsisten.
5.  Menjelaskan secara transparan evidence mana yang mendukung suatu
    requirement.
6.  Mengidentifikasi requirement yang `MET`, `PARTIAL`, `NOT_MET`, atau
    `NO_EVIDENCE`.
7.  Menghasilkan score readiness dan proposed finding.
8.  Menunjukkan secara spesifik apa yang harus diperbaiki perusahaan.
9.  Mendukung review manusia, komentar, override, dan approval.
10. Membentuk assessment version/snapshot yang immutable.
11. Membandingkan progres antar-version.
12. Menyimpan hasil external audit sebagai final historical snapshot.
13. Menyediakan Audit Copilot berbasis konteks hasil assessment.

------------------------------------------------------------------------

## 3. Non-Goals Awal

Untuk MVP/NPP, sistem belum harus:

-   menggantikan external auditor;
-   memberikan keputusan sertifikasi;
-   menjalankan seluruh ISO framework sekaligus sejak hari pertama;
-   memiliki microservices kompleks;
-   memiliki workflow multi-role penuh secara operasional.

MVP dapat menggunakan **satu Super User** untuk menjalankan keseluruhan
workflow, tetapi database dan authorization harus sudah dirancang agar
multi-role dapat diaktifkan kemudian.

------------------------------------------------------------------------

## 4. Aktor Sistem

### 4.1 Client / Company Department User

Contoh: IT, HR, GA, Sales, Finance, Operations.

Tanggung jawab: - membuat folder; - upload PDF/DOCX dan tipe evidence
yang didukung; - melihat dokumen departemennya; - mengganti/memperbarui
dokumen; - menindaklanjuti gap yang menjadi tanggung jawabnya.

Target production: akses dapat dibatasi berdasarkan company, department,
dan folder.

### 4.2 Management Representative (MR)

MR memiliki visibilitas lintas departemen.

Kemampuan: - melihat seluruh dokumen perusahaan; - memonitor kelengkapan
evidence; - melihat hasil assessment; - melihat
Major/Minor/Observation; - memonitor remediation; - melihat report dan
comparison version; - melakukan review sebelum assessment difinalisasi.

### 4.3 Admin / Assessment Coordinator

Kemampuan: - mengelola workspace assessment; - memilih
single/multiple/all controls; - menjalankan assessment; - memonitor
queue/job; - retry job gagal; - melakukan compile/finalize; - membuat
snapshot/version; - menghasilkan report.

### 4.4 Internal Auditor / Consultant

Kemampuan: - memelihara Master Reference; - melihat seluruh evidence
yang relevan; - review AI assessment; - memberi komentar; -
approve/request revision; - override proposed classification dengan
alasan; - menambahkan rekomendasi; - membantu validasi scoring/rules.

### 4.5 External Auditor

External auditor **tidak wajib menjadi user aktif**.

Setelah external audit selesai, hasil final dapat dicatat sebagai
**External Audit Final Snapshot**, sehingga sistem memiliki historical
trail dari internal readiness sampai hasil audit eksternal aktual.

------------------------------------------------------------------------

## 5. Konsep Domain Utama

### 5.1 Master Reference

Master Reference adalah sumber pengetahuan assessment yang dikurasi
consultant/internal auditor.

Untuk ISO/IEC 27001:2022, struktur harus dapat membedakan: - Clauses
4--10; - Annex A controls, dikelompokkan A.5--A.8.

Engine harus generik agar framework ISO lain dapat ditambahkan kemudian.

Setiap control/clause minimal memiliki:

``` yaml
standard: ISO/IEC 27001
standard_version: 2022
control_code: A.x.x
title: ...
consultant_summary: ...
objective: ...
requirements:
  - requirement_id: R1
    statement: ...
    criticality: HIGH
    weight: 25
    expected_evidence:
      - ...
    mandatory: true
    finding_rules:
      partial: MINOR_CANDIDATE
      not_met: MAJOR_CANDIDATE
```

### 5.2 Requirement Decomposition

Summary tidak boleh menjadi satu-satunya unit penilaian.

Satu control harus dipecah menjadi requirement/criterion yang dapat
diuji, misalnya:

  Requirement                    Criticality     Weight
  ------------------------------ ------------- --------
  R1 Kebijakan tersedia          HIGH               25%
  R2 Tanggung jawab ditentukan   HIGH               25%
  R3 Prosedur terdokumentasi     MEDIUM             20%
  R4 Evidence implementasi       HIGH               20%
  R5 Review periodik             LOW                10%

Assessment dilakukan **per requirement**, lalu diagregasi menjadi hasil
control.

### 5.3 Criticality

Criticality awal: - `HIGH` - `MEDIUM` - `LOW`

Criticality memengaruhi severity/rule, tetapi tidak boleh menggunakan
aturan simplistik seperti `HIGH gagal = MAJOR` untuk semua kasus.
Finding rules harus dapat dikonfigurasi di Master Reference.

------------------------------------------------------------------------

## 6. Document & Evidence Management

### 6.1 Penyimpanan Berlapis

Setiap dokumen perusahaan memiliki tiga representasi:

1.  **Original File** --- PDF/DOCX asli di Object Storage.
2.  **Extracted Content** --- teks hasil extraction yang disimpan
    sebagai data yang dapat diaudit/reprocess.
3.  **Chunks + Embeddings** --- unit pencarian semantic di Qdrant.

Flow:

``` text
Upload File
    ↓
Object Storage
    ↓
Extract Text
    ↓
Store Extracted Content
    ↓
Normalize / Parse Structure
    ↓
Chunk
    ↓
Embedding
    ↓
Qdrant
```

### 6.2 Chunk vs Vector

`Chunk` adalah potongan konten teks.

`Vector` adalah representasi numerik hasil embedding dari satu chunk.

Secara umum:

``` text
1 Document
   ├── Chunk 1 → Vector 1
   ├── Chunk 2 → Vector 2
   ├── Chunk 3 → Vector 3
   └── ...
```

Semua chunk tetap membawa metadata yang menunjuk ke dokumen asli.

Contoh metadata:

``` json
{
  "tenant_id": "company_001",
  "department_id": "IT",
  "document_id": "DOC-123",
  "document_version": 2,
  "filename": "Access-Control-Policy.pdf",
  "page": 8,
  "section": "Periodic Review",
  "chunk_id": "CHK-998"
}
```

### 6.3 Evidence Lintas File

Satu requirement dapat dipenuhi oleh: - satu chunk; - beberapa chunk
dalam satu file; - beberapa file berbeda.

Assessment engine harus mengagregasi evidence lintas file.

Jika Master Reference mensyaratkan evidence harus berada dalam satu
controlled document tetapi evidence aktual tersebar, sistem dapat
membedakan: - **content coverage**; - **document/control structure
compliance**.

------------------------------------------------------------------------

## 7. Qdrant & Retrieval Design

PostgreSQL adalah **source of truth**. Qdrant adalah **semantic
retrieval layer**.

Direkomendasikan minimal dua logical collection/index:

``` text
master_reference
company_documents
```

Master Reference dapat di-embedding untuk semantic discovery, tetapi
assessment tidak boleh bergantung hanya pada similarity antara dua
vector.

Flow utama:

``` text
PostgreSQL Requirement
        ↓
Generate Retrieval Query
        ↓
Qdrant Search
        ↓
Top Candidate Evidence
        ↓
LLM Evidence Verification
```

Qdrant menjawab:

> "Potongan dokumen mana yang kemungkinan relevan?"

Qdrant **tidak** menjawab:

> "Apakah perusahaan compliant?"

------------------------------------------------------------------------

## 8. Assessment Engine

### 8.1 Unit Eksekusi

Assessment harus dapat dijalankan dalam tiga mode:

1.  **Single** --- satu control.
2.  **Multiple** --- beberapa control yang dipilih.
3.  **Full** --- seluruh scope assessment.

Walaupun user memilih Full, backend tetap memproses secara **per-control
job** atau unit kecil yang retryable.

### 8.2 Pipeline Per Control

``` text
Load Master Control
        ↓
Load Requirements
        ↓
For Each Requirement
        ↓
Retrieve Evidence from Qdrant
        ↓
Rerank / Filter
        ↓
LLM Evidence Verification
        ↓
MET / PARTIAL / NOT_MET / NO_EVIDENCE
        ↓
Aggregate Criteria
        ↓
Deterministic Scoring
        ↓
Finding Rules
        ↓
Proposed Classification
        ↓
Persist Result
```

### 8.3 Status Requirement

Minimal:

-   `MET`
-   `PARTIAL`
-   `NOT_MET`
-   `NO_EVIDENCE`

### 8.4 Scoring

LLM tidak boleh bebas menghasilkan score.

LLM menghasilkan assessment per criterion; backend Golang menghitung
score berdasarkan weight/rules.

Contoh mapping default yang configurable:

``` text
MET         = 100% dari weight
PARTIAL     = 50% dari weight
NOT_MET     = 0%
NO_EVIDENCE = 0%
```

### 8.5 Finding Classification

Minimal:

-   `CONFORMITY`
-   `OBSERVATION`
-   `MINOR`
-   `MAJOR`

AI menghasilkan **proposed classification**. Auditor dapat menetapkan
final classification.

Contoh:

``` text
AI Proposed       : MAJOR
Auditor Final     : MINOR
Override          : true
Override Reason   : Evidence tambahan diverifikasi manual
Reviewed By       : auditor_id
Reviewed At       : timestamp
```

### 8.6 Explainability

Setiap kesimpulan harus dapat dijelaskan.

Contoh tampilan detail:

``` text
Requirement R3
Status: PARTIAL

Requirement:
Periodic review harus dilakukan dan terdokumentasi.

Evidence Found:
Access-Control-SOP.pdf
Page 7
Section 4.2

Covered:
✓ Proses review disebutkan
✓ PIC ditentukan

Missing:
✕ Tidak ada frekuensi review
✕ Tidak ditemukan record pelaksanaan

Proposed Finding:
MINOR

Must Fix:
1. Definisikan frekuensi review.
2. Sediakan evidence pelaksanaan review.
```

------------------------------------------------------------------------

## 9. Queue & Worker

Full assessment tidak boleh menjadi satu synchronous request besar.

Architecture:

``` text
User Run Assessment
        ↓
Create Assessment Run
        ↓
Create Control Jobs
        ↓
Queue
        ↓
Workers
        ↓
Persist Incremental Results
        ↓
Aggregate Progress
```

Status job minimal: - `QUEUED` - `PROCESSING` - `COMPLETED` - `FAILED` -
`RETRYING` - `CANCELLED`

Keuntungan: - timeout satu control tidak menggagalkan seluruh run; -
retry hanya job gagal; - progress dapat dilihat; - worker dapat
diskalakan; - hasil incremental tersimpan.

Contoh UI:

``` text
Assessment Progress
72 / 100 completed
2 failed
8 processing
18 queued
```

Redis dapat digunakan untuk queue/state/cache sesuai library queue yang
dipilih.

------------------------------------------------------------------------

## 10. Human Review & Comments

Internal auditor/consultant harus dapat:

-   comment per control;
-   comment per requirement;
-   review evidence;
-   approve AI finding;
-   override AI finding;
-   meminta revision;
-   menambahkan recommendation;
-   mencatat alasan keputusan.

Komentar dan perubahan assessment harus menjadi audit trail.

------------------------------------------------------------------------

## 11. Audit Copilot / Contextual Chat

Chat ditempatkan di bawah hasil assessment dan **terikat pada
control/result yang sedang dibuka**.

Context minimal:

``` text
tenant_id
assessment_id
assessment_version
control_id
master_reference_version
requirements
current_result
evidence
findings
comments
```

Contoh pertanyaan:

-   "Kenapa ini Minor?"
-   "Apa yang harus kami perbaiki?"
-   "Dokumen apa yang mendukung hasil ini?"
-   "Cari evidence tambahan."
-   "Apakah Risk Register ini cukup?"
-   "Berikan contoh isi yang seharusnya ada."
-   "Apa perbedaan hasil V1 dengan V2 untuk control ini?"

Chat dapat melakukan retrieval tambahan.

**Chat tidak boleh diam-diam mengubah assessment resmi.**

Jika evidence baru ditemukan:

``` text
New Evidence Found
        ↓
Suggest Reassessment
        ↓
User Confirm
        ↓
Run Control Assessment Again
```

------------------------------------------------------------------------

## 12. Remediation

Setiap gap harus menghasilkan actionable remediation.

Minimal hasil:

``` text
Finding
Missing Requirement
Evidence Gap
Impact/Reason
Must Fix
Recommendation
Owner (future)
Due Date (future)
Status (future)
```

Tujuannya agar client tidak hanya diberi tahu "tidak sesuai", tetapi
memahami **apa yang kurang dan apa yang perlu diperbaiki**.

------------------------------------------------------------------------

## 13. Versioning / Archive / Snapshot

Assessment yang sudah di-finalize dapat di-archive menjadi immutable
snapshot.

Contoh:

``` text
Assessment
 ├── V1
 ├── V2
 ├── V3
 └── External Audit Final
```

Snapshot harus menyimpan: - assessment scope; - master reference
version; - score per control; - requirement results; - evidence
references; - document versions yang digunakan; - findings; -
comments/reviews; - AI result; - auditor final decision; - timestamp; -
actor yang melakukan finalize.

Versi lama **tidak berubah** ketika dokumen perusahaan atau Master
Reference diperbarui.

------------------------------------------------------------------------

## 14. Version Comparison

Sistem membandingkan version saat ini dengan version sebelumnya.

Contoh:

``` text
                 V1      V2      V3
Overall Score    58%     76%     91%
Major              4       1       0
Minor             12       6       2
Observation        8       7       4
```

Comparison harus tersedia: - overall; - per control; - per
requirement; - finding resolved; - finding baru; - score
increase/decrease; - evidence baru/dihapus/berubah.

Gunakan istilah yang jelas: - `+18 percentage points`, bukan ambigu
"naik 18%".

------------------------------------------------------------------------

## 15. External Audit Final Snapshot

Setelah external audit selesai, hasil aktual dapat dicatat:

``` text
External Audit
Audit Date
Auditor/Organization
Major Findings
Minor Findings
Observations
Final Notes
Supporting Report
```

Tujuan: - historical record; - membandingkan internal prediction dengan
actual audit; - mengevaluasi kualitas assessment engine; - meningkatkan
Master Reference/rules.

------------------------------------------------------------------------

## 16. UI / UX Utama

Workspace desktop menggunakan konsep tiga panel.

``` text
┌────────────────────┬─────────────────────────────────────┬────────────────────┐
│ COMPANY DOCUMENTS  │       ASSESSMENT WORKSPACE          │ ISO SCOPE          │
│                    │                                     │ NAVIGATOR          │
│ + New Folder       │ Control / Clause                    │                    │
│ ↑ Upload           │ Score / Finding                     │ Clause 4           │
│                    │                                     │ Clause 5           │
│ 📁 IT              │ Requirement Coverage                │ ...                │
│ 📁 HR              │ Evidence                            │ Annex A            │
│ 📁 GA              │ Missing Items                       │  A.5               │
│ 📁 Sales           │ Must Fix                            │  A.6               │
│                    │ Recommendation                      │  A.7               │
│                    │                                     │  A.8               │
│                    ├─────────────────────────────────────┤                    │
│                    │ Audit Copilot Chat                  │                    │
└────────────────────┴─────────────────────────────────────┴────────────────────┘
```

### Panel Kiri --- Company Documents

Fungsi: - create folder; - upload; - browse; - preview; - search; -
version file; - department grouping.

### Panel Tengah --- Assessment Workspace

Menampilkan: - selected control; - score; - proposed/final finding; -
requirement breakdown; - evidence citations; - gap; - must fix; -
recommendation; - comments/review; - Audit Copilot.

### Panel Kanan --- Scope Navigator

Contoh:

``` text
ISO/IEC 27001:2022

Clause 4
  4.1   67%  MINOR
  4.2   100% CONFORMITY
  4.3   NOT TESTED

Annex A
  A.5
  A.6
  A.7
  A.8
    A.8.x  25% MAJOR
```

------------------------------------------------------------------------

## 17. Dashboard & Reporting

Summary dashboard minimal:

-   overall readiness score;
-   tested vs not tested;
-   Major count;
-   Minor count;
-   Observation count;
-   Conformity count;
-   progress antar-version;
-   department/document readiness;
-   controls needing remediation.

Visual kecil dapat berupa: - donut/pie distribution; - score
progression; - Major/Minor trend; - coverage percentage.

Report version harus dapat menampilkan: - Executive Summary; -
Assessment Scope; - Overall Score; - Result per Control; - Evidence; -
Non-Conformity; - Major/Minor; - Recommendations; - Version
Comparison; - Audit Trail.

------------------------------------------------------------------------

## 18. RBAC

Jangan mendesain `1 user = 1 role` secara hardcoded.

Model:

``` text
User
 ↓
User Role Assignment
 ↓
Role
 ↓
Role Permission
 ↓
Permission
```

Future scope dapat mendukung: - tenant/company scope; - department
scope; - workspace scope; - read/write/review/finalize permission.

### MVP/NPP

Gunakan:

``` text
SUPER_ADMIN / FULL_ACCESS
```

Satu user dapat: - upload; - manage master; - run assessment; -
review; - comment; - finalize; - archive; - report.

Namun schema RBAC tetap dibuat dari awal.

------------------------------------------------------------------------

## 19. Arsitektur Teknis

### 19.1 Technology Stack

``` text
Frontend
    │
    │ HTTP/gRPC Gateway as appropriate
    ▼
Golang Backend
    │
    ├── PostgreSQL
    ├── Qdrant
    ├── Redis
    ├── Object Storage
    ├── LLM Provider
    └── Embedding Provider
```

Internal ecosystem menggunakan gRPC sesuai kebutuhan.

### 19.2 Deployment Awal

Tidak perlu banyak microservice.

Dua aplikasi utama:

``` text
Frontend Application
Backend Application (Golang)
```

Backend menggunakan modular architecture:

``` text
/internal
  /auth
  /tenant
  /masterreference
  /documents
  /extraction
  /embedding
  /retrieval
  /assessment
  /scoring
  /findings
  /review
  /chat
  /versioning
  /reporting
  /jobs
```

Worker dapat berjalan sebagai process/deployment terpisah dari codebase
Golang yang sama.

------------------------------------------------------------------------

## 20. Source of Truth

### PostgreSQL

Menyimpan data authoritative: - users/roles; - company; - departments; -
standards; - controls; - master reference; - requirements; - master
reference versions; - document metadata; - extracted content metadata; -
assessment; - assessment results; - findings; - reviews/comments; -
snapshots; - audit trail.

### Object Storage

Menyimpan: - PDF; - DOCX; - file evidence asli; - report/artifact.

### Qdrant

Menyimpan: - vector Master Reference; - vector document chunks; -
metadata retrieval.

Qdrant bukan authoritative database.

### Redis

Digunakan untuk: - job queue; - transient worker state; - caching; -
distributed coordination sesuai kebutuhan.

------------------------------------------------------------------------

## 21. Suggested Core Data Model

``` text
companies
departments

users
roles
permissions
user_roles
role_permissions

standards
standard_sections
controls

master_references
master_reference_versions
requirements
requirement_expected_evidence
requirement_finding_rules

folders
documents
document_versions
document_extractions
document_chunks

assessments
assessment_runs
assessment_jobs

assessment_control_results
assessment_requirement_results
assessment_evidences
assessment_findings

assessment_comments
assessment_reviews

assessment_snapshots
assessment_snapshot_controls
assessment_snapshot_requirements
assessment_snapshot_evidences

chat_threads
chat_messages

external_audit_results

audit_logs
```

------------------------------------------------------------------------

## 22. Assessment State Model

Contoh lifecycle:

``` text
DRAFT
  ↓
QUEUED
  ↓
PROCESSING
  ↓
AI_COMPLETED
  ↓
UNDER_REVIEW
  ↓
REMEDIATION_REQUIRED
  ↓
REVIEWED
  ↓
FINALIZED
  ↓
ARCHIVED
```

Tidak semua state wajib digunakan pada MVP, tetapi model harus
memungkinkan workflow tersebut.

------------------------------------------------------------------------

## 23. Document Versioning

Dokumen perusahaan dapat berubah setelah remediation.

Contoh:

``` text
Access-Control-Policy.pdf
  ├── v1
  ├── v2
  └── v3
```

Assessment snapshot harus menunjuk ke **document_version** yang
digunakan saat assessment, bukan hanya `document_id`.

Dengan demikian V1 assessment tetap reproducible meskipun file sudah
diperbarui.

------------------------------------------------------------------------

## 24. Master Reference Versioning

Master Reference yang diedit consultant juga harus versioned.

Contoh:

``` text
Control A.x.x
  ├── Master Rev 1
  ├── Master Rev 2
  └── Master Rev 3
```

Assessment snapshot menyimpan Master Reference revision yang digunakan.

------------------------------------------------------------------------

## 25. Audit Trail

Event penting wajib dicatat:

-   file uploaded;
-   file version created;
-   file deleted/archived;
-   assessment started;
-   assessment completed;
-   assessment failed/retried;
-   AI finding generated;
-   auditor comment;
-   classification overridden;
-   assessment finalized;
-   snapshot created;
-   report generated;
-   external audit result recorded.

Minimal audit log:

``` text
actor
action
entity_type
entity_id
old_value
new_value
timestamp
correlation_id
```

------------------------------------------------------------------------

## 26. AI Guardrails

1.  AI tidak menentukan certification status.
2.  AI tidak boleh membuat evidence yang tidak ditemukan.
3.  Setiap evidence harus memiliki source reference.
4.  Jika evidence tidak cukup, gunakan `NO_EVIDENCE`/`PARTIAL`; jangan
    berasumsi.
5.  Score dihitung deterministic oleh backend.
6.  Major/Minor dari AI adalah proposed classification sampai direview.
7.  Chat tidak dapat mengubah official result tanpa explicit
    reassessment/review.
8.  Prompt dan Master Reference version harus dapat ditelusuri untuk
    audit/debugging.

------------------------------------------------------------------------

## 27. Retrieval Quality

Semantic similarity saja tidak cukup.

Recommended pipeline:

``` text
Requirement
   ↓
Query Expansion
   ↓
Qdrant Top-K
   ↓
Metadata Filtering
   ↓
Optional Reranking
   ↓
Evidence Verification by LLM
```

Filter minimal: - tenant/company; - active document version; -
assessment scope; - optional department/document type.

Sistem harus menghindari evidence lintas tenant.

------------------------------------------------------------------------

## 28. POC / Research Plan

Jangan langsung implementasi seluruh standard.

Mulai dengan **5--10 controls/clauses** yang dipilih bersama consultant.

Dataset uji minimal:

### Case A --- Complete

Expected: `MET`.

### Case B --- Partial

Dokumen ada tetapi beberapa requirement tidak tercakup.

Expected: `PARTIAL` + proposed Minor sesuai rule.

### Case C --- Critical Gap

Requirement kritikal tidak dipenuhi.

Expected: `NOT_MET` + proposed Major sesuai rule.

### Case D --- No Evidence

Tidak ada evidence relevan.

Expected: `NO_EVIDENCE`.

### Case E --- Multi-Document Evidence

Coverage tersebar di beberapa dokumen.

Expected: engine menggabungkan evidence dengan benar.

### Case F --- Similar Text but Wrong Evidence

Dokumen semantically mirip tetapi tidak membuktikan requirement.

Expected: LLM verifier menolak candidate evidence.

------------------------------------------------------------------------

## 29. POC Evaluation Metrics

Bandingkan hasil AI dengan manual assessment consultant.

Ukuran utama:

1.  **Evidence Retrieval Accuracy**
    -   Apakah evidence yang benar masuk Top-K?
2.  **Requirement Classification Accuracy**
    -   MET/PARTIAL/NOT_MET/NO_EVIDENCE.
3.  **Finding Agreement**
    -   Agreement proposed Major/Minor/Observation terhadap auditor.
4.  **Citation Accuracy**
    -   Apakah halaman/section/chunk benar?
5.  **False Evidence Rate**
    -   Berapa kali sistem menganggap teks relevan sebagai bukti padahal
        bukan?
6.  **Explanation Quality**
    -   Apakah missing requirement dan must-fix dapat ditindaklanjuti?
7.  **Processing Reliability**
    -   timeout/failure/retry rate per job.

------------------------------------------------------------------------

## 30. MVP/NPP Scope

### Must Have

-   Login satu full-access user.
-   Master Reference CRUD.
-   Requirement decomposition.
-   Criticality + weight + expected evidence.
-   Master Reference embedding.
-   Folder management.
-   PDF/DOCX upload.
-   Object storage.
-   Text extraction.
-   Chunking.
-   Embedding document.
-   Qdrant search.
-   Single assessment.
-   Multiple assessment.
-   Full assessment via queue.
-   Incremental job progress.
-   MET/PARTIAL/NOT_MET/NO_EVIDENCE.
-   Deterministic score.
-   Proposed Conformity/Observation/Minor/Major.
-   Evidence citation.
-   Missing requirements.
-   Must Fix.
-   Recommendation.
-   Review/comment.
-   Contextual chat.
-   Reassessment.
-   Finalize/archive.
-   Snapshot V1/V2.
-   Version comparison.
-   Basic dashboard/report.
-   Audit log.

### Should Have

-   document versioning;
-   Master Reference versioning;
-   manual finding override;
-   retry failed jobs;
-   external audit final snapshot;
-   downloadable report.

### Later

-   full department RBAC;
-   task assignment/remediation owner;
-   due dates;
-   notification;
-   approval workflow;
-   external auditor portal;
-   multiple ISO standards;
-   advanced analytics;
-   model quality feedback loop.

------------------------------------------------------------------------

## 31. End-to-End User Journey

``` text
1. Consultant creates Master Reference
        ↓
2. Master saved to PostgreSQL
        ↓
3. Master embedded to Qdrant
        ↓
4. Client uploads company documents
        ↓
5. Original files stored
        ↓
6. Content extracted and stored
        ↓
7. Content chunked + embedded
        ↓
8. User selects control(s)
        ↓
9. Assessment jobs created
        ↓
10. Worker retrieves evidence
        ↓
11. LLM verifies requirement coverage
        ↓
12. Golang engine calculates score
        ↓
13. Proposed findings generated
        ↓
14. Internal auditor reviews/comments
        ↓
15. Client performs remediation
        ↓
16. Reassessment
        ↓
17. Admin/MR finalizes
        ↓
18. Archive Snapshot V1/V2/V3
        ↓
19. Compare versions
        ↓
20. External audit
        ↓
21. Record Final External Audit Snapshot
```

------------------------------------------------------------------------

## 32. Acceptance Criteria Utama

Produk MVP dianggap berhasil jika:

1.  User dapat upload PDF/DOCX dan file dapat diproses menjadi
    searchable chunks.
2.  Satu Master Reference control dapat memiliki beberapa requirement.
3.  Satu requirement dapat menemukan evidence dari satu atau beberapa
    file.
4.  Sistem dapat menunjukkan file + page/section/chunk yang digunakan
    sebagai evidence.
5.  Sistem membedakan retrieval relevance dari actual requirement
    fulfillment.
6.  Score dihasilkan oleh rule engine, bukan angka bebas dari LLM.
7.  Sistem dapat menghasilkan proposed Minor/Major dan alasannya.
8.  Auditor dapat mengubah proposed result dengan alasan.
9.  Full assessment tidak gagal total ketika satu control timeout.
10. Job gagal dapat di-retry secara individual.
11. Hasil assessment dapat di-finalize menjadi immutable snapshot.
12. Snapshot baru dapat dibandingkan dengan snapshot sebelumnya.
13. Dokumen/master yang berubah tidak mengubah historical snapshot.
14. Chat memahami control/result yang sedang dibuka.
15. Chat tidak mengubah official assessment tanpa workflow eksplisit.
16. Seluruh tindakan penting tercatat di audit log.

------------------------------------------------------------------------

## 33. Product Principle

Produk harus selalu dapat menjawab lima pertanyaan untuk setiap control:

> **1. ISO meminta apa?**\
> **2. Evidence perusahaan ada di mana?**\
> **3. Bagian mana yang sudah terpenuhi?**\
> **4. Bagian mana yang masih kurang?**\
> **5. Apa yang harus dilakukan perusahaan untuk memperbaikinya?**

Kemudian sistem harus dapat menunjukkan bagaimana jawaban tersebut
berubah dari:

``` text
V1 → Remediation → V2 → Remediation → V3 → External Audit Final
```

Itulah core value dari ISO Assessment & Audit Readiness Platform ini.
