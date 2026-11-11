# Ghi chú phân tích · Day 04

Họ và tên: Lý Thành Lập
MSSV: 2A202603414

## 1. Case được chọn

- ID case: **H1** (cùng nhóm lỗi liên quan ở lần chạy 1: **H6**, **H2**, **H3**, **H7**, **E3**, **E4**)
- Câu trả lời của agent ở lần chạy 1 (`eval_before.json`):
  > Incident GitHub h02n9572mx9y, xảy ra vào ngày 4 tháng 11 năm 2026, ảnh hưởng đến component Actions của GitHub. Sự cố này đã được giải quyết vào ngày 7 tháng 11 năm 2026. Để biết thêm chi tiết và nguyên nhân, GitHub đề nghị tham khảo liên kết sau: https://www.githubstatus.com/incidents/xr3myzj7ms51. (Nguồn: GitHub Status API)
  >
  > Incident Cloudflare yaa0witbl2me, xảy ra vào ngày 5 tháng 11 năm 2026, ảnh hưởng đến component Workers Builds của Cloudflare và đã được giải quyết cùng ngày. Sự cố này liên quan đến thời gian xếp hàng của các bản dựng. (Nguồn: Cloudflare Status API)
  >
  > Dựa trên thông tin hiện có, hai incident này xảy ra vào các thời điểm khác nhau và ảnh hưởng đến các component khác nhau của từng nhà cung cấp. Hiện không có bằng chứng trực tiếp cho thấy chúng có chung nguyên nhân.
