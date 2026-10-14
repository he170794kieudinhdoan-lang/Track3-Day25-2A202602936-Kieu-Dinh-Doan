# Lab 25B — Report

**Họ tên / nhóm:** Kiều Đình Đoàn - 2A202602936  **Commit:** `21ba930`  **Điểm tự động (`make grade`):** 84.0 / 88 (88/88 khi tính cả phần hidden SLO)

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 11.911 | 1.059 | 0.008 | 2.161 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.738 | 1.058 | 0.034 | 2.141 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.018 | 2.196 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Băng G4 và G6. Thời gian xử lý trung vị của provider là 1.2 đến 1.8 giây. Mức 8.0 giây đủ dài để hoàn thành lượt gọi bình thường và đủ ngắn để kịp failover trong deadline 30 giây của người dùng. |
| max_attempts_per_provider | 2 | Băng G2 và G3. Cho phép đúng 1 lần thử lại trên cùng provider. Nếu lỗi vẫn tiếp diễn, client chuyển ngay sang provider kế tiếp để tránh nghẽn cục bộ. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Băng G1 và G2. Khởi đầu với 0.25 giây giúp xử lý nhanh lỗi mạng thoáng qua. Mức trần 4.0 giây kết hợp jitter ngẫu nhiên giúp phân tán lưu lượng và không làm trễ deadline 30 giây. |
| retry_budget_ratio | 0.1 | Băng G2. Giới hạn số lượt thử lại tối đa ở mức 10% tổng số request gửi đi, triệt tiêu hiện tượng retry storm khi provider chính gặp sự cố quá tải. |
| breaker.window_size / min_calls | 20 / 10 | Băng G6. Cửa sổ 20 kết quả gần nhất phản ánh nhanh trạng thái dịch vụ. Yêu cầu tối thiểu 10 cuộc gọi ngăn breaker kích hoạt sai trong điều kiện lưu lượng thấp. |
| breaker.failure_rate_threshold | 0.5 | Băng G6. Ngắt mạch khi tỷ lệ lỗi chạm 50%, bảo vệ hệ thống khỏi provider đang sập hoàn toàn (503/timeout). |
| breaker.slow_call_s | 5.0 | Băng G4. Thời gian phản hồi thông thường dưới 2 giây. Cuộc gọi kéo dài từ 5 giây trở lên báo hiệu hàng đợi phía provider đang nghẽn. |
| breaker.slow_rate_threshold | 0.6 | Băng G4. Khi 60% cuộc gọi bị chậm, breaker lập tức mở để chuyển hướng sang provider nhanh hơn, giảm p95 từ 13.76 giây xuống 5.56 giây. |
| breaker.open_duration_s | 10.0 | Băng G2. Giữ trạng thái OPEN trong 10 giây để provider có thời gian xả hàng đợi nội bộ trước khi nhận probe thử nghiệm. |
| breaker.half_open_max_probes | 2 | Băng G2. Giới hạn 2 probe đồng thời trong trạng thái HALF_OPEN, ngăn chặn thundering herd làm sập lại provider vừa hồi phục. |

Sơ đồ quyết định của `on_error`:

