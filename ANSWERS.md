# Lab 28 — Final Answers

## 1. Trade-offs

### Basic profile thay vì full profile trên máy local

Máy local có khoảng 8 GB RAM nên nhóm ưu tiên chạy basic profile để xác minh các integration có thể chạy ổn định trên tài nguyên hiện có.

Basic stack đã xác minh được Kafka, Feast, Qdrant, MLflow, API, Envoy, Prometheus, Grafana, OpenTelemetry Collector và Jaeger.

Trade-off của lựa chọn này là Airflow, Spark và Delta không được chạy local, vì vậy IP02, IP03 và một phần IP04/IP05/IP06/IP10 chưa thể được xác minh end-to-end trên máy hiện tại.

Nhóm không thay đổi cấu hình hoặc giả lập kết quả để làm các integration này trông như đã PASS.

### vLLM là dependency optional trong basic profile

Basic stack được chạy với `LAB28_VLLM_REQUIRE_REAL=false`.

Khi không có GPU/vLLM endpoint thật, readiness trả về `degraded` thay vì `not_ready`.

Điều này cho phép các thành phần còn lại của hệ thống tiếp tục được kiểm thử, nhưng IP07 vẫn được giữ ở trạng thái `UNVERIFIED`.

Nhóm không sử dụng mock OpenAI-compatible endpoint để thay thế vLLM thật.

### Machine-readable evidence ưu tiên hơn screenshot

Nhóm ưu tiên JSON, HTTP response, test output, trace ID và metrics làm evidence chính.

Screenshot chỉ được dùng bổ sung cho các hệ thống có UI như Qdrant, MLflow, Grafana và Jaeger.

Cách này giúp evidence có thể kiểm tra tự động và giảm phụ thuộc vào ảnh chụp thủ công.

### Graceful degradation

Readiness phân biệt ba trạng thái:

- mandatory dependency failure → `not_ready`
- optional dependency failure → `degraded`
- tất cả dependency ready → `ready`

Cách này giúp hệ thống không nhận traffic khi dependency bắt buộc lỗi, nhưng vẫn cho phép phục vụ hạn chế khi dependency optional chưa sẵn sàng.

---

## 2. Production gaps

### Airflow / Spark / Delta chưa được verify trên máy local

Full profile chưa được chạy do giới hạn tài nguyên của máy.

Do đó hiện chưa có live evidence cho:

- IP02 Kafka → Airflow
- IP03 Airflow/Spark → Delta MERGE
- Delta history/time travel
- full asynchronous trace xuyên Kafka consumer → Airflow → Spark → Delta

Các phần này cần được chạy trên máy có tài nguyên cao hơn hoặc môi trường dùng chung.

### vLLM chưa có GPU endpoint thật

IP07 yêu cầu vLLM thật với:

- `/version`
- `/v1/models`
- metrics có prefix `vllm:`
- model/version identity

Máy local hiện không có GPU endpoint phù hợp nên IP07 phải được đánh dấu `UNVERIFIED`.

### Trace hiện mới xác minh synchronous leg

Trace thực tế đã xác minh:

`Envoy → API → Kafka producer`

Trace ID:

`1ce9b921ee904ae49292bea2a39069f5`

Jaeger ghi nhận 8 spans, bao gồm:

- `lab28.gateway.request`
- `lab28.api.ingest`
- `lab28.kafka.produce`

Kafka record tiếp tục mang cùng Trace ID qua W3C `traceparent`.

Các spans thuộc Airflow, Spark, Delta, Feast online lookup, Qdrant query, MLflow release resolution và vLLM completion chưa được xác minh trong cùng một full happy-path trace.

### Load test chưa hoàn tất

Hiện chưa có artifact P50/P95/P99 và bottleneck analysis.

Load test cần được chạy riêng với mức tải phù hợp để tránh gây quá tải máy local 8 GB.

### Failure / recovery evidence chưa hoàn tất

Trong quá trình dựng môi trường có gặp partial Docker resources và Windows console encoding, nhưng đây chưa phải failure/recovery scenario đầy đủ theo yêu cầu submission.

Cần bổ sung một failure scenario có kiểm soát và chứng minh không mất dữ liệu.

### LangSmith chưa được xác minh

Local tracing qua OpenTelemetry và Jaeger đã hoạt động.

Nếu không có `LANGSMITH_API_KEY`, phần LangSmith phải được giữ ở trạng thái `UNVERIFIED`, không được tạo evidence giả.

---

## 3. Verified results

Các validation đã hoàn thành:

- `87 passed` cho starter tests và unit tests
- Ruff: `All checks passed`
- integration matrix static verification: `245 checks passed`
- portability check: PASS
- Kubernetes/GitOps manifest contracts: PASS
- Docker Compose basic và full profile config validation: PASS

Basic runtime đã xác minh:

- Kafka: hoạt động và record có `traceparent`, `idempotency-key`, `schema_version`
- Qdrant: collection `lab28_documents`, 13 points
- Feast: health endpoint HTTP 200
- MLflow: model `lab28-rag-release`, version 2, alias `champion`
- API: health HTTP 200
- Envoy: routing hoạt động và trả `x-request-id`
- Prometheus: required basic targets UP
- Grafana: dashboard `Lab 28 Platform Overview` đọc được metrics thật
- OTel Collector: exported spans được Prometheus scrape
- Jaeger: trace thật từ gateway → API → Kafka producer

Readiness của basic stack là `degraded` vì chỉ thiếu vLLM thật.

---

## 4. Contributions

### [Tên thành viên 1]

- [Điền phần đã thực hiện]
- [Integration/IP phụ trách]
- [Evidence hoặc validation đã thực hiện]

### [Tên thành viên 2]

- [Điền phần đã thực hiện]
- [Integration/IP phụ trách]
- [Evidence hoặc validation đã thực hiện]

### [Tên thành viên 3]

- [Điền phần đã thực hiện]
- [Integration/IP phụ trách]
- [Evidence hoặc validation đã thực hiện]

> Cập nhật phần này theo đóng góp thực tế của từng thành viên. Không ghi đóng góp chưa thực hiện.
