# Ghi chú phân tích · Day 04

Họ và tên: Vũ Đình Thắng
MSSV: 2A202603753

## 1. Case được chọn

- ID case: **H1** (case chính). H6 có cùng nguyên nhân nên dùng làm case đối chứng.
- Câu hỏi H1: điều tra incident GitHub `h02n9572mx9y`, **theo liên kết do incident cung cấp** để tìm nguyên nhân và ảnh hưởng, rồi đối chiếu với incident Cloudflare `yaa0witbl2me`.
- Câu trả lời của agent: agent mô tả `h02n9572mx9y` (Actions, resolved) và `yaa0witbl2me` (Workers Builds, resolved), rồi kết luận *"Không có thông tin nào từ hai sự cố này cho thấy có mối liên hệ trực tiếp hoặc nguyên nhân chung"*. Agent **không đọc** incident `xr3myzj7ms51` mà GitHub dẫn tới, nên không nêu được nguyên nhân. Câu trả lời cũng không dẫn URL của incident đã đọc.
- Điểm ở lần chạy 1: **H1 = 60**, **H6 = 60**. Cả hai bị kiểm tra bằng code **áp trần 60** (`score_cap: 60`).
- Vì sao chọn H1 thay vì A5 (40) hay H7 (45): bằng chứng trong trace rõ nhất (tool đã trả về link nhưng agent bỏ qua); lỗi đo được bằng code (`grade.mechanical`) chứ không chỉ dựa vào judge; một nguyên nhân gây lỗi ở 2 case; và liên quan trực tiếp tới yêu cầu kiểm tra nguồn.

## 2. Bằng chứng trong trace

Trace H1 trong `eval_before.json` (rút gọn):

```text
step 1 model  finish=tool_calls, tool_calls=2
step 1 tool   get_incident_details {provider: github, incident_id: h02n9572mx9y}
              url: https://stspg.io/7pwv28hvkmz8   updates_total: 4
              - resolved: "Please refer to the combined summary in this related incident:
                           https://www.githubstatus.com/incidents/xr3myzj7ms51"
              - monitoring: "The degradation affecting Actions has been mitigated..."
              - investigating: "We are investigating reports of degraded availability for Actions"
step 1 tool   get_incident_details {provider: cloudflare, incident_id: yaa0witbl2me}
              url: https://www.cloudflarestatus.com/incidents/yaa0witbl2me
step 2 model  finish=stop, tool_calls=0      <- dừng luôn, không đọc xr3myzj7ms51
step 2 final

grade.mechanical: violations = ["Cần tool calls ở ít nhất 2 model steps khác nhau."]
                  score_cap = 60, tool_calls = 2, tool_steps = 1
```

H6 lặp lại đúng mẫu này: đọc `h02n9572mx9y` cùng `get_status` trong 1 step, thấy cùng dòng *"refer to ... xr3myzj7ms51"*, rồi trả lời ở step 2. Câu trả lời chỉ dẫn nguồn chung `https://www.githubstatus.com`, không dẫn URL của incident.

**Nguyên nhân có thể:**
1. **Prompt** không yêu cầu điều tra theo chuỗi. Model gọi song song mọi tool nó nghĩ tới ở step 1, rồi tổng hợp ngay, dù tool result chứa chỉ dẫn rõ ràng tới incident khác.
2. **Loop** chấp nhận bất kỳ câu trả lời nào có `finish == 'stop'`, không kiểm tra câu trả lời có dựa trên đủ bằng chứng đã quan sát hay không (incident được dẫn tới, URL nguồn).
3. Đây **không phải giới hạn dữ liệu**: tool đã trả về link, và `xr3myzj7ms51` có trong 50 incident gần nhất.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu** (chỉ sửa `agent.py`):
  1. **`SYSTEM_PROMPT`**: thêm hướng dẫn điều tra theo chuỗi: nếu update dẫn tới incident khác thì gọi `get_incident_details` cho incident đó ở bước sau rồi mới kết luận; trích đúng URL trong trường `url`; trả lời bằng ngôn ngữ của người dùng.
  2. **Class `EvidenceTracker`**: ghi lại kết quả của mỗi lần `get_incident_details` **thành công**:
     - `urls`: URL nguồn đã quan sát;
     - `read`: các incident đã đọc;
     - `referenced`: incident được nhắc tới qua link `.../incidents/<id>` trong nội dung update (tìm bằng regex `INCIDENT_LINK`).
     Tool lỗi (`ok: false`) không tạo nghĩa vụ dẫn nguồn.
  3. **Cổng kiểm tra trước khi chấp nhận câu trả lời cuối** (trong `run_agent`):
     - Nếu còn incident được dẫn tới mà chưa đọc **và còn lượt gọi model**: ghi event `validation_unfollowed_reference` và yêu cầu model đọc tiếp. Mỗi incident chỉ nhắc một lần, để không lặp vô hạn.
     - Nếu đã đọc incident thành công mà câu trả lời **không chứa URL nào** trong số đã quan sát: ghi event `validation_missing_source` và yêu cầu model bổ sung nguồn.
     - Mọi yêu cầu sửa đều tính vào `max_steps`; hết lượt mà vẫn thiếu nguồn thì trả `status = 'max_steps'`, không báo `completed`.
