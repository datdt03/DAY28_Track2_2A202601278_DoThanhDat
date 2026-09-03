# ANSWERS.md — Technical Decisions, Trade-offs & Production Gaps

## 1. Trade-offs (Các lựa chọn & Đánh đổi kỹ thuật)

### 1.1. Context Propagation qua Distributed Boundaries (IP01 & IP10)
- **Lựa chọn:** Tự động lan truyền W3C `traceparent` và `idempotency-key` từ HTTP client qua Gateway Envoy, FastAPI, Kafka Headers, Airflow tasks, đến Spark/Delta và vLLM.
- **Đánh đổi:** 
  - *Ưu điểm:* Cho phép truy vết một yêu cầu duy nhất qua toàn bộ 10 điểm kết nối trên Jaeger/LangSmith, dễ dàng root-cause analysis khi có sự cố.
  - *Nhược điểm:* Tăng thêm overhead nhỏ trên per-message payload và yêu cầu tất cả các adapter/client phải tuân thủ chuẩn header injection/extraction.

### 1.2. Idempotence & Replay Safety với Spark Delta MERGE (IP03)
- **Lựa chọn:** Sử dụng `dedupe_latest` để deduplicate batch bản tin theo `idempotency_key` và cặp `(occurred_at, event_id)` trước khi MERGE vào Delta Lake.
- **Đánh đổi:** 
  - *Ưu điểm:* Đảm bảo tuyệt đối không bị trùng lặp dữ liệu (no-data-loss & no-duplicate) ngay cả khi Kafka replay lại bản tin cũ hoặc có retry bão táp.
  - *Nhược điểm:* Tốn thêm chi phí tính toán (CPU/RAM) khi Spark thực hiện MERGE operation so với việc append-only đơn thuần.

### 1.3. Explicit Readiness Semantics & Degraded State (IP07 & IP08)
- **Lựa chọn:** Phân loại rõ rệt 3 trạng thái sức khỏe của platform (`ready`, `degraded`, `not_ready`) dựa trên `readiness_status`.
- **Đánh đổi:**
  - *Ưu điểm:* Tránh hiện tượng cascading failure. Khi dịch vụ bổ trợ (như Feature Store) gặp sự cố, hệ thống chuyển sang `degraded` thay vì sập hoàn toàn (trả về câu trả lời không có feature enrichment thay vì báo lỗi 500).
  - *Nhược điểm:* Cần thiết kế logic fallback ở layer serving để xử lý an toàn khi các feature bổ trợ bị thiếu.

### 1.4. Dynamic Model Promotion & Rollback bằng MLflow Champion Alias (IP06)
- **Lựa chọn:** Sử dụng MLflow Registry `champion` alias thay vì hardcode version ID của model / prompt.
- **Đánh đổi:**
  - *Ưu điểm:* Cho phép hot-swap model release và rollback tức thì trong thời gian thực mà không cần restart service hay sửa mã nguồn (GitOps friendly).
  - *Nhược điểm:* Cần network call đến MLflow server để resolve release thông qua cache layer.

---

## 2. Production Gaps & Future Improvements (Khoảng cách sản xuất & Hướng phát triển)

1. **Auto-scaling Infrastructure cho Inference (vLLM):**
   - *Hiện tại:* vLLM chạy trên GPU đơn hoặc Kaggle endpoint.
   - *Production:* Cần triển khai vLLM trên Kubernetes Cluster kết hợp với KEDA (Kubernetes Event-driven Autoscaling) tự động scale pods theo GPU duty cycle và request queue depth.

2. **Real-time Feature Ingestion cho Feature Store (Feast):**
   - *Hiện tại:* Feast online store được materialize từ offline Delta snapshots theo định kỳ.
   - *Production:* Triển khai Spark Streaming / Flink job để ghi nhận trực tiếp từ Kafka stream vào Redis Online Store nhằm giảm latency tính toán feature xuống sub-second.

3. **Dead Letter Queue (DLQ) Automated Recovery:**
   - *Hiện tại:* Việc kiểm tra và replay DLQ thực hiện qua lệnh CLI `lab28 dlq`.
   - *Production:* Triển khai tự động hóa quy trình DLQ handler kết hợp cảnh báo tự động khi số lượng thông điệp trong DLQ vượt quá ngưỡng quy định (SLO alert).

4. **Security & Zero Trust Networking (mTLS):**
   - *Hiện tại:* Envoy Gateway xử lý SSL termination ở ngoài biên.
   - *Production:* Áp dụng Service Mesh (Istio / Linkerd) với mTLS cho toàn bộ giao tiếp microservice nội bộ để bảo mật tuyệt đối dữ liệu PII và RAG context.

---

## 3. Contribution & Ownership Matrix (Phân công nhiệm vụ theo vai trò)

| Vai trò | Điểm kết nối | Thành viên phụ trách | Nhiệm vụ chính |
|---|---|---|---|
| **Ingestion & Orchestration** | IP01, IP02 | Đỗ Thành Đạt | Triển khai Kafka header propagation (`event_headers`), mã theo dõi W3C trace, Airflow 3 DAG, quản lý DLQ & idempotency key. |
| **Data & ML** | IP03, IP04, IP06 | Đỗ Thành Đạt | Triển khai replay-safe Delta MERGE (`dedupe_latest`), Feast online feature contract (`feast_online_request`), quản lý MLflow Registry champion alias & model provenance. |
| **Serving & Retrieval** | IP05, IP07 | Đỗ Thành Đạt | Triển khai Qdrant deterministic UUID vector indexing, tích hợp vLLM OpenAI-compatible endpoint, RAG prompt grounding & readiness logic (`readiness_status`). |
| **Platform & Observability** | IP08, IP09, IP10 | Đỗ Thành Đạt | Cấu hình Envoy Gateway rate-limiting & header forwarding, Prometheus/Grafana metrics & alerts, OpenTelemetry end-to-end tracing. |
