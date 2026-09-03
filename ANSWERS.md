# Reflection — Day 28 Track 2

## Lựa chọn kỹ thuật và trade-offs

### IP01/IP10 — Kafka headers

`idempotency-key` luôn được mã hóa UTF-8 thành `bytes`; `traceparent` chỉ được gửi
khi có trace hợp lệ. Việc bỏ hẳn header khi không có trace tránh truyền một W3C
header rỗng, nhưng consumer phải chấp nhận message không bắt đầu từ một active
trace.

### IP03 — replay-safe Delta source

Batch được duyệt đúng một lần và lưu event mới nhất theo từng
`idempotency_key`. Cặp `(occurred_at, event_id)` là tiêu chí chọn winner, còn kết
quả được sắp xếp theo key để không phụ thuộc thứ tự Kafka giao message. Cách này
dùng bộ nhớ O(k), với `k` là số key khác nhau trong batch; đổi lại Delta MERGE
luôn nhận source duy nhất trên merge key và replay không tạo dòng trùng.

### IP04 — Feast contract

Request online sử dụng `FEATURE_REFS` từ contract chung thay vì sao chép danh
sách feature. Điều này giảm schema drift giữa API và feature registry, nhưng mọi
thay đổi không tương thích vẫn cần versioning và rollout phối hợp giữa producer,
materialization job và serving.

### IP07/IP08 — readiness

Dependency bắt buộc fail làm hệ thống `not_ready`; dependency tùy chọn fail chỉ
làm `degraded`. Chính sách fail-closed bảo vệ đường phục vụ cốt lõi, còn degraded
mode duy trì availability khi feature enrichment hoặc LLM tùy chọn gián đoạn.
`/health` chỉ là liveness và không được dùng thay `/ready`. Bản thân Envoy cũng
phải tôn trọng ranh giới này: health check chủ động của gateway lúc đầu trỏ vào
`/health` nên không bao giờ ghi nhận việc pod `not_ready`, khiến gateway tiếp
tục route vào một pod thiếu dependency bắt buộc. Sửa thành health check trỏ
`/ready` (200 cho `ready`/`degraded`, 503 cho `not_ready`) để việc loại pod khỏi
rotation khớp đúng ngữ nghĩa readiness đã công bố; `unhealthy_threshold` được
nâng lên 3 lần liên tiếp vì `/ready` gọi thật đến vLLM qua tunnel — một lần
mạng giật không nên bị coi là outage.

## Khả năng khôi phục và vận hành

- Kafka replay được hấp thụ bằng idempotency key và Delta MERGE.
- Message không đọc được được đưa vào DLQ trước khi replay sau khi sửa nguyên
  nhân; không reset volume để mô phỏng recovery.
- MLflow `champion` alias cho phép promotion/rollback không sửa source code.
- Envoy gắn request ID, áp rate limit tại edge và xuất trace sang OTLP.
- Prometheus/Grafana theo dõi golden signals; HPA, PDB và NetworkPolicy thể hiện
  desired state cho production deployment.
- GitOps rollback thực hiện bằng cách revert desired Git revision/image tag rồi
  để Argo CD reconcile, thay vì sửa trực tiếp workload đang chạy.

## Load profile

`load-tests/run_profile.py --requests 200 --workers 8` qua gateway
(`evidence/load-profile.json`): 200/200 request 200 OK, P50 ≈ 1519 ms, P95 ≈
2267 ms, P99 ≈ 2426 ms. Endpoint đo là `/ready`, nên số đo phản ánh đúng chi phí
của bản thân readiness fan-out chứ không phải overhead gateway: so với việc gọi
trực tiếp `api:8000/ready` (cũng ~1.4-1.5 s ổn định), gateway gần như không
cộng thêm latency. Bottleneck nằm ở fan-out phụ thuộc bên trong `/ready` — cụ
thể là vòng round-trip thật tới vLLM qua Cloudflare tunnel (Kaggle), so với
Kafka/Qdrant/Feast/MLflow cục bộ chỉ mất vài ms. Đây là chi phí mạng thật của
một endpoint GPU free-tier, không phải giới hạn của code phục vụ.

## GitOps drift/rollback (kind + Argo CD)

Chạy trên một kind cluster riêng (`kind-lab28`, tách biệt khỏi cluster lab khác
đang có sẵn trên máy) với Argo CD cài mới. `gitops/application.yaml` được trỏ
sang fork thật (`tert62/Track2-...`) ở tag `v3.0.1` để Argo CD sync đúng code đã
sửa, thay vì tag mẫu của lớp. Kết quả (`evidence/gitops-drift-rollback.json`):

- Drift: `kubectl scale deploy/lab28-api --replicas=5` → `selfHeal` tự đưa về
  `replicas: 2` trong ~6 giây, không cần lệnh sync thủ công.
- Promotion: tag `v3.0.2` (đổi `replicas: 2` → `3`) → Argo CD tự sync sang
  revision mới trong ~15 giây.