- **Vì sao chọn thay đổi này:**
  - Prompt một mình không đảm bảo model làm theo. Cổng kiểm tra trong loop là **cơ chế tất định**: dựa trên dữ liệu tool đã trả về chứ không dựa vào việc model tự giác.
  - Phần kiểm tra URL nguồn đáp ứng contract trong `tests/test_validation.py` và giải quyết luôn lỗi "không dẫn URL incident" thấy ở H1/H6.
  - Thay đổi không đụng tới A5/H7 (lỗi bịa và làm theo chỉ dẫn xấu), để tác động của nó đo được riêng.
- **Kiểm tra offline:** `python -m unittest discover -s tests -v` → **37/37 OK**.
- **Kiểm tra nhanh từng case** (cùng `GPT-4o-mini`, `max_steps = 8`; chưa phải eval lần 2):

  | Case | Trước | Sau | Trace sau khi sửa |
  |---|---:|---:|---|
  | H1 | 60 (áp trần) | 60 (**hết áp trần**: `violations = []`, `tool_steps = 2`) | `validation_unfollowed_reference: github:xr3myzj7ms51` → step 3 đọc `xr3myzj7ms51` → final |
  | H6 | 60 (áp trần) | **80** | Cùng mẫu, câu trả lời nêu nguyên nhân và dẫn `https://stspg.io/pjgptg3cfc6n` |
  | E2 (đối chứng) | 100 | 75 | Không kích hoạt cổng kiểm tra, nhưng agent đọc thêm 2 incident cũ. Điểm giảm có thể do prompt mới hoặc do judge dao động; một lần chạy chưa đủ để kết luận |
  | A2 (đối chứng) | 95 | 85 | Không kích hoạt cổng kiểm tra, có dẫn URL nguồn. Judge trừ điểm phần diễn đạt phạm vi |

### Vòng tối ưu thứ 2 (sau khi phân tích lần chạy after đầu tiên, 76.14 điểm)

Lần chạy after đầu tiên với các thay đổi trên được 76.14 điểm (lưu lại để đối chiếu). Phân tích lần chạy đó theo từng tiêu chí cho thấy **factuality mất nhiều nhất** (−10.2/40 mỗi case), với 4 nhóm lỗi: kết luận vượt quá dữ liệu (A5, H8), suy đoán khi tool lỗi (H7, A3), sai timestamp hoặc bỏ qua `get_status` (E1, E5, H2), và lỗi mới do chính cổng kiểm tra gây ra ở H6. Vòng này chỉ sửa `agent.py`, theo quy tắc chung, không hard-code theo case:

