# Bài làm — Case 1: Support Ticket Triage

## 1. Unit of Work

Tôi chọn một đơn vị công việc là: **một ticket mới đi vào inbox, AI tạo quyết định triage gồm category, urgency, route, requires_human và lý do ngắn**. Output này phục vụ nhân viên support và workflow queue nội bộ, không gửi trực tiếp cho khách hàng.

Lát cắt này đủ nhỏ để đánh giá vì mỗi field đều dẫn tới một hành động vận hành cụ thể: ticket vào team nào, có được đẩy ưu tiên hay không, và có cần người thật tiếp nhận ngay hay không. Nếu sai, ticket enterprise hoặc ticket đang chặn công việc có thể bị trì hoãn hoặc bị xử lý bởi team không phù hợp.

## 2. Quality Question

**AI có tạo triage đúng, có bằng chứng từ ticket và kích hoạt route/escalation phù hợp để ticket không đi sai hàng xử lý hoặc bị bỏ sót khi có dấu hiệu chặn công việc không?**

Hành vi bắt buộc là gắn được route và escalation nhất quán với nội dung, tier khách hàng và business rules. Hành vi bị cấm là tự bịa nguyên nhân, đánh ticket có tín hiệu `locked out`, `account disabled` hoặc `blocking work` thành low, hoặc để enterprise ticket high/critical không có human review.

## 3. Output Contract tối thiểu

| Field | Kiểu / giá trị gợi ý | Vì sao cần |
| --- | --- | --- |
| `ticket_id` | string | Liên kết kết quả triage với ticket và trace để audit. |
| `category` | enum: `technical`, `billing`, `account_access`, `product_question`, `unknown` | Hiển thị nhãn trên UI và là input chính cho routing. |
| `urgency` | enum: `low`, `medium`, `high`, `critical` | Quyết định queue, SLA và mức ưu tiên. |
| `route_to` | enum: `support_l1`, `technical_support`, `billing_ops`, `human_escalation` | Hành động vận hành trực tiếp; không dùng text tự do. |
| `requires_human` | boolean | Trigger checkpoint người thật, đặc biệt với enterprise/high-risk. |
| `queue` | enum: `normal`, `priority` | Render queue trên UI và kiểm tra mapping với urgency. |
| `summary` | string ngắn | Cho agent nhìn nhanh vấn đề chính trước khi mở ticket. |
| `reason_codes` | list enum, ví dụ `billing_failed`, `locked_out`, `blocking_work`, `enterprise_customer`, `insufficient_info` | Giải thích có cấu trúc để audit và kiểm tra rule tự động. |
| `evidence_spans` | list trích đoạn hoặc vị trí trong `subject/message` | Buộc lý do phải bám input; hỗ trợ kiểm tra groundedness. |
| `confidence` | float 0–1 | Dùng để route low-confidence sang human review, không phải để thay thế rule an toàn. |
| `prompt_version`, `model_version`, `trace_id` | string | Reproduce lỗi, so sánh prompt/model và rollback khi regression. |

## 4. Eval Decision Map

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema, enum, kiểu dữ liệu, `confidence` | ✓ |  |  |  | Có hợp đồng xác định; code nhanh, rẻ và reproducible. |
| Tính nhất quán `category → route_to → queue` | ✓ |  |  |  | Mapping đã được định nghĩa trước trong workflow nội bộ. |
| Hard escalation rule cho enterprise + high/critical | ✓ |  | ✓ |  | Code chặn lỗi rõ ràng; human audit để phát hiện rule thiếu hoặc taxonomy không còn phù hợp. |
| Nhận diện semantic category/urgency từ ngôn ngữ tự nhiên |  | ✓ | ✓ |  | Từ đồng nghĩa, tiếng Việt không dấu và ngữ cảnh làm rule thuần khó bao phủ; human dùng để calibrate judge. |
| `summary`, `reason_codes` có support bởi ticket | ✓ một phần | ✓ | ✓ |  | Code kiểm tra evidence span tồn tại; judge kiểm tra evidence có thật sự nâng đỡ kết luận. |
| Xử lý ambiguity/thiếu thông tin |  | ✓ | ✓ |  | Cần hiểu ticket có đủ tín hiệu để triage hay nên route review/clarification. |
| Tính chấp nhận được của flow trong queue thật |  |  | ✓ |  | Support ops biết ticket nào làm đội xử lý bị nghẽn hoặc sai SLA. |

## 5. Kiểm tra tự động bằng code

- **Kiểm tra JSON/schema, required fields và enum hợp lệ.**  
  Vì sao nên giao cho code: đây là invariant kỹ thuật; output không parse được không được đi vào queue.
- **`confidence` nằm trong [0, 1] và không phải `NaN`.**  
  Vì sao nên giao cho code: ngưỡng số xác định.
- **`ticket_id` khớp ticket đầu vào và có `trace_id`, `model_version`, `prompt_version`.**  
  Vì sao nên giao cho code: bảo đảm auditability và chống ghi nhầm kết quả sang ticket khác.