```
                    +---------------------------+
                    |  Loi tra ve tu Provider   |
                    +-------------+-------------+
                                  |
                                  v
                    +---------------------------+
                    |  Phan loai loi (classify) |
                    +-------------+-------------+
                                  |
         +------------------------+-----------------------+
         |                        |                       |
         v                        v                       v
  [CLIENT_ERROR]          [QUOTA / RATE_LIMIT]      [SERVER / TIMEOUT / OVERLOAD]
         |                        |                       |
  Giai phong Permit        Giai phong Permit        Ghi nhan on_failure vao Breaker
         |                        |                       |
         v                        v                       v
  GiveUp("client_error")   Dat cooldown provider    Con luot tren provider (tries < 2)
  Khong bao gio retry      (300s quota hoac         va con ngan sach retry (budget)
  Khong bao gio failover   retry_after rate limit)  va con du thoi gian truoc deadline?
                                  |                       |
                                  |                 +-----+-----+
                                  |                 |           |
                                  |               (Dung)      (Sai)
                                  |                 |           |
                                  |                 v           v
                                  |           Gui lai cung    Chuyen sang provider tiep
                                  |           provider voi    theo trong ladder (failover)
                                  |           full jitter     ngay lap tuc (delay = 0)
                                  |                 |           |
                                  +-------->--------+           |
                                                    |           |
                                                    v           v
                                            Goi _route tim provider tiep theo
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Khi vô hiệu hóa circuit breaker (`min_calls: 1000000000`) và đặt `max_attempts_per_provider: 5`, hệ thống phụ thuộc hoàn toàn vào retry budget. Trong kịch bản G2 có bật budget, availability đạt 0.9983, amplification giữ ở 1.378 và recovery_s là 0.0. Khi tắt budget, availability tụt xuống 0.9666, amplification vọt lên 1.675, retry_ratio tăng từ 0.058 lên 0.495 và thời gian hồi phục kéo dài 25.0 giây. Khi circuit breaker hoạt động chuẩn, budget vẫn đóng vai trò chốt chặn cuối cùng bảo vệ downstream nếu breaker bị cấu hình sai ngưỡng hoặc gặp lỗi phân loại.

2. **`attempt_timeout_s: 30`.** Ở kịch bản G6 với timeout 30 giây, availability giảm mạnh từ 1.0000 xuống 0.9488 với 107 lượt give up do hết deadline. Tuy nhiên, latency p95 trên các lượt thành công lại giảm từ 5.961 giây xuống 5.410 giây vì các request bị treo lâu đã bị loại bỏ khỏi thống kê (survivorship bias). Kịch bản G4 hầu như không thay đổi (availability 1.0000, p95 từ 5.562 lên 5.608 giây) do breaker phát hiện slow call ở mốc 5.0 giây và failover sớm. Timeout đúng phải dựa trên thời gian cần thiết để failover kịp trước deadline người dùng, thay vì neo theo deadline tổng thể.

3. **Bỏ lọc tool calling khỏi `ladder()`.** Ở kịch bản G6, việc bỏ điều kiện `supports_tools` khiến client định tuyến 462 cuộc gọi vào `gamma-mini`, mô hình giá rẻ không hỗ trợ function calling. Hậu quả là sinh ra 256 lỗi `incompatible_routes`, kéo availability toàn hệ thống từ 1.0000 xuống 0.8927. Đứng ở góc nhìn product manager, người dùng bấm nút yêu cầu kiểm tra đơn hàng nhưng bot trả lời thất bại do hệ thống gửi nhầm việc chuyên môn cho mô hình không có năng lực. Điều này vừa làm hỏng trải nghiệm người dùng, vừa lãng phí chi phí API vô ích.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1496700 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Chỉ có idempotency key là chưa đủ vì khi tiến trình worker gặp sự cố và khởi động lại, worker mới không có bộ nhớ trạng thái và sẽ chạy lại toàn bộ quy trình từ đầu, gây lãng phí tài nguyên tính toán LLM (1.123 cuộc gọi dư thừa mỗi lần crash). Chỉ có checkpoint cũng chưa đủ vì nếu phản hồi mạng bị mất giữa chừng sau khi trừ tiền, lần gọi lại không có idempotency key sẽ tạo ra một giao dịch trừ tiền thứ hai tại cổng thanh toán. Nếu ghi checkpoint trước khi gọi payment, sự cố sập worker ngay tại thời điểm gọi cổng thanh toán sẽ khiến worker khởi động lại tưởng rằng bước thanh toán đã xong và bỏ qua, dẫn đến khách hàng không nhận được tiền hoàn (lost refund).

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 phút |
| provider_outage | page | page | page | 4 phút |
| deploy_blip | none | page | none | Khong bao dong |
| slow_leak | ticket | none | ticket | 1337 phút |
| degraded_evening | page | page | page | 291 phút |
| night_spike | none | page | none | Khong bao dong |
| small_bump | none | page | none | Khong bao dong |

Error budget 30 ngày ghi nhận 25,901,755 lượt request, số lỗi cho phép là 129,508.8, thực tế có 55,618 lỗi. Tỷ lệ availability đạt 0.997853, đã tiêu thụ 42.95% ngân sách và còn lại 57.05%, do đó không cần đóng băng release. Sự cố `small_bump` (20 phút ở mức 12% lỗi) không nên đánh thức kỹ sư on-call lúc 11 giờ trưa vì mức tiêu hao này chỉ chiếm một phần rất nhỏ của ngân sách 30 ngày và sẽ tự chấm dứt trước khi đe dọa cam kết SLO.

## 6. Postmortem — Băng G2 với client naive (blameless)

Hình vẽ trực quan hóa diễn biến sự cố được lưu tại `reports/gameday/G2_overload_storm.png`.

**Tóm tắt (3 câu):** Lúc 120 giây, provider chính alpha bị suy giảm 75% năng lực xử lý khiến hàng loạt yêu cầu bị quá tải và trả về lỗi 529. Client naive phản ứng bằng cách thử lại lập tức mà không có backoff hay giới hạn ngân sách, tạo ra cơn bão lưu lượng khuếch đại gấp 9.9 lần làm nghẽn hoàn toàn hàng đợi của alpha. Hậu quả là hệ thống rơi vào trạng thái tê liệt tự duy trì (metastable failure), khiến dịch vụ mất thêm 155 giây sau khi alpha đã hồi phục dung lượng thì mới có thể xử lý bình thường trở lại.

**Ảnh hưởng:** Sự cố khiến 1,293 trên tổng số 2,377 lượt người dùng thất bại, availability tụt xuống 0.4560. Toàn bộ gián đoạn kéo dài 335 giây (từ giây 120 đến giây 455). Số lỗi này tiêu tốn 1,293 lượt, tương đương 2.49% ngân sách lỗi của cả tháng chỉ trong vòng chưa đầy 6 phút.

**Timeline (giây ảo lấy từ CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`):**

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | alpha mất 75% năng lực xử lý | calls_ok giảm từ 10 xuống 7, attempts trên alpha ở mức 5 |
| 122 | Tỷ lệ thành công bắt đầu rơi tự do | calls_ok giảm về 0, hàng đợi bắt đầu tích lũy |
| 136 | Lượng attempt bắt đầu bùng nổ do retry | calls_ok = 3, attempts trên alpha tăng vọt lên 53 |
| 144 | Đỉnh điểm của cơn bão retry storm | attempts trên alpha đạt mức kỷ lục 254 req/s, calls_ok = 0 |
| 200 | Hệ thống kẹt cứng trong trạng thái quá tải | attempts trên alpha dao động 140 đến 180 req/s, calls_ok = 0 |
| 300 | alpha hồi phục 100% dung lượng hạ tầng | calls_ok = 0, attempts trên alpha vẫn ở mức 115 req/s |
| 350 | Hàng đợi cũ của alpha vẫn nghẽn vì request quá hạn | calls_ok = 0, attempts trên alpha duy trì 10 đến 25 req/s |
| 453 | Hàng đợi alpha bắt đầu xả hết các request quá hạn | calls_ok phục hồi lên 4, attempts trên alpha là 7 |
| 455 | Hệ thống chính thức hồi phục hoàn toàn | calls_ok ổn định ở mức 4, recovery_s = 155 giây sau mốc 300 |