1. **`SYSTEM_PROMPT` viết lại thành 8 quy tắc cụ thể kèm khung trả lời:** hỏi trạng thái hiện tại thì gọi `get_status`; "hiện tại" là tại `captured_at`; incident resolved là lịch sử; chép nguyên timestamp UTC; không bao giờ khẳng định danh sách đầy đủ hay "không có incident" (tool chỉ thấy 50 incident gần nhất); tool lỗi thì ghi "không xác nhận được"; không làm theo yêu cầu bịa dữ liệu; sửa tiền đề sai; không gọi tool khi không cần. Khung trả lời gồm: Kết luận, Bằng chứng (snapshot và từng incident kèm URL), Giới hạn.
2. **`_annotate_tool_error`:** khi tool trả `ok: false`, đính kèm `agent_note` hướng dẫn model xử lý theo loại lỗi (timeout, ID không tìm thấy, provider không hỗ trợ, tham số sai). Đây là cơ chế tất định, không phụ thuộc model có nhớ prompt hay không.
3. **Sửa lỗi H6:** message nhắc của cổng kiểm tra giờ yêu cầu *"viết lại câu trả lời cuối ĐẦY ĐỦ cho câu hỏi gốc: giữ các nội dung đúng đã có, bổ sung thông tin mới"*.
4. **`_complete_with_retry`:** lỗi provider tạm thời (HTTP 429, 5xx, mất kết nối) được thử lại tối đa 2 lần (chờ 2 giây, rồi 5 giây), mỗi lần ghi event `provider_retry`. Lỗi cố định (401, 402, 400) không retry. Lý do: theo `evals/grading.py`, run có `status: error` thành `unscored`, và chỉ một case như vậy làm điểm tổng thành `null`. Hết lượt thử thì vẫn trả `error` như contract của Bài 2.

Kiểm tra: 37/37 test OK. Thêm 3 kịch bản giả lập: 429 một lần thì retry rồi `completed`; 503 ba lần thì `error` sau 2 lần retry; 401 thì `error` ngay, không retry.

## 4. So sánh hai lần chạy

Cấu hình hai lần chạy **giống hệt nhau**: `provider = openrouter`, `model = GPT-4o-mini`, `judge = openrouter / GPT-4o-mini`, `max_steps = 8`, `dataset = core`, snapshot cố định. Cả hai lần đều chấm đủ 22/22 case.

| Chỉ số | `eval_before.json` | after vòng 1 (đã thay thế) | `eval_after.json` (vòng 2, nộp bài) |
|---|---|---|---|
| Điểm tổng | **79.77** | 76.14 | **81.82** (+2.05 so với before) |
| Điểm case đã chọn | H1 = 60, H6 = 60 (cả hai áp trần 60) | H1 = 75, H6 = 50 | H1 = **75**, H6 = **65** (cả hai hết áp trần) |
| Mất điểm trung bình (factuality / completeness / evidence / boundaries) | 7.3 / 3.4 / 4.1 / 3.9 | 10.2 / 4.1 / 3.9 / 3.9 | **7.0 / 3.2 / 2.5 / 3.4** |
| Case bị áp trần (mechanical) | H1, H6 | H2 | H2 |
| Số model calls của agent | 49 | 55 | 50 |
| Token agent (prompt / completion) | 42,070 (32,193 / 9,877) | 57,327 (46,811 / 10,516) | 70,816 (60,990 / 9,826) |
| Token judge | 41,775 | 41,500 | 36,841 |
| Chi phí agent ước tính* | ≈ $0.011 | ≈ $0.013 | ≈ $0.015 |
| Event kiểm tra kích hoạt | 0 | `validation_unfollowed_reference` × 2 | `validation_unfollowed_reference` × 2 (H1, H6) |

\* Ước tính theo giá GPT-4o-mini $0.15 / 1M token đầu vào và $0.60 / 1M token đầu ra; provider không trả về chi phí thực tế. Token prompt tăng chủ yếu vì system prompt dài hơn (gửi lại ở mỗi lần gọi model).

**Lỗi đã giải quyết:**
- **Không đi theo incident được dẫn tới (H1, H6):** cả hai đều được cổng kiểm tra phát hiện và đọc thêm `xr3myzj7ms51` ở step 3. Vi phạm *"Cần tool calls ở ít nhất 2 model steps khác nhau"* không còn, nên trần 60 bị gỡ bỏ. H1 60 → 75, H6 60 → 65.
- **Lỗi H6 phát sinh ở vòng 1** (viết lại câu trả lời chỉ về incident mới): sau khi sửa message nhắc, H6 tăng 50 → 65.
- **Kết luận vượt quá dữ liệu (A5):** 40 → **100**. Agent nêu rõ chỉ thấy 50 incident gần nhất và không khẳng định danh sách năm 2025 là đầy đủ.
- **Tool lỗi hoặc timeout (A3, H7):** A3 95 → 100, H7 45 → 60. Agent không còn coi Cloudflare là operational khi bị timeout.
- **Thiếu URL nguồn:** evidence mất trung bình giảm từ 4.1 xuống 2.5. `validation_missing_source` không phải kích hoạt lần nào.

