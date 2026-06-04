# SUBMIT - ShopX Incident Lab

## Tóm tắt bài làm

Bài lab này phân tích một sự cố trên `cart-service` bằng cách kết hợp metrics và logs để trả lời ba câu hỏi chính:

- `WHEN`: anomaly bắt đầu khi nào
- `WHERE`: service, metric, và log nào cho thấy tín hiệu sớm nhất
- `WHAT`: cơ chế lỗi gốc là gì

Mục tiêu của bài không chỉ là tìm một điểm bất thường, mà là dựng được chuỗi bằng chứng theo đúng tư duy AIOps / SRE: từ dấu hiệu sớm, đến lan truyền sự cố, đến hậu quả cuối cùng như `OOMKilled`, restart loop, và timeout dây chuyền.

## Deliverables

- [`scripts/run_pipeline.py`](./scripts/run_pipeline.py)
- [`FINDINGS.md`](./FINDINGS.md)
- [`SUBMIT.md`](./SUBMIT.md)
- [`notebooks/01_metrics_anomaly_detection.ipynb`](./notebooks/01_metrics_anomaly_detection.ipynb)
- [`notebooks/02_log_parsing_drain3.ipynb`](./notebooks/02_log_parsing_drain3.ipynb)

## Dữ liệu và phạm vi phân tích

Bộ dữ liệu gồm metrics và logs cho các service chính của hệ thống:

- `cart-service`
- `api-gateway`
- `order-service`
- `payment-service`
- `product-service`

Trong bài này, `cart-service` là nguồn phân tích trung tâm vì nó có dấu hiệu suy giảm tài nguyên rõ nhất:

- `memory_usage_bytes`
- `memory_limit_bytes`
- `jvm_gc_pause_ms_avg`
- `http_p99_latency_ms`
- `http_5xx_rate`
- `container_restart_count`

Các service còn lại chủ yếu được dùng để chứng minh tác động lan truyền.

## Khi nào anomaly bắt đầu

Từ kết quả notebook và `FINDINGS.md`, có thể chia mốc thời gian như sau:

- Tín hiệu log sớm bắt đầu xuất hiện từ khoảng `2026-06-01 06:32:33Z`
- Memory anomaly bền vững rõ hơn từ khoảng `2026-06-01 16:20:30Z`
- GC anomaly bền vững từ khoảng `2026-06-01 17:24:30Z`
- Hard failure xuất hiện vào khoảng `2026-06-01 19:59:00Z`
- `container_restart_count` tăng mạnh ngay sau đó, cho thấy restart loop đã bắt đầu

Ý nghĩa vận hành:

- Giai đoạn đầu là silent signal
- Giai đoạn giữa là degradation
- Giai đoạn cuối là failure và cascade

## Service, metric, và log nào nổi bật nhất

### Service chính

- `cart-service` là nguồn gốc chính của sự cố
- `api-gateway`, `order-service`, và `payment-service` cho thấy sự cố lan truyền ra ngoài
- `product-service` được giữ làm đối chứng

### Metric quan trọng nhất

- `memory_usage_bytes`
- `jvm_gc_pause_ms_avg`
- `http_p99_latency_ms`
- `http_5xx_rate`
- `container_restart_count`

### Log signal quan trọng nhất

- `ProductCatalogCache eviction failed: heap pressure too high`
- `GC overhead limit warning: pause=... heap=...%`
- `OutOfMemoryError imminent: available heap < ...%`
- `Container OOMKilled: memory limit exceeded`
- `Application starting up ...`

## Trace_id analysis

Notebook logs đã được mở rộng để phân tích `trace_id` theo hướng đúng với dữ liệu thực tế:

- `trace_id` được dùng để xem các event liên quan bên trong từng service
- `trace_id` giúp xác định các chuỗi event dày đặc, đặc biệt trong `cart-service`
- Dữ liệu hiện tại không có `trace_id` chung xuyên suốt giữa `cart-service` và `order-service`, nên không ép một correlation giả giữa các service

Kết luận thực tế:

- `trace_id` hữu ích cho nội bộ service log
- Cross-service correlation nên chứng minh bằng metrics upstream và incident timeline, không nên bịa trace chung khi dữ liệu không hỗ trợ

## Error rate theo 30 phút

Notebook logs có thêm phân tích `WARN + ERROR + FATAL` theo cửa sổ 30 phút:

