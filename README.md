# AI Quality Intelligence Platform
> Dự án AI20K-223 — Phân tích phản hồi & nâng chất lượng dịch vụ cho hãng gọi xe

## Nhóm

| Thành viên | MSSV | 
|---|---|
| Lê Quốc Anh | 2A202600824 | 
| Nguyễn Đức Khang | 2A202600588 | 
| Nguyễn Đức Mạnh | 2A202600945 |

## Tổng quan

Hệ thống tự động phân tích hàng nghìn phản hồi khách hàng mỗi ngày, phát hiện vấn đề nổi lên theo khu vực/thời gian và định tuyến xử lý đúng phòng ban — giúp đội ops phát hiện sự cố trong **< 15 phút** thay vì 1 ngày đọc thủ công.

## Pipeline

```
Feedback (CSV/webhook)
  → Cleaning (PII mask, dedup)
  → ML Classify (TF-IDF + LinearSVC): topic, sentiment, severity
  → Spike Detection (SQL): so sánh vs baseline 7 ngày
  → LLM Agent (Gemini Flash): insight + ticket có evidence
  → Dashboard (React/TS): ops team xem và xử lý
```

## Tech Stack

| Layer | Tech |
|---|---|
| ML Pipeline | Python, scikit-learn, underthesea, APScheduler |
| LLM | Gemini Flash API |
| Database | Supabase (PostgreSQL) |
| Frontend | TypeScript, React, Tailwind CSS |
| Deploy | Vercel + Railway |

## Cấu trúc repo

```
/
├── pipeline/          # Python ML pipeline
│   ├── clean.py       # ETL: PII mask, dedup, normalize
│   ├── classify.py    # TF-IDF + LinearSVC + safety rules
│   ├── spike.py       # Spike detection logic
│   ├── llm_agent.py   # Gemini Flash integration
│   └── scheduler.py   # APScheduler batch job (10 phút)
├── frontend/          # React TypeScript app
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Alerts.tsx
│   │   │   ├── Tickets.tsx
│   │   │   └── Chat.tsx
│   │   └── components/
├── supabase/
│   └── migrations/    # DB schema migrations
├── docs/
│   ├── BRIEF.md
│   ├── PRD.md
│   └── WIREFRAME.svg
└── README.md
```

## Setup

### 1. Clone repo
```bash
git clone <repo-url>
```

### 2. Pipeline
```bash
cd pipeline
pip install -r requirements.txt
cp .env.example .env   # điền SUPABASE_URL, SUPABASE_KEY, GEMINI_API_KEY
python scheduler.py
```

### 3. Frontend
```bash
cd frontend
npm install
cp .env.example .env   # điền VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY
npm run dev
```

## Deliverables

- [BRIEF.md](docs/BRIEF.md) — 1-page tóm tắt dự án
- [PRD.md](docs/PRD.md) — Product Requirements Document
- [WIREFRAME.svg](docs/WIREFRAME.svg) — UI Flow diagram
