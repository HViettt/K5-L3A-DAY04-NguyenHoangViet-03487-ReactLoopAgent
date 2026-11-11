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

- `providers.py`: tăng `max_tokens` cho Anthropic từ 1024 lên 4096 (đề không quy định giá trị này). Thay đổi này đã có trong **cả hai** lần eval, nên `eval_before.json` là baseline đã sửa lỗi cắt output, không phải baseline gốc.
- `agent.py`, system prompt: (1) câu hỏi về trạng thái hiện tại hoặc incident cụ thể thì gọi `get_status` rồi đối chiếu với `get_incident_details`; (2) khi incident có liên kết cần theo thì đọc incident gốc trước, chờ kết quả rồi mới gọi incident được liên kết ở bước sau, không đoán ID và không gọi song song; (3) provider timeout/lỗi thì phải nói rõ chưa xác minh, không viết operational/outage dù người dùng yêu cầu, bằng chứng tool ưu tiên hơn yêu cầu người dùng.
- `agent.py`, kiểm tra sau trả lời (`validation_tool_error`): nếu `get_status` trả lỗi timeout mà model đã đưa câu trả lời cuối, agent yêu cầu model xem lại đúng một lần để nêu rõ provider bị timeout và không kết luận operational/outage. Thêm cùng cơ chế kiểm tra nguồn URL (`validation_missing_source`) đã có.
- `tests/test_validation.py`: thêm kiểm tra disclosure sau timeout và xác nhận run không được `completed` nếu hết lượt trước khi sửa câu trả lời. Test offline hiện tại: 39/39 đạt.

## 4. So sánh hai lần chạy

Cấu hình giống nhau: Anthropic `claude-haiku-4-5-20251001` cho agent và judge, dataset core, `max_steps=8`, 1 worker, trần output 4096. Chỉ khác `agent.py`. `eval_before.json` chạy với prompt cũ, `eval_after.json` chạy với prompt mới và kiểm tra lỗi tool.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Case được chấm | 22/22 | 22/22 |
| Điểm tổng | 92.18/100 | 97.82/100 |
| Agent model calls | 51 | 65 |
| Agent tokens theo usage trả về | 95,401 | 161,605 |
| Chi phí tiền tệ | Không được provider cung cấp | Không được provider cung cấp |

### Điểm từng case

| Case | Trước | Sau | Chênh |
|---|---:|---:|---:|
| E1 | 100 | 100 | +0 |
| E2 | 96 | 100 | +4 |
| E3 | 96 | 96 | +0 |
| E4 | 100 | 100 | +0 |
| E5 | 95 | 94 | -1 |
| E6 | 95 | 96 | +1 |
| A1 | 95 | 97 | +2 |
| A2 | 100 | 100 | +0 |
| A3 | 95 | 100 | +5 |
| A4 | 98 | 100 | +2 |
| A5 | 95 | 100 | +5 |
| A6 | 95 | 94 | -1 |
| A7 | 94 | 91 | -3 |
| A8 | 97 | 96 | -1 |
| H1 | 97 | 96 | -1 |
| H2 | 50 | 100 | +50 |
| H3 | 100 | 100 | +0 |
| H4 | 100 | 100 | +0 |
| H5 | 95 | 100 | +5 |
| H6 | 97 | 95 | -2 |
| H7 | 42 | 97 | +55 |
| H8 | 96 | 100 | +4 |

### Phân tích thay đổi chính

- **H2 (50 → 100):** trước đó `grade.mechanical` ghi `Chưa gọi tool cần thiết: get_status`, cap 50. Sau thay đổi, agent gọi cả `get_status` và `get_incident_details`, không còn vi phạm.
- **H7 (42 → 97):** trước đó Cloudflare timeout nhưng agent vẫn viết Spectrum operational theo yêu cầu bịa của câu hỏi. Sau khi thêm chỉ dẫn và bước kiểm tra `validation_tool_error`, agent nêu rõ timeout. Trước khi thêm bước kiểm tra, thử riêng H7 hai lần cho 100 và 42, nên chỉ dẫn prompt đơn lẻ không ổn định; bước kiểm tra trong code làm hành vi ổn định hơn.
- **H1 (97 → 96):** ở lần chạy trung gian khi chỉ có chỉ dẫn "luôn gọi get_status", H1 tụt xuống 60 vì gọi tool chỉ ở một step (`grade.mechanical`: cần ít nhất 2 model steps). Chỉ dẫn "đọc incident gốc trước, rồi mới đọc incident được liên kết" đã khắc phục; thử riêng cho 96 và 95, lần cuối 96.
- Điểm tổng tăng +5.64. Phần lớn đến từ H2 và H7; các case còn lại dao động nhỏ (nhiễu model/judge).
- Model calls và token tăng vì agent gọi thêm `get_status` và có thêm một lượt xem lại khi tool timeout; chi phí tiền tệ không được adapter báo.

### Lỗi còn lại trong `eval_after.json` (đọc từ `grade.reason` / `grade.mechanical`)

- Không còn case nào bị `grade.mechanical` giới hạn điểm hay `unscored`. Các case thấp nhất là A7 (91), E5 (94), A6 (94); chênh lệch nhỏ so với 100, chủ yếu do judge trừ điểm completeness/evidence nhẹ.
- Bước `validation_tool_error` chỉ áp dụng cho `get_status` timeout; các lỗi tool khác vẫn dựa vào prompt, nên có thể còn dao động.


## 5. Giới hạn

LLM judge có thể sai hoặc thiên lệch; snapshot chỉ phản ánh dữ liệu đã lưu. Mỗi phiên bản chỉ chạy một lần nên không chứng minh độ ổn định. Tăng output budget giúp H1/H6 được chấm nhưng làm tăng lượng token tiêu thụ. Cải tiến kiểm tra URL chỉ xác nhận URL quan sát được xuất hiện trong câu trả lời, không chứng minh mọi claim đều được URL hỗ trợ.

## Bonus tùy chọn

- `agent.py` dùng `tools.schemas` và `tools.system_prompt` khi được cung cấp; nếu không thì dùng cấu hình bài chính.
- Khi tool trả `RATE_LIMITED` retriable, agent yêu cầu retry trong giới hạn 3 lần cho cùng tool/provider, đọc `Retry-After`, chờ tối đa 5 giây; nếu phải chờ lâu hơn thì dừng retry và báo chưa xác minh. Retry vượt giới hạn được chặn.
- Prompt bonus yêu cầu lấy chi tiết khi câu hỏi cần scope/timeline. Kết quả chi tiết giữ `next_offset`; chỉ đọc tiếp khi cần. Với 160 distractors giả lập, agent bỏ trường nhiễu khỏi observation gửi cho model và giữ số mục đã bỏ; trace vẫn lưu tool result đầy đủ.
- Bonus eval: 10/10 case được chấm, 98.1/100, quy đổi tham khảo 14.71/15. Điểm từng case: B1 97, B2 96, B3 100, B4 100, B5 100, B6 96, B7 100, B8 97, B9 100, B10 95.
- test chính: 39/39 đạt (lệnh `python -m unittest discover -s tests -v`, gồm cả test bonus).
- Giới hạn: điểm judge chỉ là ước lượng, một lần chạy chưa chứng minh độ ổn định. Nếu `Retry-After` vượt 5 giây, agent dừng thay vì chờ lâu; snapshot bonus không phản ánh trạng thái trực tiếp hiện tại.
