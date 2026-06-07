# PRD — AI Quality Intelligence Platform
**Dự án:** AI20K-223 | **Version:** 1.0 | **Ngày:** 07/06/2025
**Nhóm:** Lê Quốc Anh (2A202600824) · Nguyễn Đức Khang (2A202600588) · Nguyễn Đức Mạnh (2A202600945)

---

## 1. Mục tiêu sản phẩm

Xây dựng nền tảng phân tích phản hồi khách hàng tự động cho hãng gọi xe, giúp đội vận hành **phát hiện vấn đề chất lượng dịch vụ sớm hơn 10×** so với đọc thủ công, và theo dõi xử lý đến khi resolved.

Sản phẩm phải trả lời được 5 câu hỏi vận hành:
1. Khách hàng đang phàn nàn nhiều nhất về điều gì?
2. Vấn đề đó xảy ra ở đâu, khi nào, liên quan tài xế/khu vực nào?
3. Vấn đề nào nghiêm trọng nhất, cần xử lý ngay?
4. Ai đang chịu trách nhiệm xử lý và đã xử lý đúng SLA chưa?
5. Sau khi xử lý, chất lượng có cải thiện không?

---

## 2. Người dùng mục tiêu

| Role | Pain point | Dùng tính năng nào |
|---|---|---|
| Ops Manager | Không biết khu vực nào đang xuống chất lượng | Dashboard, Alerts, Báo cáo |
| CS Team | Đọc thủ công không ưu tiên được case nghiêm trọng | Human Review, Correction |
| Safety Team | Có thể bỏ sót phản hồi nguy hiểm | Alert severity 5, Human Review bắt buộc |
| Driver Management | Không biết tài xế nào có pattern xấu | Dashboard drill-down, Tickets |

---

## 3. Kiến trúc hệ thống

### 3.1 Python Pipeline (batch, chạy offline mỗi 10 phút)

```
[Feedback Input]
  source: app / hotline / review / survey
  fields: text, area, driver_id, timestamp
       │
       ▼
[① INGESTION]
  Upload CSV hoặc POST /ingest webhook
       │
       ▼
[② CLEANING — ETL mini]
  - Bỏ spam, khử trùng lặp
  - PII mask: regex SĐT/email → [MASKED]
  - Chuẩn hóa text tiếng Việt
       │
       ├──► ghi vào feedback_raw (bất biến)
       │
       ▼
[③ ML PHÂN LOẠI — TF-IDF + LinearSVC]
  - topic: 10 nhóm (Driver, Safety, App, Pricing,
           Delay, Vehicle, Payment, Route, CS, Other)
  - sentiment: score -1.0 → +1.0
  - severity: 1–5 (Safety keyword → hardcode 5)
  - confidence score → needs_review nếu < 0.6
       │
       ├──► ghi vào feedback_processed
       │
       ▼
[④ SPIKE DETECTION — SQL aggregate]
  - Đếm negative theo (area, category) trong 1h gần nhất
  - So với baseline trung bình 7 ngày
  - Nếu > 2× baseline → tạo alert + evidence_ids
       │
       ├──► ghi vào alerts
       │
       ▼
[⑤ LLM AGENT — Gemini Flash]
  Input: alert + aggregated stats (không phải raw text)
  Output JSON: { team, priority, sla, action_draft, evidence_ids }
  - Route đúng phòng ban
  - Set SLA tự động
  - Draft hành động đề xuất
  - Evidence IDs bắt buộc (chống bịa)
       │
       └──► ghi vào tickets
```

**Trục nối: Supabase (PostgreSQL) — Python ghi, Frontend đọc**

### 3.2 Frontend (TypeScript + React, chỉ đọc DB)

| Route | Nội dung |
|---|---|
| `/dashboard` | KPI cards, sentiment trend chart, heatmap khu vực |
| `/alerts` | Danh sách spike alerts, evidence feedback, human review, sửa nhãn |
| `/tickets` | Giao phòng ban, SLA countdown, status workflow |
| `/chat` | RAG insight: hỏi tự nhiên, trả lời có trích dẫn evidence |

---

## 4. Database Schema

### feedback_raw
| Cột | Kiểu | Mô tả |
|---|---|---|
| id | uuid | PK |
| source | text | app / hotline / review / survey |
| text | text | Nội dung gốc (sau PII mask) |
| area | text | Khu vực địa lý |
| driver_id | text | Mã tài xế (nullable) |
| rating | int | 1–5 sao (nullable) |
| created_at | timestamptz | Thời điểm phản hồi |

### feedback_processed
| Cột | Kiểu | Mô tả |
|---|---|---|
| id | uuid | PK |
| feedback_id | uuid | FK → feedback_raw |
| topic | text | 1 trong 10 nhóm |
| subtopic | text | Nhãn phụ |
| sentiment | float | -1.0 đến +1.0 |
| severity | int | 1–5 |
| urgency | text | now / 24h / week / backlog |
| confidence | float | 0.0–1.0 |
| needs_review | bool | True nếu confidence < 0.6 hoặc Safety |
| model_version | text | Version classifier |

