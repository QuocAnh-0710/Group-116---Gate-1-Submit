# BRIEF — AI Quality Intelligence Platform
**Dự án:** AI20K-223 | **Nhóm:** Lê Quốc Anh · Nguyễn Đức Khang · Nguyễn Đức Mạnh

---

## Bài toán

Hãng gọi xe nhận hàng nghìn đánh giá mỗi ngày (app, hotline, review, survey) nhưng đội vận hành không thể đọc thủ công toàn bộ. Vấn đề lặp lại bị bỏ sót đến khi thành khủng hoảng — rating tụt, khách hàng rời bỏ — trước khi đội ops kịp phát hiện.

## Giải pháp

**AI Quality Intelligence Platform** — hệ thống tự động phân tích phản hồi khách hàng, phát hiện vấn đề nổi lên theo khu vực/thời gian, định tuyến xử lý đúng phòng ban và theo dõi SLA.

Pipeline: **Thu thập → Làm sạch → ML phân loại (TF-IDF + LinearSVC) → Spike detection → LLM Agent (Gemini Flash) sinh insight + ticket → Dashboard ops.**

## Khách hàng mục tiêu

Đội vận hành (Ops Manager, CS Team, Safety Team) của các hãng gọi xe / logistics vừa và nhỏ tại Việt Nam.

## Giá trị cốt lõi

- Phát hiện spike vấn đề trong **< 15 phút** thay vì 1 ngày đọc thủ công
- Safety feedback → **severity 5 + human review bắt buộc**, không để AI tự quyết
- LLM insight **có evidence_ids**, không bịa nguyên nhân
- Vòng lặp khép kín: correction log → nguyên liệu cải thiện model

## Tech stack

| Layer | Công nghệ |
|---|---|
| ML Pipeline | Python, TF-IDF + LinearSVC, APScheduler |
| LLM Agent | Gemini Flash API |
| Database | Supabase (PostgreSQL) |
| Frontend | TypeScript, React |
| Deploy | Vercel (frontend) · Railway (pipeline) |

## Thành viên

| Thành viên | MSSV | Role |
|---|---|---|
| Lê Quốc Anh | 2A202600824 | AI/Data — ML pipeline, NLP, LLM prompt |
| Nguyễn Đức Khang | 2A202600588 | Backend — Supabase schema, API, scheduler |
| Nguyễn Đức Mạnh | 2A202600945 | Frontend — Dashboard, Alerts, Tickets UI |