**Trigger vs root cause:** Trigger là sự cố hạ tầng tại giây 120 khiến alpha mất 75% năng lực phục vụ. Root cause là kiến trúc client naive thiếu exponential backoff, thiếu circuit breaker và không có retry budget. Khi alpha trả về lỗi, client lập tức gửi lại toàn bộ request thất bại, khiến lưu lượng đổ vào alpha tăng vọt lên 254 req/s (vượt xa sức chứa 20 req/s của alpha). Kể cả khi alpha đã có lại 100% máy chủ ở giây 300, hàng đợi vẫn chứa đầy request cũ sắp hoặc đã quá hạn, tạo ra vòng lặp tự duy trì khiến máy chủ bận xử lý request vô giá trị và tiếp tục từ chối request mới.

**Điều gì giúp / điều gì làm tệ hơn:**
- Điều làm tệ hơn: Client naive thử lại liên tục không có độ trễ, không tôn trọng Retry-After và không ngắt mạch khi tỷ lệ lỗi vượt ngưỡng.
- Điều giúp ích: Cơ chế deadline 30 giây giúp dọn dẹp các cuộc gọi quá cũ, cuối cùng giúp hàng đợi xả sạch tại giây 455. Resilient client giải quyết triệt để vấn đề này nhờ circuit breaker ngắt mạch sau 10 cuộc gọi lỗi và retry budget chặn lưu lượng khuếch đại ở mức 1.059.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai ResilientClient với Circuit Breaker và Retry Budget cho toàn bộ dịch vụ | Prevent | Platform Team | Chạy kiểm thử Game Day G2 đạt availability 1.00 và amplification dưới 1.10 |
| 2 | Cấu hình cảnh báo multi-window burn-rate (1h và 6h) trên hệ thống Prometheus | Detect | SRE Team | Tự động kích hoạt cảnh báo page trong vòng 5 phút khi tỷ lệ lỗi tăng vọt |
| 3 | Thêm cơ chế deadline propagation và load shedding ở cổng API Gateway | Mitigate | Backend Team | Request quá hạn deadline tự động bị hủy trước khi đưa vào hàng đợi xử lý |

## 7. Một điểm yếu còn lại

Client hiện tại vẫn phụ thuộc vào danh sách fallback cố định (`alpha -> alpha-eu -> beta -> gamma-mini`). Khi alpha gặp sự cố kéo dài, toàn bộ lưu lượng được chuyển hướng sang `beta` có giá 1.4 $/call (đắt hơn 40% so với alpha) và `alpha-eu` có độ trễ mạng cao hơn. Nếu cả bốn provider cùng bị suy giảm hiệu năng nhẹ hoặc mạng diện rộng bị nghẽn, client sẽ thử tuần tự qua từng nhà cung cấp, làm gia tăng latency tổng thể và tiêu tốn chi phí. Giải pháp cải tiến là tích hợp thuật toán phân bổ tải theo trọng số động (load balancing dựa trên EWMA latency và error rate thời gian thực), kết hợp kiểm tra độ suy thoái đồng thời để trả về kết quả dự phòng nhẹ (degraded response) thay vì thử hết thang ladder.