### alerts
| Cột | Kiểu | Mô tả |
|---|---|---|
| id | uuid | PK |
| alert_type | text | spike / safety / pattern |
| topic | text | Category liên quan |
| area | text | Khu vực xảy ra |
| severity | int | 1–5 |
| evidence_ids | uuid[] | List feedback_id làm bằng chứng |
| status | text | new / assigned / resolved |
| created_at | timestamptz | |

### tickets
| Cột | Kiểu | Mô tả |
|---|---|---|
| id | uuid | PK |
| alert_id | uuid | FK → alerts |
| owner_team | text | safety / cs / ops / product |
| assignee | text | Email người được giao |
| due_at | timestamptz | SLA deadline |
| status | text | new / investigating / resolved / closed |
| resolution_note | text | Ghi chú xử lý |
| action_draft | text | LLM đề xuất hành động |

### correction_logs
| Cột | Kiểu | Mô tả |
|---|---|---|
| id | uuid | PK |
| feedback_id | uuid | FK → feedback_processed |
| old_label | text | Nhãn AI gán |
| new_label | text | Nhãn nhân viên sửa |
| corrected_by | text | User ID |
| reason | text | Lý do sửa |
| created_at | timestamptz | |

### users
| Cột | Kiểu | Mô tả |
|---|---|---|
| id | uuid | PK |
| email | text | |
| role | text | admin / ops / analyst / viewer |
| team | text | safety / cs / product / ops |
| active | bool | |

---

## 5. Tính năng chi tiết

### 5.1 ML Classifier
- **Model:** TF-IDF (unigram + bigram) + LinearSVC
- **NLP tiếng Việt:** underthesea tokenize trước khi vectorize
- **Safety rule cứng:** keyword list (vượt đèn, quấy rối, tai nạn, say rượu...) → severity = 5 + needs_review = True, bất kể model output
- **Confidence:** probability estimate từ LinearSVC decision function → normalize 0–1
- **Retrain:** dùng correction_logs làm labeled data bổ sung

### 5.2 Spike Detection
```sql
WITH recent AS (
  SELECT area, topic, COUNT(*) as cnt
  FROM feedback_processed
  WHERE sentiment < -0.3
    AND created_at > NOW() - INTERVAL '1 hour'
  GROUP BY area, topic
),
baseline AS (
  SELECT area, topic,
    COUNT(*) / 7.0 as daily_avg
  FROM feedback_processed
  WHERE sentiment < -0.3
    AND created_at BETWEEN NOW() - INTERVAL '8 days'
                       AND NOW() - INTERVAL '1 day'
  GROUP BY area, topic
)
SELECT r.area, r.topic, r.cnt, b.daily_avg
FROM recent r JOIN baseline b USING (area, topic)
WHERE r.cnt > b.daily_avg * 2.0
```

### 5.3 LLM Agent — Gemini Flash
- **Input:** Alert metadata + aggregated stats (không raw text → tiết kiệm token)
- **Prompt structure:** Role → Context → Task → Output format (JSON strict)
- **Output bắt buộc:** evidence_ids array — nếu không có evidence thì output `"insufficient_data": true`
- **SLA tự động:** Safety → 2h · Payment → 24h · App → 3 ngày · Driver → 48h

### 5.4 Human Review (Correction Loop)
- Feedback có `needs_review = True` → hiển thị trong `/alerts` với badge
- Nhân viên xem text gốc + AI label + confidence
- Nhấn Accept (giữ nguyên) hoặc Edit (chọn nhãn đúng + nhập lý do)
- Mọi edit → ghi vào `correction_logs`
- Correction logs dùng làm training data cho lần retrain tiếp theo

### 5.5 RAG Chat (/chat)
- Index `feedback_processed` + `alerts` vào vector store
- Ops Manager hỏi: *"Vì sao rating giảm ở quận 7 tuần này?"*
- Trả lời kèm trích dẫn feedback IDs cụ thể

---

## 6. Phân quyền

| Role | Quyền |
|---|---|
| admin | Full access, quản lý user |
| ops | Xem dashboard, đọc báo cáo, assign ticket |
| analyst | Upload data, human review, edit label |
| viewer | Chỉ đọc dashboard |

---

## 7. Metrics đánh giá

| Metric | Mục tiêu |
|---|---|
| Topic F1 | ≥ 0.80 |
| Safety Recall | ≥ 0.95 (lỗi đắt nhất) |
| Sentiment Accuracy | ≥ 0.85 |
| Time-to-Alert | < 15 phút |
| SLA Compliance | ≥ 90% với alert severity cao |

---

## 8. MVP Scope

**Trong MVP (4 tuần):**
- Upload CSV + webhook ingest
- ML pipeline: clean → classify → severity → spike detection
- Gemini Flash: alert → ticket với evidence
- Dashboard: KPI + trend + heatmap
- Alerts: human review + correction log
- Tickets: SLA + status
- Auth: 4 roles
- Deploy online (Vercel + Railway + Supabase)

**Ngoài MVP (sau):**
- RAG chat (/chat)
- Fine-tune classifier từ correction logs
- Real-time streaming (Kafka)
- Driver risk score analytics
- Multilingual support