- Rollback: đổi `targetRevision` trở lại `v3.0.1` → Argo CD reconcile về đúng
  revision cũ (`replicas: 2`) trong ~20 giây; `status.history` giữ đầy đủ 3
  revision đã deploy để audit.

Giới hạn môi trường: image `ghcr.io/vinuni-ai20k/day28-platform-api:3.0.0` là
registry riêng của lớp, trả `403 Forbidden` khi pull ẩn danh — pod ở trạng thái
`ImagePullBackOff` trong suốt demo. Điều này không ảnh hưởng đến việc chứng
minh cơ chế GitOps (desired-state reconciliation, self-heal, rollback đều xảy
ra ở tầng Kubernetes resource, độc lập với việc container có khởi động được
hay không) nhưng nghĩa là chưa chứng minh được pod thật sự Ready/serving trên
cluster này. Cluster cũng thiếu sẵn Gateway API CRD (`gateway.networking.k8s.io`)
— phải cài `gateway-api v1.3.0 standard-install` trước khi Argo CD sync được
`Gateway`/`HTTPRoute`, một bước setup không nằm trong repo mà một cluster
production cần chuẩn bị trước.

## Production gaps

1. Docker Compose là môi trường lab một node; chưa chứng minh multi-zone HA,
   broker replication, disaster recovery hoặc restore từ backup.
2. NetworkPolicy hiện mới giới hạn ở mức namespace/port; production cần mTLS,
   workload identity, secret manager, image signing/SBOM và policy enforcement.
3. Local Envoy rate limit không chia sẻ quota giữa replica; production nên dùng
   global rate-limit service và policy theo tenant/principal.
4. SLO và autoscaling cần được hiệu chỉnh bằng workload đại diện, warm-up, soak
   test và capacity test trên phần cứng production; số đo laptop không phải
   capacity estimate.
5. Data governance còn cần retention/deletion workflow, PII classification,
   access audit, schema migration và lineage hoàn chỉnh.
6. IP07 chỉ được coi là verified khi endpoint GPU trả identity vLLM thật,
   `/v1/models` đúng model và metric `vllm:`. IP10 LangSmith chỉ verified khi có
   credential thật; nếu thiếu phải báo `UNVERIFIED`, không dùng mock. Trong lần
   chạy này cả hai đều verified thật: vLLM (Qwen/Qwen3-4B-Instruct-2507 qua
   Kaggle GPU + Cloudflare tunnel) và LangSmith (project `lab28`,
   `otelcol_exporter_sent_spans` xác nhận nhánh export thứ hai không lỗi).
   Endpoint tunnel free-tier không ổn định (đổi URL, DNS/502/530 gián đoạn vài
   lần trong phiên chạy) — đây là đặc tính của hạ tầng free-tier, không phải
   lỗi code, nhưng có nghĩa gateway health check phải chấp nhận vài lần retry
   trước khi coi một pod là outage thật (xem mục IP07/IP08 ở trên).
7. Feast feature server (`feast serve`) chạy qua gunicorn, và gunicorn luôn
   fork ít nhất một worker process — luồng nền xuất OTel span (dù dùng OTLP/gRPC
   hay OTLP/HTTP) được khởi tạo ở tiến trình cha trước fork nên không tồn tại
   trong worker xử lý request thật. Feast vẫn có `OTEL_SERVICE_NAME=lab28-feast`
   và tự xuất hiện trong danh sách service của Jaeger, nhưng span của nó không
   join đúng vào cùng một trace với request `/ask` đi qua worker đó — vì vậy
   `test_the_trace_spans_the_processes_the_contract_claims` (yêu cầu ≥4 service
   riêng biệt trên một trace) không đạt dù `test_every_required_span_appears_on_one_trace`
   (11 tên span bắt buộc của IP10) vẫn pass. Fix triệt để cần một
   `gunicorn.conf.py` với hook `post_fork` để khởi tạo lại span processor sau
   khi fork — feast CLI hiện không có cách truyền config đó vào `feast serve`.

## Đóng góp cá nhân

Nguyễn Hoàng Thảo Tiên chịu trách nhiệm hoàn thiện bốn boundary do sinh viên sở
hữu trong `integration_tasks.py`: Kafka trace/idempotency headers, Delta batch
deduplication, Feast online request và readiness semantics. Đồng thời thực hiện
fast test suite, lint, integration-matrix validation, portability check,
Kubernetes/GitOps manifest validation và chuẩn bị runbook/evidence checklist cho
demo. Toàn bộ 10 evidence file, 5 live journey (J1–J5), load profile và gate
GPU/LangSmith đã được chạy và ghi nhận trên hạ tầng thật (Docker Compose full
profile + Kaggle GPU vLLM + LangSmith) trong phiên hoàn thiện cuối, bao gồm cả
việc chẩn đoán và sửa 3 vấn đề hạ tầng phát sinh khi chạy thật: Airflow bị kẹt
DAG run do Spark Connect bị OOM-kill, Envoy health check trỏ sai endpoint
(`/health` thay vì `/ready`), và Feast thiếu OTEL instrumentation cho span
IP10.
