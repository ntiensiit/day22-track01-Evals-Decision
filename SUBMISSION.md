# Bài làm — Day 22 Track 1: Eval Decision Workbook

Repository này giữ nguyên ba đề bài gốc và bổ sung bài làm tại thư mục [`answers/`](./answers/).

## Nội dung đã hoàn thành

| Case | File bài làm | Trọng tâm quyết định |
| --- | --- | --- |
| 1. Support Ticket Triage | [`answers/01-ticket-triage-answer.md`](./answers/01-ticket-triage-answer.md) | Route, urgency, escalation và release gate vận hành |
| 2. Sales Chat Copilot | [`answers/02-sales-copilot-answer.md`](./answers/02-sales-copilot-answer.md) | Lookup an toàn, ambiguity, privacy và human-in-the-loop |
| 3. Medical Call Routing Copilot | [`answers/03-medical-routing-answer.md`](./answers/03-medical-routing-answer.md) | Safety routing, domain-expert review và clinical release gate |

## Nguyên tắc dùng xuyên suốt

1. **Code trước** cho schema, enum, permission, lookup integrity, tool calls, privacy và các ngưỡng vận hành.
2. **LLM-as-judge đã calibration** cho groundedness, summary, ambiguity handling và mức hợp lý của recommendation.
3. **Human review** cho case low-confidence, conflict, high-risk, user complaint và judge disagreement.
4. **Domain expert** chỉ bắt buộc ở case y tế; không được dùng AI để chẩn đoán hoặc thay thế quyết định chuyên môn.
5. Mọi thay đổi prompt/model/tool phải chạy offline eval và qua release gate trước khi ship.

## Giả định chung cho dự toán

- Model dùng để tính chi phí: **GPT-5 mini**.
- Giá planning: **USD 0.25 / 1M input tokens** và **USD 2.00 / 1M output tokens**. Giá cần được kiểm tra lại trên trang pricing của nhà cung cấp tại thời điểm mua.
- Quy đổi planning: **1 USD = 26,000 VND**; đây là giả định ngân sách nội bộ, không phải tỷ giá thị trường.
- Nhân sự được tính theo giờ công nội bộ; các đơn giá là giả định để xin pilot, không phải báo giá thị trường.

Mục tiêu của pilot không phải chứng minh AI “hoàn hảo”, mà chứng minh được giới hạn chất lượng hiện tại, failure modes trọng yếu, và điều kiện tối thiểu để có thể tiến tới triển khai có kiểm soát.