- **`category` phải map tới `route_to` được phép.** Ví dụ `billing` không được route `product_team`; `technical` không route `billing_ops`.  
  Vì sao nên giao cho code: bảng mapping là deterministic.
- **`urgency=high|critical` phải có `queue=priority`; `requires_human=true` phải có `urgency=high|critical`.**  
  Vì sao nên giao cho code: consistency rule rõ.
- **Nếu `customer_tier=enterprise` và urgency là high/critical thì `requires_human=true`.**  
  Vì sao nên giao cho code: đây là business rule được cho sẵn.
- **Nếu ticket chứa các tín hiệu đã chuẩn hóa `blocking work`, `locked out`, `account disabled` thì không được gắn `urgency=low`.**  
  Vì sao nên giao cho code: đây là hard safety floor; danh sách pattern được versioned và review định kỳ.
- **Nếu `reason_codes` chứa `locked_out`, `account_disabled` hoặc `billing_failed`, phải có ít nhất một `evidence_span` khớp subject/message.**  
  Vì sao nên giao cho code: kiểm tra truy vết bằng chứng cơ bản.
- **Không lộ PII không cần thiết trong `summary` hoặc `reason_codes`.** Ví dụ không in email, số thẻ, token, UUID nội bộ.  
  Vì sao nên giao cho code: regex/redaction policy phù hợp với hard rule.
- **Độ dài summary trong giới hạn 20–300 ký tự và không rỗng.**  
  Vì sao nên giao cho code: UI và thao tác vận hành cần output đủ ngắn, đủ thông tin.
- **Không được tự thực hiện action ngoài phạm vi.** Output chỉ là triage; không chứa lệnh đóng ticket, hoàn tiền hoặc gửi email.  
  Vì sao nên giao cho code: allowlist action/tool rõ.
- **Latency P95 < 2 giây, tỷ lệ lỗi kỹ thuật < 1%, cost/case < USD 0.004 ở pilot.**  
  Vì sao nên giao cho code: metric observability có ngưỡng cụ thể.
- **Regression suite không được làm fail các case đã từng pass ở nhóm P0/P1.**  
  Vì sao nên giao cho code: so sánh baseline tự động trước release.

## 6. Tiêu chí chấm bằng LLM-as-Judge

Judge chỉ chạy sau hard checks và được calibration với human labels.

- **Category có phản ánh đúng intent chính của ticket không?**  
  Vì sao code không bắt tốt: một ticket có thể nói gián tiếp, dùng tiếng Việt không dấu, hoặc chứa nhiều intent.
- **Urgency có phản ánh mức độ gián đoạn công việc và hậu quả khách hàng mô tả không?**  
  Vì sao code không bắt tốt: cùng một từ “gấp” không luôn là critical; judge cần đọc toàn bộ ngữ cảnh.
- **Route có hợp lý với category, urgency và customer tier không?**  
  Vì sao code không bắt tốt: những case biên như billing + access issue cần xác định vấn đề chính và route phù hợp.
- **Summary có đúng trọng tâm, đủ hành động và không làm nhẹ mức nghiêm trọng không?**  
  Vì sao code không bắt tốt: chất lượng tóm tắt là semantic quality.
- **Reason/evidence có grounded, không suy diễn thêm account history, outage hoặc lỗi hệ thống chưa được nêu không?**  
  Vì sao code không bắt tốt: kiểm tra span tồn tại không đủ để đánh giá quan hệ ngữ nghĩa giữa evidence và kết luận.
- **Ticket mơ hồ có được xử lý bằng `unknown`/review thay vì tự tin gán route không?**  
  Vì sao code không bắt tốt: thiếu tín hiệu có nhiều cách biểu đạt.

Judge output nên trả JSON gồm `label`, `failure_modes`, `severity`, `critique`, `confidence`; với `confidence < 0.75` hoặc `missed_escalation`, case bắt buộc chuyển human review.

## 7. Human / Expert Review

**Human review:** Support Operations Lead và QA Support review các case high/critical, low-confidence, LLM judge disagreement, case mới từ production, ticket bị thumbs-down/reopen, và toàn bộ P0/P1 failure. Họ xác nhận taxonomy có phù hợp với queue thực tế không, SLA có bị đe dọa không, và rule nào cần bổ sung vào dataset.

**Domain expert:** Không áp dụng. Đây là bài toán vận hành support/SaaS; taxonomy và SLA thuộc quyền quyết định của Support Ops/Product Owner. Human review vận hành là đủ, miễn là người review có quyền thay đổi queue policy và xác nhận impact thực tế.

### 7A. Màn hình Domain Expert

Không áp dụng vì case không cần domain expert chuyên sâu.

### 7B. Tiêu chí Domain Expert

Không áp dụng.

## 8. Release Gate

### Block release ngay khi

- Có **bất kỳ P0**: ticket enterprise có dấu hiệu account disabled/blocked work nhưng không escalation; billing bị route sang product; output lộ dữ liệu nhạy cảm; schema không parse được trên critical case.
- `schema_pass_rate < 99.5%` trên full reference dataset.
- `critical_escalation_recall < 100%` trên bộ 20 case red-flag/enterprise đã được human xác nhận.
- `enterprise_high_or_critical_routing_accuracy < 97%`.
- `groundedness_pass_rate < 90%` theo LLM judge đã calibration, hoặc có hallucination severity P1 trở lên.
- Có regression P0/P1 so với baseline production.

