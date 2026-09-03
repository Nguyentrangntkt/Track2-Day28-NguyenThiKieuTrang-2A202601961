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

### Integration suite bị chặn bởi dependency Airflow của full profile

Lệnh final validation `uv run pytest integration-tests -m "not gpu and not langsmith" -q` tạo 56 setup errors vì fixture yêu cầu Airflow tại `localhost:8082`.

Airflow thuộc full profile và không được khởi động trên máy local 8 GB theo giới hạn thực hiện của bài. Đây là environment blocker của live integration suite, không phải code failure và không được sửa bằng cách thay test hoặc giả lập service.

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

### Load test chỉ có baseline nhẹ của basic stack

Artifact `evidence/load-profile-basic.json` ghi nhận 10 request tuần tự qua Envoy `/ready`: P50 368.45 ms, P95/P99 1586.11 ms, error rate 0% và throughput 1.918 req/s.

Đây chỉ là baseline phù hợp máy 8 GB, không phải kết luận năng lực production. Bottleneck quan sát được là tail latency của readiness dependency fan-out; metrics hiện chỉ phân rã theo route nên không quy kết chính xác cho một dependency.

### Failure / recovery đã xác minh cho Qdrant

`evidence/failure-recovery-qdrant.json` ghi lại Qdrant được stop tạm thời: gateway `/ready` chuyển HTTP 503 `not_ready`; sau khi start lại, collection trở về HTTP 200 với đúng 13 points trước/sau.

Không dùng `down -v` hoặc xóa volume. Scenario này chứng minh recovery và không mất dữ liệu Qdrant trong phạm vi basic stack; data-plane Airflow/Delta recovery vẫn chưa verify.

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
- Final local unit suite: `83 passed`
- Final integration suite: BLOCKED bởi dependency Airflow/full profile trên máy 8 GB; không phải code failure

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

Project được thực hiện cá nhân. Tôi chịu trách nhiệm toàn bộ phần implementation, validation, runtime verification, evidence collection và submission.