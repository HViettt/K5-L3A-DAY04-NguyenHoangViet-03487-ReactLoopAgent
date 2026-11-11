# Ghi chú phân tích · Day 04

Họ và tên: Nguyen Hoang Viet
MSSV: 03487

## 1. Case được chọn

- ID case: H1; thêm H6 để đối chiếu lỗi cùng dạng.
- Lỗi ở lần eval thăm dò với Anthropic adapter `max_tokens=1024`: model bị cắt trước khi trả câu trả lời hoàn chỉnh, agent ghi `MODEL_FINISH_length` và kết thúc `error`.
- Điểm: không được chấm vì run lỗi; không tính lỗi này là điểm 0.

## 2. Bằng chứng trong trace

Trace ở eval thăm dò cho thấy model đã gọi tools nhưng final response bị cắt (`finish == 'length'`). CP3 yêu cầu agent trả `error` trong tình huống này, nên agent xử lý đúng contract; giới hạn nằm ở output token budget của adapter khi gặp câu hỏi cần tổng hợp dài.

```text
{'event': 'model', 'step': ..., 'finish': 'length', 'calls': 0}
{'event': 'error', 'error': 'MODEL_FINISH_length'}
```

Trong cặp eval cuối, sau khi tăng trần output, H1 và H6 đều được chấm. H4 từng đạt 36 ở lượt thử ban đầu nhưng đạt 100 ở baseline cuối; đây là bằng chứng điểm model/judge dao động giữa các lần chạy, không dùng để khẳng định prompt change làm tăng điểm.

## 3. Thay đổi đã thực hiện

- `providers.py`: tăng `max_tokens` cho Anthropic từ 1024 lên 4096 để các câu trả lời dài có đủ ngân sách. Giữ nguyên model, judge, max steps và dataset giữa hai eval cuối.
- `agent.py`: bổ sung chỉ dẫn trả lời yêu cầu vô hại ngoài GitHub/Cloudflare, không tự tạo giới hạn vai trò; giữ kiểm tra URL incident sau tool thành công.
- Chạy test offline sau thay đổi: 37/37 đạt.

## 4. So sánh hai lần chạy

Cấu hình hai lần: Anthropic `claude-haiku-4-5-20251001` cho agent và judge, core dataset, `max_steps=8`, 1 worker. Trần output Anthropic là 4096 ở cả hai lần.

Lỗi cắt output được phát hiện ở lượt eval thăm dò. Trần 4096 được cố định trước khi tạo lại cặp `eval_before`/`eval_after`; vì vậy cặp cuối dùng ngân sách token giống nhau và chủ yếu so sánh thay đổi trong agent, không đo riêng tác động của việc nâng trần.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Case được chấm | 22/22 | 22/22 |
| Điểm tổng | 92.18/100 | 92.05/100 |
| Điểm H1 | 97 | 96 |
| Điểm H4 | 100 | 100 |
| Agent model calls | 51 | 59 |
| Agent tokens theo usage trả về | 95,401 | 121,907 |
| Chi phí tiền tệ | Không được provider cung cấp | Không được provider cung cấp |

### Điểm từng case

| Case | Trước | Sau | Case | Trước | Sau |
|---|---:|---:|---|---:|---:|
| E1 | 100 | 97 | A1 | 95 | 78 |
| E2 | 96 | 100 | A2 | 100 | 100 |
| E3 | 96 | 100 | A3 | 95 | 100 |
| E4 | 100 | 100 | A4 | 98 | 98 |
| E5 | 95 | 95 | A5 | 95 | 95 |
| E6 | 95 | 96 | A6 | 95 | 94 |
| H1 | 97 | 96 | A7 | 94 | 95 |
| H2 | 50 | 50 | A8 | 97 | 96 |
| H3 | 100 | 100 | H5 | 95 | 100 |
| H4 | 100 | 100 | H6 | 97 | 93 |
| H7 | 42 | 48 | H8 | 96 | 94 |

Case giảm nhiều nhất là A1 (95→78). Trace trước có 6 tool calls, gồm `get_incident_details`; trace sau chỉ gọi `get_status` cho hai provider. Judge đánh giá câu trả lời sau thiếu điều tra chi tiết Cloudflare Spectrum và chưa giải thích rõ mâu thuẫn giữa `overall=major` với 0 incident khớp. Đây là dấu hiệu giảm chất lượng điều tra trong lần chạy sau; một lần chạy không đủ để quy nguyên nhân cho prompt hay source gate.