### Chỉ ship khi

- Macro category/route accuracy ≥ 92% trên toàn bộ dataset và ≥ 97% trên segment enterprise high/critical.
- `requires_human` recall ≥ 98% ở segment high-risk; false negative phải được báo cáo riêng, không bị che bởi overall average.
- 100% case low-confidence, disagreement hoặc safety exception được gắn `human_review_required=true`.
- Latency P95 < 2 giây; chi phí trung bình ≤ USD 0.004/case; tỷ lệ technical error < 1%.
- Support Ops duyệt blind sample 20 case, trong đó không có P1 unresolved.

### Warn, không block tự động

- Overall category accuracy giảm 1–2 điểm phần trăm nhưng không chạm high-risk segment.
- Cost tăng ≤15% hoặc P95 latency tăng ≤20% so với baseline. Cả hai phải có chủ sở hữu và thời hạn xử lý.

## 9. Năm Dataset Edge Cases bổ sung

1. **Happy path — technical rõ ràng, khách standard:** “Không đăng nhập được sau reset password; không có dấu hiệu chặn toàn bộ công việc.”  
   Bắt failure: route sai technical hoặc escalation quá mức.
2. **Ambiguous input — nhiều intent:** “Tôi không vào được tài khoản và cũng thấy hóa đơn tháng này lạ.”  
   Bắt failure: AI bỏ mất một intent hoặc chọn route tự tin khi cần human review.
3. **Missing information — câu quá ngắn:** “Tài khoản của tôi có vấn đề, xử lý gấp.”  
   Bắt failure: bịa category/urgency thay vì `unknown` hoặc review.
4. **High-risk / escalation — enterprise bị khóa vì billing:** “Toàn đội bị disabled sau khi payment failed, hôm nay phải xuất hóa đơn.”  
   Bắt failure: `billing` bị route sai, urgency thấp, thiếu `requires_human=true`.
5. **Regression case — wording từng gây sai:** “Không thể access invoice và màn hình nói account locked.”  
   Bắt failure: model nhìn thấy từ “invoice” rồi route product question, bỏ sót account access + blocking work.

## 10. Kế hoạch chạy thử và dự toán chi phí

### Mục tiêu pilot

Trong 5 ngày làm việc, chứng minh ba điều: (1) triage có thể đạt gate vận hành cho segment thường; (2) segment enterprise/high-risk được chặn bằng deterministic rule + human checkpoint; và (3) team biết chính xác failure mode nào cần sửa prompt, taxonomy hay workflow trước khi tích hợp thật.

### Phạm vi và cách chạy

- **80 reference cases**: 30 happy/normal, 20 ambiguity/thiếu thông tin, 20 high-risk enterprise, 10 regression từ mock/production-like cases.
- **40 full-suite iterations**: 80 × 40 = **3,200 candidate runs**. Mỗi run chạy hard checks; LLM judge chấm tất cả ở giai đoạn pilot để xây baseline.
- **Token assumption/candidate call:** 900 input + 200 output tokens.
- **Token assumption/judge call:** 1,200 input + 250 output tokens.
- **Model pricing used:** GPT-5 mini, USD 0.25/1M input tokens và USD 2.00/1M output tokens; phải kiểm tra lại giá tại thời điểm mua.

### API cost calculation

- Candidate: `3,200 × ((900/1,000,000 × 0.25) + (200/1,000,000 × 2.00)) = USD 2.00`.
- Judge: `3,200 × ((1,200/1,000,000 × 0.25) + (250/1,000,000 × 2.00)) = USD 2.56`.
- API subtotal: **USD 4.56**.
- Thêm 30% rerun/debug buffer: **USD 5.93**, quy đổi planning **~154,000 VND**.

### Giờ công và ngân sách

- PM/eval design: **12 giờ × 250,000 VND = 3,000,000 VND**.
- Engineer/ops instrumentation: **10 giờ × 300,000 VND = 3,000,000 VND**.
- Support QA/human review: **8 giờ × 180,000 VND = 1,440,000 VND**.
- Tổng nhân sự: **7,440,000 VND**.
- API + nhân sự: **7,594,000 VND**.
- Contingency 10% cho re-label/rerun: **~759,000 VND**.
- **Tổng pilot đề nghị: ~8,350,000 VND** (làm tròn 8.4 triệu VND), timeline **5 ngày làm việc**.

Giá API được dùng như một giả định planning với GPT-5 mini; cần xác nhận lại trên pricing chính thức trước khi mua. Với quy mô này, chi phí máy nhỏ hơn chi phí human review; điều đó phù hợp vì rủi ro chính của triage là route/escalation vận hành, không phải số token. Pilot đủ nhỏ để kiểm chứng hướng đi, nhưng đủ 3,200 lần chạy để nhìn thấy regression và phân tách kết quả theo segment.
