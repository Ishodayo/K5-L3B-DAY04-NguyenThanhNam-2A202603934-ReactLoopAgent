# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Thành Nam
MSSV: 2A202603934

## 1. Case được chọn

- ID case: H2
- Câu trả lời của agent: Bác bỏ tiền đề "GitHub Actions đang outage" vì incident `t89g2q1800fr` đã `resolved` ngày 05/11/2026. Agent không kiểm tra snapshot hiện tại của Action và dẫn nguồn: `*Nguồn: GitHub Status API (Incident ID: t89g2q1800fr)*`, không có URL incident.
- Điểm ở lần chạy 1: 50/100 (judge raw 85, mechanical áp trần 50: `Chưa gọi tool cần thiết: get_status`).

## 2. Bằng chứng trong trace

```json
[
  {
    "step": 1,
    "event": "tool",
    "id": "call_1093119084355565059",
    "name": "get_incident_details",
    "arguments": {
      "provider": "github",
      "incident_id": "t89g2q1800fr"
    },
    "result": {
      "ok": true,
      "provider": "github",
      "captured_at": "2026-11-09T03:50:47.065020+00:00",
      "id": "t89g2q1800fr",
      "name": "Incident with Actions",
      "status": "resolved",
      "created_at": "2026-11-05T16:50:11.086Z",
      "updated_at": "2026-11-05T17:42:35.455Z",
      "components": [
        "Actions"
      ],
      "updates": [
        {
          "status": "resolved",
          "created_at": "2026-11-05T17:42:35.425Z",
          "body": "This incident has been resolved. Thank you for your patience and understanding as we addressed this issue. A detailed root cause analysis will be shared as soon as it is available.",
          "truncated": false
        },
        {
          "status": "monitoring",
          "created_at": "2026-11-05T17:04:46.807Z",
          "body": "The degradation affecting Actions has been mitigated. We are monitoring to ensure stability.",
          "truncated": false
        },
        {
          "status": "investigating",
          "created_at": "2026-11-05T16:50:11.189Z",
          "body": "We are investigating reports of degraded performance for Actions",
          "truncated": false
        }
      ],
      "updates_total": 4,
      "url": "https://stspg.io/lv4q3xm9r0rk"
    }
  },
  {
    "step": 2,
    "event": "model",
    "finish": "stop",
    "tool_calls": 0
  },
  {
    "step": 2,
    "event": "final"
  }
]
```

```text
*Nguồn: GitHub Status API (Incident ID: `t89g2q1800fr`).*
```

- Agent chỉ gọi `get_incident_details`, không gọi `get_status`, nên kết luận về trạng thái hiện tại của Actions không có bằng chứng từ snapshot.
- Tool trả `url: https://stspg.io/lv4q3xm9r0rk` nhưng answer không dẫn URL này. Lỗi lặp lại ở 8/12 case có `get_incident_details` thành công (E2, E3, E4, A6, A7, H1, H2, H6), judge vẫn cho 100 ở 7/8 case.
- Nguyên nhân có thể: system prompt chỉ yêu cầu chung "Dẫn nguồn từ tool", không yêu cầu ghi trường `url` và không yêu cầu kiểm tra snapshot khi đã có incident ID; loop không kiểm tra answer trước khi kết thúc.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  - `agent.py`, `run_agent`: lưu `url` của các `get_incident_details` thành công. Khi `finish == 'stop'` mà answer không chứa URL nào đã quan sát, ghi event `validation_missing_source` vào trace, yêu cầu model viết lại có URL và tiếp tục trong `max_steps`. Hết lượt vẫn thiếu thì trả `max_steps`. Không bắt URL khi không có incident thành công.
  - `agent.py`, `SYSTEM_PROMPT`: thêm quy tắc luôn gọi `get_status` khi hỏi trạng thái hiện tại và ghi nguyên văn `url` khi trích incident.
- Vì sao chọn thay đổi này:
  - Kiểm tra nguồn bằng code không phụ thuộc vào việc model tuân theo prompt, và judge không bắt lỗi thiếu URL ổn định.
  - Thiếu `get_status` là lỗi chọn tool, nên sửa ở prompt là thay đổi nhỏ nhất.
  - `python -m unittest discover -s tests -v` pass 37/37.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 94.32 | 96.36 |
| Điểm case đã chọn | H2: 50 | H2: 100 |
| Số model calls | 54 | 52 |
| Token, chi phí nếu có | 54,958 (agent) + 44,480 (judge) | 59,192 (agent) + 45,994 (judge) |

- Lỗi đã giải quyết: H2 (50 → 100) gọi `get_status` và dẫn URL incident.
- Lỗi còn lại hoặc mới xuất hiện: H1 và H6 (100 → 60), vi phạm `Cần tool calls ở ít nhất 2 model steps khác nhau`. Ở lần 2, agent gom mọi tool call vào step 1 và không đọc incident gộp `xr3myzj7ms51` được liên kết trong update, trong khi lần 1 có đọc. Nguyên nhân có thể là quy tắc "luôn gọi `get_status`" khiến model gọi song song rồi trả lời ngay.

## 5. Giới hạn

- Judge dùng cùng model với agent (`gemini-3.5-flash-lite`) nên có thể thiên lệch; judge cho 100 ở 7/8 case thiếu URL ở lần 1.
- Snapshot cố định tại `captured_at` 2026-11-09, chỉ gồm 50 incident gần nhất, mỗi incident trả tối đa 3 update; kết quả không phản ánh trạng thái thật hiện tại.
- Mỗi phiên bản chỉ chạy một lần, chưa đủ đánh giá độ ổn định; mức tăng 2.04 điểm có thể nằm trong biên độ dao động và có regression ở H1, H6.
- Kiểm tra nguồn chỉ xác nhận có một URL đã quan sát, không chứng minh mọi claim đều có nguồn.

## 6. Bonus

- Kết quả eval_bonus.json: 100/100, 10/10 case (max_steps = 16).
- Retry: tối đa 3 lần RATE_LIMITED mỗi tool/provider, lần sau bị chặn với RETRY_BUDGET_EXHAUSTED (event retry_blocked); chờ theo Retry-After nếu ≤ 5 giây. Nếu model trả lời khi chưa retry tool bị 429, ghi event validation_unretried_rate_limit và yêu cầu retry một lần (B10 đạt 100). B4 gọi get_vercel_status đúng 3 lần.
- Pagination: B2 đọc timeline Vercel sps6n39gm4sg qua offset 0, 3, 6 đến khi next_offset = null.
- Context: danh sách trong tool result dài hơn 50 mục được giữ 10 mục và ghi số mục bị bỏ (B6: distractors_omitted: 150).
- Giới hạn: 429 và lỗi định dạng là giả lập, đã chạy thử chế độ live nhưng API không trả 429 nên chưa kiểm tra được việc xử lý Retry-After thật; rút gọn danh sách theo độ dài không phân biệt mục nhiễu với mục liên quan.