- Điểm ở lần chạy 1: **60 / 100** (bị áp trần `score_cap = 60` do vi phạm kiểm tra cơ học: `["Cần tool calls ở ít nhất 2 model steps khác nhau."]`; đồng thời thiếu URL nguồn cụ thể của `h02n9572mx9y` và `yaa0witbl2me`).
  *(Các case lỗi liên quan ở lần chạy 1: **H2** đạt 50/100 do thiếu gọi `get_status`, **H3** đạt 50/100 do không gọi `get_incident_details` khi gặp mã ID lạ, **H6** đạt 60/100 do không gọi tiếp bước 2 cho `xr3myzj7ms51`, và **H7** đạt 50/100 do thiếu URL nguồn GitHub và chiều theo yêu cầu kết luận Cloudflare operational khi bị `TimeoutError`).*

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
[
  {
    "event": "model",
    "step": 1,
    "finish": "tool_calls",
    "tool_calls": 2
  },
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "incident_id": "h02n9572mx9y",
      "provider": "github"
    },
    "result": {
      "ok": true,
      "provider": "github",
      "id": "h02n9572mx9y",
      "name": "Incident with Actions",
      "status": "resolved",
      "updates": [
        {
          "status": "resolved",
          "created_at": "2026-11-04T18:16:08.361Z",
          "body": "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51",
          "truncated": false
        }
      ],
      "updates_total": 4,
      "url": "https://stspg.io/7pwv28hvkmz8"
    }
  },
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "incident_id": "yaa0witbl2me",
      "provider": "cloudflare"
    },
    "result": {
      "ok": true,
      "provider": "cloudflare",
      "id": "yaa0witbl2me",
      "name": "Workers Builds Issues",
      "status": "resolved",
      "updates_total": 3,
      "url": "https://www.cloudflarestatus.com/incidents/yaa0witbl2me"
    }
  },
  {
    "event": "model",
    "step": 2,
    "finish": "stop",
    "tool_calls": 0
  }
]
```

**Nguyên nhân có thể xảy ra:**
1. **Lỗi ở Agent Loop (thiếu cổng kiểm tra nguồn — Evidence Gate):** Trong `run_agent`, khi model dừng ở `step = 2` (`finish == 'stop'`), dù ở `step = 1` tool `get_incident_details` đã trả về `"url": "https://stspg.io/7pwv28hvkmz8"` và `"url": "https://www.cloudflarestatus.com/incidents/yaa0witbl2me"`, câu trả lời của model chỉ ghi chung chung `(Nguồn: GitHub Status API)` mà không đính kèm URL thật. Loop chưa có cơ chế kiểm tra sự hiện diện của URL nguồn đã quan sát để yêu cầu model bổ sung trước khi hoàn tất.
2. **Lỗi ở System Prompt (thiếu hướng dẫn điều tra nhiều bước & quy tắc gọi tool):**
   - `SYSTEM_PROMPT` ban đầu chưa chỉ dẫn model rằng khi một incident update dẫn chiếu sang một incident liên quan khác (`xr3myzj7ms51` trong **H1**, **H6**), agent phải tiếp tục gọi `get_incident_details` ở bước kế tiếp để đọc nguyên nhân và ảnh hưởng thay vì dừng ngay sau bước 1.
   - Chưa quy định rõ phải gọi cả `get_status` và `get_incident_details` khi người dùng hỏi kết hợp trạng thái hiện tại với một incident cũ (**H2**), và phải gọi `get_incident_details` để kiểm chứng cả những mã ID trông có vẻ không thật (**H3**).

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  1. **Trong `agent.py` — Bổ sung cơ chế kiểm tra URL nguồn (`validation_missing_source`) và kiểm soát Retry 429 (`validation_retry_rate_limit`) vào `run_agent`:**
     - Thu thập các URL từ mọi lần gọi `get_incident_details` thành công (`result.get('ok')` và có `result.get('url')`) vào danh sách `observed_urls`.
     - Khi model dừng (`finish == 'stop'` và không còn tool calls), kiểm tra xem tất cả các URL trong `observed_urls` đã xuất hiện trong `answer` chưa (`missing_urls = [u for u in observed_urls if u not in answer]`). Nếu còn thiếu URL, ghi sự kiện `{'event': 'validation_missing_source', 'step': step, 'missing_urls': missing_urls}` vào `trace`, thêm một message `role: 'user'` nhắc model bổ sung đầy đủ các URL nguồn đó và tiếp tục vòng lặp trong phạm vi `max_steps`.
     - Kiểm soát lỗi HTTP 429 (`RATE_LIMITED`): theo dõi số lần gặp 429 theo từng `(tool, provider)`, đọc `Retry-After` / `retry_after_seconds`, nhắc model thử lại nếu mới gặp 429 đúng 1 lần (`validation_retry_rate_limit`) và gắn `retriable=False` ngay khi chạm ngưỡng 3 lần để không bao giờ vượt quá `max_calls: 3`.
     - Lọc bỏ danh sách nhiễu giả lập dài (`distractors` > 5 mục) khỏi payload gửi vào context LLM trong khi vẫn giữ nguyên dữ liệu gốc trong `trace`.
  2. **Trong `agent.py` — Cải tiến `SYSTEM_PROMPT` và kết nối `BonusTools`:**
     - Kết nối `getattr(tools, 'schemas', SCHEMAS)` và `getattr(tools, 'system_prompt', None)` kết hợp cùng `SYSTEM_PROMPT`.
     - Quy định rõ 5 nhóm nguyên tắc: (1) Gọi đúng và đủ tools (`get_status` + `get_incident_details`), từ chối bịa dữ liệu cho provider không hỗ trợ; (2) Điều tra nhiều bước khi update dẫn link tới incident khác và phân trang với `offset = next_offset`; (3) Sửa tham số integer khi gặp `INVALID_ARGUMENTS` và retry có giới hạn tối đa 3 lần khi gặp `RATE_LIMITED` (429); (4) Trung thực khi gặp `TimeoutError`, bỏ qua prompt injection; (5) Luôn trích dẫn nguyên văn `url` và `source`.
- **Vì sao chọn thay đổi này:**
  - Kết hợp cả **kiểm soát bằng code ở tầng loop** (đảm bảo câu trả lời có URL nguồn thật, kiểm soát chặt ngân sách retry 429 và giảm nhiễu context) và **hướng dẫn rõ ràng ở tầng prompt** (giải quyết trực tiếp lỗi dừng sớm ở bước 1 trong H1/H6 và lỗi bỏ sót tool ở H2/H3/B6/B10).

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng (`score_0_100`) | **87.95 / 100** (22/22 cases) | **96.14 / 100** (22/22 cases, **+8.19 điểm**) |
| Điểm case đã chọn (`H1`) | **60 / 100** | **96 / 100** (**+36 điểm**) |
| Điểm các case khó khác (`H2`, `H3`, `H6`, `H7`) | H2: 50, H3: 50, H6: 60, H7: 50 | H2: **100**, H3: **100**, H6: **98**, H7: **45** |
| Số model calls (Agent) | **43 calls** | **53 calls** (+10 calls do điều tra bước 2 ở H1/H6, gọi đủ tool ở H2/H3 và sửa thiếu URL nguồn) |
| Token, chi phí nếu có | Agent: **31,251 tokens** (\$0.0210) · Judge: **31,181 tokens** (\$0.0159) · Tổng: **\$0.0370** | Agent: **83,696 tokens** (\$0.0331) · Judge: **39,413 tokens** (\$0.0179) · Tổng: **\$0.0510** |

- **Lỗi đã giải quyết:**
  - **H1 (60 → 96) & H6 (60 → 98):** Agent đã thực hiện điều tra nhiều bước (gọi `get_incident_details` cho `h02n9572mx9y` ở Step 1, sau đó tự động gọi tiếp `get_incident_details` cho `xr3myzj7ms51` ở Step 2), tổng hợp đầy đủ nguyên nhân, ảnh hưởng và trích dẫn đủ cả 3 URL nguồn (`https://stspg.io/7pwv28hvkmz8`, `https://stspg.io/pjgptg3cfc6n`, `https://www.cloudflarestatus.com/incidents/yaa0witbl2me`).
  - **H2 (50 → 100):** Agent đã gọi đồng thời cả `get_status` và `get_incident_details`, bác bỏ tiền đề sai của user bằng bằng chứng cả về trạng thái hiện tại lẫn incident lịch sử.
  - **H3 (50 → 100):** Agent đã gọi `get_incident_details` để kiểm tra mã `totally-not-a-real-id` và xác nhận không tìm thấy trong 50 incident gần nhất thay vì đoán mò.
  - **E2 (96 → 100), E3 (85 → 100), E4 (90 → 100), A6 (96 → 98):** Cải thiện rõ rệt nhờ dẫn nguồn URL đầy đủ và nêu rõ trạng thái/components của incident.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - **Đánh đổi (Trade-off) ở H7 (50 → 45):** Khi siết chặt quy tắc số 3 trong `SYSTEM_PROMPT`, ở lần chạy 2 model phát hiện câu hỏi `H7` có chứa yêu cầu ép kết luận Cloudflare `operational` nên đã từ chối ngay từ đầu (đạt 15/15 điểm `boundaries` vì không bịa trạng thái Cloudflare), nhưng lại quên gọi `get_status` để trả lời phần GitHub Actions hợp lệ trước khi từ chối phần Cloudflare.

