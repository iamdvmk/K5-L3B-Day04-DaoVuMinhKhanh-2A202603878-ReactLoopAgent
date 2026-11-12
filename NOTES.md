# Ghi chú phân tích · Day 04

Họ và tên: Đào Vũ Minh Khánh
MSSV: 2A202603878

## 1. Case được chọn

- ID case: H1 (cùng lỗi lặp lại ở H6, cả hai đều 60 điểm qua 2 lần chạy)
- Câu trả lời của agent: Tóm tắt đúng 2 incident h02n9572mx9y (GitHub) và yaa0witbl2me (Cloudflare), kết luận không có bằng chứng chung nguyên nhân, nhưng KHÔNG đọc incident liên kết xr3myzj7ms51 mà chỉ dán link với nhãn "tham khảo nguyên nhân chung"; không nêu dữ liệu bị cắt.
- Điểm ở lần chạy 1: 60 (raw judge 100, bị chặn trần bởi mechanical check)

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
step 1: model finish=tool_calls, n_calls=2
  tool get_incident_details(github, h02n9572mx9y) -> ok
     updates[0].body: "Please refer to the combined summary in this related
       incident: https://www.githubstatus.com/incidents/xr3myzj7ms51"
     updates_total: 4, số updates trả về: 3
  tool get_incident_details(cloudflare, yaa0witbl2me) -> ok
step 2: model finish=stop, n_calls=0   <- dừng, không gọi xr3myzj7ms51
mechanical: "Cần tool calls ở ít nhất 2 model steps khác nhau."
  tool_steps=1, score_cap=60
