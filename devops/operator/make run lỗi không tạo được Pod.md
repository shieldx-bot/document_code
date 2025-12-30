## Báo cáo sự cố: lỗi khi chạy `make run` + không tạo được Pod/CR

> Thời gian: 26/12/2025  
> Dự án: `image-policy-operator` (kubebuilder/operator)  
> Môi trường: cluster local (API server `127.0.0.1:41787`)

---
```bash
export CERT_DIR=/tmp/k8s-webhook-server/serving-certs
mkdir -p "$CERT_DIR"

OPENSSL_CONF=/dev/null openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout "$CERT_DIR/tls.key" \
  -out "$CERT_DIR/tls.crt" \
  -days 365 \
  -subj "/CN=webhook-service.system.svc" \
  -addext "subjectAltName=DNS:webhook-service.system.svc,DNS:webhook-service.system.svc.cluster.local,DNS:webhook-service.system,DNS:webhook-service"
```

## 1) Triệu chứng / lỗi gặp phải

### 1.1 Lỗi khi chạy `make run`
```
ERROR setup problem running manager {"error": "open /tmp/k8s-webhook-server/serving-certs/tls.crt: no such file or directory"}
make: *** [Makefile:113: run] Error 1
```

### 1.2 Không tạo được `ClusterImagePolicy`
```
strict decoding error: unknown field "spec.images[0].pattern"
```

### 1.3 Không tạo được Pod (bị chặn bởi webhook khác)
```
failed calling webhook "signed-image.security.local": ... service "signed-image-webhook" not found
```

---

## 2) Nguyên nhân gốc (Root cause)

### 2.1 Thiếu TLS cert cho webhook server khi chạy local
Sau khi scaffold webhook (Bước `kubebuilder create webhook ...`), controller-runtime **khởi động webhook server** và mặc định nó sẽ tìm chứng chỉ tại:

- tls.crt
- tls.key

Khi chạy `make run` trên máy local, **không có cert-manager** hay manifest deploy trong cluster để tự tạo/mount cert ⇒ file không tồn tại ⇒ manager fail ngay lúc startup.

**Vì sao thường không gặp khi deploy?**  
Khi deploy vào cluster (ví dụ `make deploy`), thường có cơ chế tạo cert (cert-manager hoặc job/secret) và mount vào Pod, nên webhook server có cert.

---

### 2.2 CRD schema khác với YAML bạn apply
CRD thực tế trong cluster định nghĩa:

- `spec.images[].glob` (required)

Nhưng YAML bạn apply lại dùng:

- `spec.images[].pattern`

Kubernetes bật “strict decoding” nên **reject** field không có trong schema ⇒ báo `unknown field`.

---

### 2.3 Có một ValidatingWebhookConfiguration “rác” trỏ tới Service không tồn tại
Trong cluster có:

- `ValidatingWebhookConfiguration`: `signed-image-webhook`
- webhook name: `signed-image.security.local`
- trỏ tới Service: `security/signed-image-webhook`

Nhưng Service đó **không tồn tại** ⇒ mọi request tạo Pod bị API server gọi webhook và fail ⇒ Pod không tạo được.

---

### 2.4 “make run Error 1” sau khi bạn bấm Ctrl+C là bình thường
Trong log bạn có `^C` rồi manager shutdown. Khi Ctrl+C, process exit code != 0 ⇒ `make` báo `Error 1`.  
Đây **không phải crash**, chỉ là cách `make` báo khi chương trình bị interrupt.

---

## 3) Cách đã giải quyết (What fixed it)

### 3.1 Tạo cert tạm cho webhook server (dev)
Vì OpenSSL của máy bạn bị lỗi config mặc định (`legacy_sect not found`), bạn đã tạo cert bằng cách bypass config:

- Tạo thư mục:
  - serving-certs
- Tạo `tls.crt`/`tls.key` bằng OpenSSL với `OPENSSL_CONF=/dev/null`

Kết quả: webhook server start được và log có:
- `Updated current TLS certificate ... /tmp/k8s-webhook-server/serving-certs/tls.crt`
- `Serving webhook server ... port 9443`