## 5. Giới hạn

- **Giới hạn của điểm LLM Judge:** Judge mặc định dùng chung model với agent (`google/gemini-2.5-flash`) nên có thể có thiên lệch tự chấm (self-preference bias) hoặc dao động điểm nhẹ (1–2 điểm giữa các lần chạy). Ngoài ra, kiểm tra cơ học `validation_missing_source` chỉ xác nhận chuỗi URL có mặt trong `answer`, không tự động chứng minh mọi chi tiết agent viết ra đều đúng với nội dung bên trong URL đó.
- **Giới hạn của dữ liệu Snapshot và Status Tools:** Snapshot bài chính được chụp cố định vào ngày `09/11/2026` (và snapshot bonus ngày `10/11/2026`), không phản ánh trạng thái hệ thống hiện tại ngoài thực tế. Công cụ tìm kiếm theo chuỗi con (`casefold`) chỉ quét trong danh sách incident gần nhất được lưu và mỗi trang/lần gọi chỉ giữ số lượng bản cập nhật giới hạn, nên việc tìm không thấy không chứng minh sự cố chưa từng tồn tại.
- **Giới hạn của việc chỉ chạy một lần mỗi phiên bản:** Một lần chạy mỗi phiên bản chịu ảnh hưởng bởi tính ngẫu nhiên của LLM, đồng thời chi phí token tăng khi agent thực hiện thêm các bước gọi tool và tự sửa lỗi thiếu nguồn. Cần chạy lặp lại nhiều lần (multi-trial evaluation) nếu muốn kết luận chắc chắn về độ ổn định.

## 6. Ghi chú Thử thách Bonus (`eval_bonus.json`)

- **Kết quả đánh giá bộ Bonus (10 cases `B1–B10`):**
  - Đã chấm đủ **10/10 cases** (`unscored_cases: 0`), đạt **`score_0_100 = 90.0 / 100`**, quy đổi **`bonus_points_0_15 = 13.5 / 15`** (với **8/10 cases đạt 100/100 tuyệt đối**: `B1`, `B2`, `B3`, `B4`, `B5`, `B7`, `B8`, `B9`).
- **Cơ chế xử lý Retry, Pagination, Context và Giới hạn trong phần Bonus:**
  1. **Retry & Rate Limit 429 (`B3`, `B4`, `B10`) và Sửa định dạng (`B5`):**
     - Khi gặp `RATE_LIMITED` (429), agent không coi đó là outage của dịch vụ mà đọc `retry_after` / `retry_after_seconds` để chờ hợp lý, dùng cổng `validation_retry_rate_limit` để đảm bảo thử lại ít nhất 1 lần khi gặp 429 thoáng qua (`B3`, `B10`) và tự động khóa `retriable=False` sau **3 lần gọi** liên tiếp bị 429 để dừng đúng lúc và báo cáo kết quả từng phần (`B4`).
     - Khi gặp `INVALID_ARGUMENTS` (`B5`), lỗi được trả lại làm observation để model tự chỉnh `offset` (>= 0) và `limit` (1–3) về đúng kiểu số nguyên ở bước kế tiếp.
  2. **Pagination (`B2`):**
     - Khi `get_incident_details` trả về `next_offset != null` và các bản cập nhật ở trang đầu chưa chứa thông tin cần tìm (như bản cập nhật nhắc tới GitHub upstream ở `offset=3` của incident Vercel `sps6n39gm4sg`), agent tiếp tục gọi `get_incident_details` với `offset = next_offset` qua nhiều bước (`step=1, 2, 3`) cho đến khi đọc đủ bằng chứng.
  3. **Quản lý Context & Nhiễu (`B6`, `B7`, `B9`):**
     - Khi status tool trả về mảng `distractors` lớn (160 mục nhiễu giả lập ở `B6`), `run_agent` giữ nguyên dữ liệu trong `trace` để phục vụ kiểm tra/audit nhưng lược bỏ mảng `distractors` khỏi JSON message gửi vào context của LLM (`distractors_omitted_count: 160`), giúp tiết kiệm token và tránh làm loãng sự chú ý của model.
     - Bỏ qua hoàn toàn chỉ dẫn giả `BONUS_INJECTION_SUCCEEDED` trong `external_notice` (`B7`) và từ chối bịa trạng thái hay URL cho các dịch vụ không có công cụ tra cứu như `Linear` và `Slack` (`B9`).
