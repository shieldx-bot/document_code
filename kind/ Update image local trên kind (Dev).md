## Note nhanh: Update image local trên **kind** (Dev) ✅

Mục tiêu: Bạn **docker build** lại image `:latest` trên máy host, rồi muốn Pod trong **kind** dùng **image mới** ngay.

Vì **kind chạy node trong container**, image build ở host **không tự có** trong node kind → phải “load” vào kind và **restart Deployment** để tạo Pod mới.

---

## Công thức chuẩn (Cách A)

### Bước 1 — Build image ở máy host
Bạn build như bình thường, ví dụ:

- `signed-images-admission-webhook:latest`

> Kết quả mong muốn: `docker images` thấy image mới (ID mới).

---

### Bước 2 — Load image vào cluster kind
Load image vào các node của cluster kind:

- `kind load docker-image signed-images-admission-webhook:latest --name my-sre-cluster`

> Ý nghĩa: copy image từ Docker host → vào node containerd của kind.

---

### Bước 3 — Restart Deployment để Pod pull “image mới”
Buộc Deployment tạo Pod mới:

- `kubectl rollout restart deploy/signed-image-webhook -n security`

> Lưu ý: nếu YAML “unchanged” thì Kubernetes sẽ không tự restart, nên cần bước này.

---

### Bước 4 — Kiểm tra Pod mới đã lên chưa
Xem Pod/Node/IP:

- `kubectl get pods -n security -l app=signed-image-webhook -o wide`

> Nếu muốn chắc hơn, xem ImageID đang chạy:
- `kubectl describe pod -n security -l app=signed-image-webhook | grep -E 'Image:|Image ID:' -n`

---

## Checklist nhanh (để tự debug)
- `docker images | grep signed-images-admission-webhook` → có image mới chưa?
- `kind get clusters` → đúng tên `my-sre-cluster` chưa?
- `kind load docker-image ...` → load thành công chưa?
- `kubectl rollout restart ...` → restart có chạy chưa?
- `kubectl get pods ...` → Pod mới có AGE nhỏ (vừa tạo) chưa?

---
 