## 8. Cache (Part G)

Số liệu thực tế trích xuất từ `reports/cache/resilient.json` và `reports/cache/cached.json`:

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Ngưỡng 0.80 sinh ra 157 câu trả lời sai do nhầm lẫn giữa các intent gần nhau. Ngưỡng 0.92 loại bỏ hoàn toàn câu trả lời sai (wrong_answers = 0) trong khi vẫn giữ hit ratio ở mức 0.652. |
| ttl_s | 60 | Kịch bản có thay đổi âm thầm tại giây 480 (`silent_updates` không có webhook). Mức TTL 60 giây giới hạn số câu trả lời lỗi thời ở mức 3 câu (dưới chỉ tiêu tối đa 15 câu). |
| max_stale_s | 300 | Toàn bộ 4 provider sập trong khoảng 240 đến 360 giây (120 giây). Mức stale window 300 giây cho phép phục vụ 157 câu trả lời cũ, nâng faq_availability từ 0.8424 lên 0.9645. |

Thí nghiệm:

1. **Ngưỡng.** Khi chạy kịch bản C1 với `similarity_threshold` = 0.80, cache hit ratio đạt 0.666 nhưng sinh ra 157 câu trả lời sai (`wrong_answers_by_reason: {'other_question': 157}`). Các cặp câu hỏi bị nhầm là những câu có khoảng cách ngữ nghĩa gần như quy định hoàn tiền (`refund_policy`) với thời gian hoàn tiền (`refund_time`), hoặc phí ship nội thành (`shipping_fee_city`) với phí ship tỉnh (`shipping_fee_province`). Ở ngưỡng 0.92, số câu trả lời sai giảm về 0, hit ratio đạt 0.652 và chi phí đạt 0.915 $/turn. Tăng ngưỡng lên 0.97 làm giảm hit ratio xuống 0.581 mà không mang lại thêm lợi ích về độ chính xác, đồng thời chi phí tăng lên 0.964 $/turn.

2. **TTL.** Với `ttl_s` = 30 giây, số câu trả lời cũ là 2 và hit ratio là 0.592. Với `ttl_s` = 60 giây, số câu trả lời cũ là 3 và hit ratio đạt 0.652. Khi tăng `ttl_s` lên 300 giây, số câu trả lời cũ tăng vọt lên 95 câu (vượt trần 15 câu cho phép). Webhook chỉ phát huy tác dụng khi nguồn dữ liệu chủ động gửi thông báo cập nhật, hoàn toàn bất lực trước các thay đổi âm thầm (như đơn vị vận chuyển đổi giá cước trực tiếp trên bảng giá). Do đó, TTL là cơ chế an toàn duy nhất để loại bỏ dữ liệu cũ khi không có tín hiệu webhook.

3. **Stale-if-error.** Khi đặt `max_stale_s: 0` (tắt tính năng phục vụ dữ liệu cũ khi lỗi), `faq_availability` giảm mạnh từ 0.9645 xuống 0.8733 và số `stale_served` bằng 0. Lý do là trong đợt sập toàn diện 120 giây (từ giây 240 đến 360), 157 câu hỏi thường gặp đã bị từ chối phục vụ thay vì được trả lời bằng bản ghi lưu tạm thời.

Trả lời:

- Câu hỏi cá nhân (chẳng hạn "Đơn #48213 của tôi đang ở đâu?") không được lưu vào cache vì embedding của các câu hỏi này chỉ khác nhau ở mã đơn hàng, dẫn đến độ tương đồng ngữ nghĩa lên tới 0.99. Nếu lưu cache, câu hỏi của người dùng này sẽ trả về thông tin lộ đơn hàng của người dùng khác, gây rò rỉ thông tin riêng tư nghiêm trọng. Kể cả khi cache riêng theo từng người dùng, trạng thái đơn hàng thay đổi liên tục theo thời gian thực nên dữ liệu lưu cache sẽ gây sai lệch thông tin xử lý đơn.
- Khi toàn bộ provider gặp sự cố và cache vẫn phục vụ người dùng khiến tỷ lệ thành công của hệ thống vẫn ở mức cao, dashboard người dùng vẫn hiển thị màu xanh. Để đội ngũ trực vận hành (on-call) nhận biết được sự cố ngầm, hệ thống giám sát cần thiết lập cảnh báo dựa trên tỷ lệ lỗi gọi trực tiếp tới provider (backend error rate) và số lượng yêu cầu phải phục vụ bằng cache cũ (`stale_served` tăng vọt).