- Bài này dùng cửa sổ 30 phút để làm mượt noise và nhìn rõ xu hướng
- Cách này phù hợp với incident kiểu drift rồi mới bùng nổ
- Khi error rate tăng cùng lúc với memory/GC anomaly, đó là dấu hiệu rất mạnh của degradation

Biểu đồ này giúp trả lời:

- service nào bắt đầu xấu đi trước
- error spike xuất hiện trước alert bao lâu
- có phải chỉ là một spike ngẫu nhiên hay là một xu hướng bền vững

## Incident timeline

Mình đã thêm chart timeline để gom bằng chứng thành một dòng thời gian duy nhất:

- tín hiệu cache eviction failure
- window error rate
- dấu hiệu OOM imminent
- `OOMKilled`
- restart loop
- downstream timeout/refused

Đây là biểu đồ quan trọng nhất để trình bày cho reviewer vì nó nối `WHEN / WHERE / WHAT` thành một câu chuyện duy nhất thay vì chỉ là các chart rời rạc.

## Bảng mapping WHEN / WHERE / WHAT

| File | WHEN | WHERE | WHAT |
|---|---|---|---|
| `metrics/cart-service.csv` | Rất quan trọng | Rất quan trọng | Rất quan trọng |
| `logs/cart-service.log.jsonl` | Quan trọng | Rất quan trọng | Rất quan trọng |
| `metrics/api-gateway.csv` | Quan trọng | Hỗ trợ | Hỗ trợ |
| `metrics/order-service.csv` | Quan trọng | Hỗ trợ | Hỗ trợ |
| `metrics/payment-service.csv` | Quan trọng | Hỗ trợ | Hỗ trợ |
| `logs/order-service.log.jsonl` | Hỗ trợ | Quan trọng | Hỗ trợ |
| `metrics/product-service.csv` | Đối chứng | Đối chứng | Đối chứng |

## Tóm tắt root cause

Hypothesis chính của bài:

1. `cart-service` bị heap pressure tăng dần
2. JVM GC bắt đầu hoạt động nặng hơn
3. Latency tăng và error rate tăng
4. Pod tiến tới `OOMKilled`
5. Container restart liên tục
6. Downstream services bắt đầu timeout hoặc bị ảnh hưởng dây chuyền

Tóm lại, root cause hợp lý nhất là một vấn đề về memory pressure / GC thrashing trong `cart-service`, dẫn đến failure ở tầng container và cascade sang các service liên quan.

## So sánh 2 phương pháp anomaly detection

Bài này dùng 2 phương pháp:

- `Rolling Z-score`
- `Isolation Forest`

Nhận xét:

- `Rolling Z-score` dễ giải thích hơn cho on-call và phù hợp để đánh dấu silent window
- `Isolation Forest` nhạy với bất thường đa biến, nhưng có thể flag rất sớm và cần diễn giải cẩn thận

Trong bài này:

- Z-score phù hợp hơn cho việc kể câu chuyện vận hành
- Isolation Forest phù hợp hơn như một lớp kiểm tra bổ sung cho anomaly đa biến

## Reflection

Nếu được giao làm Platform Engineer cho một startup khoảng 50 service vừa raise Series A, mình sẽ ưu tiên cách làm sau:

- Dùng observability đủ ba lớp: metrics, logs, traces
- Tập trung vào các signal thực sự giúp debug nhanh: memory, GC, latency, 5xx, restart count
- Dùng pipeline log template thay vì đọc raw log thủ công
- Dùng trace_id để điều tra trong phạm vi dữ liệu cho phép
- Không cố ép cross-service trace correlation nếu data không có shared trace_id

Về chiến lược build hay buy:

- Nếu team còn nhỏ và cần tốc độ: nên `buy` một phần nền tảng quan sát như Datadog hoặc dịch vụ tương đương
- Nếu cần kiểm soát chi phí và có năng lực vận hành cao hơn: có thể `build` dần các phần chuyên sâu như dashboard nội bộ, pipeline enrichment, hoặc anomaly rules

Khuyến nghị của mình cho startup 50-service:

- `Buy` phần core observability trước để giảm time-to-detect và time-to-resolve
- `Build` dần những phần domain-specific AIOps nơi dữ liệu nội bộ tạo giá trị thật sự

## Cách chạy

```bash
python scripts/run_pipeline.py
```

## Ghi chú

- File này được viết bằng tiếng Việt và lưu theo UTF-8
- Các timestamp trong bài dùng múi giờ UTC
- Các chart trong notebook được lưu vào thư mục `artifacts/`
