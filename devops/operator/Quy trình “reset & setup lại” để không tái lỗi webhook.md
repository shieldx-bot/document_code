# Quy trình “reset & setup lại” để không tái lỗi webhook

Bạn muốn “chạy lại từ đầu” sao cho **không dính 3 lỗi** (refused / TLS unknown authority / sai path) và đến đoạn **đăng ký path webhook** là chạy ngon. Dưới đây là checklist + quy trình chuẩn cho repo hiện tại của bạn.

## Trước khi chạy lại: hiểu đúng “path webhook” của Pod
Với Pod (core API, `apiGroup=""`), controller-runtime **mặc định** đăng ký path dạng:

- `/validate--v1-pod`  (có **double dash**)

Repo của bạn hiện đã đúng:
- Marker trong pod_webhook.go là `/validate--v1-pod`
- manifests.yaml cũng đang gọi `/validate--v1-pod`

Điểm mấu chốt để không lỗi “404 the server could not find…” là: **ValidatingWebhookConfiguration trong cluster phải trỏ đúng path này**.

---

## Quy trình “reset & setup lại” để không tái lỗi

### 1) Dọn sạch các webhook “rác”/cũ trong cluster
Lỗi *connect refused* lúc đầu là do có webhook config trỏ nhầm `webhook-service.system`. Nếu bạn cài lại nhiều lần, nên dọn sạch trước.

Khuyến nghị: dọn theo 2 mức:

- **Mức sạch sẽ nhất**: xoá hẳn namespace operator (sẽ xoá Deployment/Service/Secret/Certificate…)
- **Mức tối thiểu**: xoá các `ValidatingWebhookConfiguration` cũ không dùng

Nếu bạn muốn “cài lại từ đầu” đúng nghĩa, cứ xoá namespace `image-policy-operator-system` + xoá các `ValidatingWebhookConfiguration` cũ còn sót (đặc biệt cái tên chung chung kiểu `validating-webhook-configuration`).

### 2) Đảm bảo cert-manager có “CA chain” đúng (tránh lỗi TLS unknown authority)
Repo bạn hiện đang đi đúng hướng: tạo CA cert + CA issuer:

- certificate-ca.yaml tạo `Certificate` `isCA: true`
- issuer-ca.yaml tạo `Issuer` kiểu `ca: secretName: ca-cert`
- và kustomization.yaml đã include 2 file này

Điều kiện để APIServer tin webhook:
- `ValidatingWebhookConfiguration` có annotation `cert-manager.io/inject-ca-from: <ns>/<serving-cert>`
- cert-manager sẽ inject `caBundle` **là CA thật** (CA:TRUE)
- serving certificate (tls.crt) của webhook server phải được ký bởi CA đó

Nếu bạn deploy bằng default, phần injection này sẽ được xử lý.

### 3) Deploy operator bằng **kustomize overlay chuẩn** (tránh tạo thêm webhook xung đột)
Bạn nên deploy bằng:

- `kubectl apply -k config/default`

Và **tránh apply** các YAML “demo/handwritten” khác tạo thêm webhook config (ví dụ file mẫu kiểu `validating-webhook-configuration` hoặc “signed-image-webhook”) vì dễ gây:
- 2 webhook cùng target 1 service nhưng caBundle khác nhau → TLS fail
- 2 webhook cùng rule/pod → khó debug

### 4) Kiểm tra nhanh sau deploy (3 check là ra bệnh ngay)
Sau khi apply default, kiểm tra:

1. Webhook config đang trỏ đúng path:
   - `... clientConfig.service.path` phải là `/validate--v1-pod`

2. Service webhook có endpoints:
   - service `image-policy-operator-webhook-service` phải có Endpoints trỏ tới pod manager (port target 9443)

3. Webhook config có `caBundle` được inject (không rỗng) và là CA (CA:TRUE)

### 5) Cuối cùng mới apply policy / tạo Pod test
Khi 3 check trên ok thì:
- `kubectl apply -f k8s/policy` sẽ không bị chặn ở admission transport/TLS nữa.
- Nếu bị chặn thì lúc này là do **logic validate** của bạn (expected/desired), không phải lỗi infra.

---

## “Đến đoạn đăng ký path webhook” thì chạy lại từ đâu trong code?
Chỗ “đăng ký webhook handler” nằm ở:

- main.go: gọi `webhookv1.SetupPodWebhookWithManager(mgr)` (trừ khi `ENABLE_WEBHOOKS=false`)
- pod_webhook.go: `SetupPodWebhookWithManager` dùng builder `ctrl.NewWebhookManagedBy(mgr).For(&corev1.Pod{})...`

Controller-runtime sẽ map Pod validator vào path mặc định (và log ra), bạn đã thấy log kiểu:
- `Registering webhook {"path": "/validate--v1-pod"}`

Nếu sau này bạn muốn **đỡ phụ thuộc vào double-dash**, bạn có thể chủ động đặt path khác (ví dụ `/validate-v1-pod`) bằng cách:
- chỉnh marker + regenerate manifests (controller-gen), **hoặc**
- cấu hình flags `--validation-path` (ít phổ biến hơn trong scaffold này)

Nhưng hiện tại: cứ theo mặc định `/validate--v1-pod` là ổn và repo đã align.

---

## Try it (chuỗi thao tác “cài lại sạch”)
Nếu bạn muốn mình đưa đúng chuỗi lệnh copy/paste 100% theo Makefile của repo (install/deploy/undeploy), nói mình biết bạn đang dùng flow nào:
- A) `kubectl apply -k config/default` (deploy bằng YAML)
- B) `make deploy` / `make undeploy` (kubebuilder Makefile)

Mình sẽ bám đúng target có sẵn trong Makefile để bạn chạy 1 phát là sạch.

---
## Tóm tắt nhanh quy trình
1. Xoá namespace `image-policy-operator-system` (nếu có) +
    Xoá `ValidatingWebhookConfiguration` cũ (nếu có)
2. Đảm bảo cert-manager + CA issuer + serving cert đúng
3. Deploy operator bằng `kubectl apply -k config/default`
4. Kiểm tra 3 điểm: path đúng, endpoints có, caBundle đúng
5. Apply policy / tạo Pod test

---
 