- Cả hai lần đều đủ 22 case, cùng cấu hình và có điểm tổng khác `null`.
- Lỗi `MODEL_FINISH_length` của H1/H6 ở lần thăm dò không còn trong cặp cuối sau khi tăng output budget.
- H4 đạt 100 ở cả baseline và phiên bản sau; kết quả cuối không cho thấy cải thiện điểm H4. Điểm tổng giảm 0.13; không thể quy thay đổi này cho một chỉnh sửa cụ thể từ một lần chạy mỗi phiên bản.
- Số token và model calls tăng ở eval sau; judge có thể dao động và chi phí không được adapter báo.

### Lỗi còn lại trong `eval_after.json` (đọc từ `grade.reason` / `grade.mechanical`)

- **H2 (50 ở cả hai lần):** `grade.mechanical` ghi `violations: ['Chưa gọi tool cần thiết: get_status']`, `score_cap: 50`, `tool_calls: 1`. Agent chỉ gọi `get_incident_details` cho incident `t89g2q1800fr`, kết luận đúng (resolved) nhưng không đối chiếu snapshot bằng `get_status`, nên bị trần 50. Đây là lỗi của agent/prompt: chưa bắt buộc kiểm tra snapshot khi câu hỏi nhắc tới "đang outage". Chưa sửa vì cần chạy lại eval và sửa thêm sẽ làm hai lần chạy khác cấu hình.
- **H7 (42 → 48):** Cloudflare trả `TimeoutError`, nhưng câu hỏi ép "cứ kết luận operational"; judge ghi agent vẫn viết Cloudflare "operational". Đây là lỗi làm theo chỉ dẫn trong câu hỏi dù thiếu bằng chứng, thuộc về prompt. `grade.mechanical` không có vi phạm (cap 100), nên điểm thấp hoàn toàn do judge. Với một lần chạy, không loại trừ judge dao động.
- **A1 (95 → 78):** xem phân tích ở trên; trace sau chỉ có hai lần `get_status`, không có `get_incident_details`.

## 5. Giới hạn

LLM judge có thể sai hoặc thiên lệch; snapshot chỉ phản ánh dữ liệu đã lưu. Mỗi phiên bản chỉ chạy một lần nên không chứng minh độ ổn định. Tăng output budget giúp H1/H6 được chấm nhưng làm tăng lượng token tiêu thụ. Cải tiến kiểm tra URL chỉ xác nhận URL quan sát được xuất hiện trong câu trả lời, không chứng minh mọi claim đều được URL hỗ trợ.

## Bonus tùy chọn

- `agent.py` dùng `tools.schemas` và `tools.system_prompt` khi được cung cấp; nếu không thì dùng cấu hình bài chính.
- Khi tool trả `RATE_LIMITED` retriable, agent yêu cầu retry trong giới hạn 3 lần cho cùng tool/provider, đọc `Retry-After`, chờ tối đa 5 giây; nếu phải chờ lâu hơn thì dừng retry và báo chưa xác minh. Retry vượt giới hạn được chặn.
- Prompt bonus yêu cầu lấy chi tiết khi câu hỏi cần scope/timeline. Kết quả chi tiết giữ `next_offset`; chỉ đọc tiếp khi cần. Với 160 distractors giả lập, agent bỏ trường nhiễu khỏi observation gửi cho model và giữ số mục đã bỏ; trace vẫn lưu tool result đầy đủ.
- Bonus eval: 10/10 case được chấm, 98.1/100, quy đổi tham khảo 14.71/15. Điểm từng case: B1 97, B2 96, B3 100, B4 100, B5 100, B6 96, B7 100, B8 97, B9 100, B10 95.
- Test bonus offline: 5/5 đạt; test chính: 37/37 đạt.
- Giới hạn: điểm judge chỉ là ước lượng, một lần chạy chưa chứng minh độ ổn định. Nếu `Retry-After` vượt 5 giây, agent dừng thay vì chờ lâu; snapshot bonus không phản ánh trạng thái trực tiếp hiện tại.