### 3.2 Apply CRD đúng
Bạn chạy `make install` để tạo CRD `clusterimagepolicies.security.shieldx-bot.io`.

### 3.3 Dọn webhook “signed-image” gây chặn Pod
Bạn xác định đúng resource gây lỗi là **`validatingwebhookconfiguration/signed-image-webhook`** (tên resource), không phải `signed-image.security.local` (chỉ là tên webhook entry bên trong).

Sau khi xử lý, `kubectl run nginx ...` đã thành công (exit code 0).

---

## 4) Đề xuất cách giải quyết & tránh lỗi về sau (Recommendations)

### 4.1 Với webhook TLS khi chạy local
**Khuyến nghị 1 (đơn giản nhất): tắt webhook khi dev local**
- Chỉnh code để khi `ENABLE_WEBHOOKS=false` thì **không attach WebhookServer** vào manager (không chỉ “không register handler”).
- Khi dev:
  - `ENABLE_WEBHOOKS=false make run`

**Khuyến nghị 2: tự động generate cert tạm khi run local**
- Trong main.go, nếu không truyền `--webhook-cert-path`, tự generate self-signed cert vào thư mục temp và dùng nó cho webhook server.

**Khuyến nghị 3 (production-like): deploy vào cluster**
- Dùng `make deploy` + cert-manager (nếu bật) để cert được tạo đúng cách và webhook hoạt động như môi trường thật.

---

### 4.2 Với CRD/YAML bị “unknown field”
**Nguyên tắc:** YAML phải khớp schema CRD đã apply vào cluster.

- Luôn kiểm tra `kubectl get crd <name> -o yaml` phần `openAPIV3Schema`.
- Hoặc nhìn struct Go trong `api/v1/*_types.go`, rồi chạy:
  - `make manifests`
  - `make install`

**Ví dụ YAML đúng với schema hiện tại (dùng `glob`):**
```yaml
apiVersion: security.shieldx-bot.io/v1
kind: ClusterImagePolicy
metadata:
  name: test-policy
spec:
  action: Audit
  images:
    - glob: "nginx:*"
  publicKeys: []
```

---

### 4.3 Với webhook “rác” làm hỏng cả cluster
**Khuyến nghị:**
- Khi cluster là môi trường dev dùng chung nhiều thử nghiệm, thỉnh thoảng audit:
  - `kubectl get validatingwebhookconfigurations`
  - `kubectl get mutatingwebhookconfigurations`
- Nếu thấy webhook trỏ Service không tồn tại → hoặc deploy lại service đó, hoặc xóa webhook config.

**Rule of thumb:**  
Webhook `failurePolicy: Fail` + Service chết/mất ⇒ **cluster “brick”** (tạo resource sẽ fail). Nên dev thì cân nhắc:
- `failurePolicy: Ignore` (chỉ cho môi trường dev), hoặc
- đảm bảo lifecycle deploy/uninstall dọn sạch webhook config.

---

### 4.4 Vấn đề OpenSSL “legacy_sect not found”
Đây là lỗi cấu hình OpenSSL hệ thống, có thể gây rắc rối về sau khi bạn tạo cert.

**Tránh/khắc phục:**
- Reinstall openssl + libssl + ca-certificates (tùy distro)
- Hoặc khi tạo cert dev, luôn dùng workaround:
  - `OPENSSL_CONF=/dev/null openssl ...`

---

## 5) Checklist “tránh lỗi” cho lần sau

- [ ] Dev local với webhook: có cert ở serving-certs **hoặc** tắt webhook server.
- [ ] Mỗi lần sửa `*_types.go`: chạy `make manifests` rồi `make install`.
- [ ] Apply CR/YAML đúng field theo CRD (`glob` vs `pattern`).
- [ ] Kiểm tra webhook configs trong cluster để tránh “service not found”.
- [ ] Nếu dừng `make run` bằng Ctrl+C: chấp nhận `make Error 1` là bình thường.

---
 