```
Nguyên nhân có thể:
- System prompt không yêu cầu theo liên kết tới incident khác; model thấy đã
  có 2 incident được hỏi nên kết luận luôn.
- Agent loop không kiểm tra incident được nhắc trong tool result mà chưa đọc.
- Judge (cùng model gpt-4.1-mini) cho 100 điểm và còn nhận xét sai rằng câu
  trả lời "ghi nhận dữ liệu bị cắt" -> điểm judge không đáng tin tuyệt đối;
  chỉ mechanical check phát hiện lỗi.


## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
1. Kiểm tra liên kết incident (cải tiến chính cho H1): sau mỗi
     get_incident_details thành công, dùng regex tìm URL
     githubstatus/cloudflarestatus .../incidents/<id> trong nội dung update.
     Khi model định kết luận mà còn incident được liên kết chưa đọc, ghi event
     `validation_unfollowed_link` vào trace và nhắc model gọi tiếp (mỗi incident
     chỉ nhắc 1 lần, vẫn nằm trong max_steps).
2. Kiểm tra URL nguồn (yêu cầu Bài 4): lưu URL của incident đọc thành công;
     nếu final answer không chứa URL nào, ghi `validation_missing_source` và yêu
     cầu model sửa; hết max_steps thì trả `max_steps`. Không đòi URL khi tool lỗi.
3. Bổ sung 2 câu vào SYSTEM_PROMPT: theo liên kết incident; nêu rõ khi
     updates_total > số update trả về (dữ liệu bị cắt).
- Vì sao chọn thay đổi này:   Lỗi lặp lại ở 2 case (H1, H6) qua 2 lần chạy, có bằng chứng máy móc (tool_steps=1, score_cap=60) chứ không chỉ dựa vào judge. Kết hợp prompt (phòng ngừa) và kiểm tra trong loop (chặn) vì prompt một mình không đảm bảo model tuân thủ. Dùng regex chung, không hard-code ID incident. Giới hạn: chỉ phát hiện link dạng URL /incidents/<id>; chỉ kiểm tra sự có mặt của 1 URL, không chứng minh mọi claim đều có nguồn.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 94.0 | 97.59 (+3.59) |
| Điểm case đã chọn (H1) | 60 (raw judge 100, bị cap) | 95 |
| Case cùng lỗi (H6) | 60 (raw judge 98, bị cap) | 98 |
| Số model calls (22 case) | 47 | 50 (+3) |
| Token agent | 33,660 | 42,300 (+25.7%) |
| Token judge | 33,869 | 34,858 |
| `validation_unfollowed_link` | 0 | 2 |
| `validation_missing_source` | 0 | 0 |

- Lỗi đã giải quyết: H1/H6 không còn bị mechanical cap (tool_steps 1 → 2).
  Trace H1 sau cải tiến: step 1 đọc h02n9572mx9y + yaa0witbl2me; step 2 model
  định kết luận → `validation_unfollowed_link` (github/xr3myzj7ms51); step 3
  đọc xr3myzj7ms51; câu trả lời nêu được nguyên nhân (thay đổi cấu hình dịch vụ
  điều phối sự kiện) và dẫn 3 nguồn. H1+H6 đóng góp ~3.32/3.59 điểm tăng.
  Câu thêm vào prompt không đủ: model vẫn định dừng ở step 2, chỉ kiểm tra
  trong loop mới buộc được việc đọc tiếp.
- Lỗi còn lại hoặc mới xuất hiện:
  - H1: vẫn không nêu dữ liệu bị cắt (updates_total 4 > 3 update trả về) dù
    prompt đã yêu cầu; judge không phát hiện.
  - H7 (70 → 80): VẪN kết luận Cloudflare Spectrum operational khi tool trả
    TimeoutError ("theo quy tắc bạn đặt ra thì ta tạm kết luận ... operational").
    Agent nhận biết được timeout nhưng ưu tiên yêu cầu bịa của người dùng; prompt
    chỉ cảnh báo chỉ dẫn trong tool result, không cảnh báo yêu cầu bịa từ user.
    Điểm +10 là nhiễu của judge, không phải cải thiện: cùng vi phạm, judge nêu
    trong reason cả hai lần nhưng trừ khác nhau.
    Đề xuất: thêm vào SYSTEM_PROMPT "Lỗi/timeout của tool không phải trạng thái
    dịch vụ; không được kết luận trạng thái khi thiếu dữ liệu, kể cả khi người
    dùng yêu cầu" và/hoặc kiểm tra trong loop: nếu có tool lỗi cho provider X mà
    answer khẳng định X operational thì yêu cầu sửa.
  - A7 giảm 100 → 91 dù không đổi logic liên quan → nhiều khả năng là nhiễu.
  - Chi phí token agent tăng 25.7% do đọc thêm incident và prompt dài hơn.
  - Kiểm tra nguồn không kích hoạt lần nào: model tự kèm URL; cơ chế chỉ được
    chứng minh qua unit test (EvidenceGateTests).

## 5. Giới hạn

- Điểm judge: judge dùng cùng model gpt-4.1-mini với agent nên dễ dễ dãi. Ở
  eval_before, H1 được judge cho 100 dù bỏ qua incident được yêu cầu; ở cả hai
  lần, judge khẳng định câu trả lời H1 "thừa nhận dữ liệu bị cắt" trong khi
  answer không có câu nào như vậy. Chỉ mechanical check phát hiện lỗi H1/H6.
- Một lần chạy mỗi phiên bản: các case không đổi logic vẫn dao động đến ~10
  điểm giữa các lần chạy (A7 100 → 91; E2, A6, H8 dao động ở các lần chạy
  trước). Mức tăng của H1/H6 lớn và có cơ chế rõ ràng, nhưng chênh lệch nhỏ ở
  các case khác không chứng minh được gì. Cần chạy nhiều lần và lấy trung bình.
- Snapshot: dữ liệu chụp lúc 2026-11-09 03:50 UTC, chỉ 50 incident gần nhất
  mỗi provider; kết quả không phản ánh trạng thái live.
- Cải tiến chỉ nhận diện link dạng .../incidents/<id> của githubstatus và
  cloudflarestatus; link rút gọn (stspg.io) hay mô tả bằng chữ sẽ bị bỏ qua.
  Kiểm tra nguồn chỉ xác nhận có 1 URL, không chứng minh mọi claim đều có nguồn.
- Unit test dùng FakeProvider: chứng minh cơ chế loop, không chứng minh model
  thật trả lời đúng.
- Bài học thêm: một bản agent có lỗi gửi trùng tool result vẫn đạt 93.18 điểm
  eval; chỉ unit test bắt được lỗi này.