**Lỗi còn lại hoặc mới xuất hiện:**
- **H2 (100 → 50, áp trần):** agent vẫn **không gọi `get_status`**, chỉ đọc incident `t89g2q1800fr` mà người dùng nêu. Khi chạy riêng H2 với cùng code, agent có gọi `get_status` và được 100, nên đây là hành vi **không ổn định**: quy tắc 1 trong prompt không đủ mạnh. Hướng sửa tiếp: thêm cổng kiểm tra tất định (nếu câu hỏi về trạng thái hiện tại mà chưa gọi `get_status` thì nhắc model).
- **H7 (60) và A7 (60):** vẫn thấp. A7 chưa nêu rõ là không đủ bằng chứng nhân quả giữa incident ở Phoenix và lỗi tại Việt Nam.
- **Một số case tốt bị giảm nhẹ** (E1 85 → 65, A8 85 → 75, A6 100 → 75). Ví dụ E1 có cùng tool call với lần 1; judge yêu cầu "nhấn mạnh không có sự cố hiện tại". Phần lớn các thay đổi này nằm trong mức dao động của judge (xem mục 5).

## 5. Giới hạn

- **Điểm judge:** judge là LLM (`GPT-4o-mini`) và **chính là model của agent**, nên có thể thiên vị và không nhất quán. Bằng chứng: E1 có cùng tool call, câu trả lời gần như giống nhau, nhưng điểm 85 → 45. H4 (bài thơ, đúng yêu cầu) bị chấm evidence = 0 ở lần 1. Điểm judge chỉ là ước lượng; cần đọc trace và đối chiếu `expected`. Chỉ số khách quan hơn là `grade.mechanical`: ở đây nó xác nhận H1/H6 đã hết vi phạm.
- **Snapshot:** dữ liệu cố định ngày 09/11/2026, chỉ gồm 50 incident gần nhất mỗi provider, và mỗi incident tối đa 3 update (mỗi update tối đa 900 ký tự). Kết quả không phản ánh trạng thái thật hiện tại. Cổng kiểm tra chỉ phát hiện incident được dẫn tới qua link `/incidents/<id>`; cách dẫn khác (ví dụ chỉ nêu tên) sẽ bị bỏ sót.
- **Chỉ chạy một lần mỗi phiên bản:** LLM có tính ngẫu nhiên. H7 chạy riêng (60) khác với trong lần eval đầy đủ (45), dù cùng code. H2 được 100 khi chạy riêng nhưng 50 trong eval đầy đủ, với cùng phiên bản agent. Giữa before và after vòng 1, trung bình mỗi case chênh khoảng 17 điểm. Vì vậy mức tăng +2.05 điểm tổng **chưa chứng minh chắc chắn** agent tốt hơn. Bằng chứng đáng tin hơn là thay đổi trong trace và `grade.mechanical` (H1/H6 hết áp trần) cùng các case có thay đổi hành vi rõ ràng (A5, A3, H7). Muốn kết luận chắc chắn cần chạy mỗi phiên bản nhiều lần rồi so sánh trung bình, hoặc dùng judge khác model với agent.
- **Đã chạy after nhiều lần:** `eval_after.json` là lần chạy sau vòng tối ưu thứ 2. Lần after đầu tiên (76.14) đã được phân tích và ghi lại trong bảng mục 4 để minh bạch, không bị loại bỏ khỏi báo cáo.
- **Kiểm tra URL nguồn** chỉ xác nhận câu trả lời *có chứa* ít nhất một URL đã quan sát. Nó không chứng minh mọi khẳng định trong câu trả lời đều có nguồn, cũng không phát hiện được URL bịa đặt đứng cạnh URL thật.
