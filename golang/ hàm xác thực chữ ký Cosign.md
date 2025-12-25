 
``` bash
func VerifyImageSignature(image string) error {
 
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	img := strings.TrimSpace(image)
	if img == "" {
		return fmt.Errorf("empty image")
	}
 
	if strings.HasSuffix(img, ".sig") && strings.Contains(img, ":sha256-") {
		return fmt.Errorf("image looks like a cosign signature artifact tag (ends with .sig); verify the real image tag/digest instead, e.g. repo:tag or repo@sha256:...")
	}
	ref, err := name.ParseReference(image)
	if err != nil {
		return err
	}


	pubKeyPath := getenv("COSIGN_PUB_KEY", "./cosign.pub")
	verifier, err := signature.LoadPublicKey(ctx, pubKeyPath)
	if err != nil {
		return fmt.Errorf("load public key %q: %w", pubKeyPath, err)
	}

	co := &cosign.CheckOpts{SigVerifier: verifier}
 
	if rekorPubs, e := cosign.GetRekorPubs(ctx); e == nil {
		co.RekorPubKeys = rekorPubs
	} else {
 		if getenv("COSIGN_IGNORE_TLOG", "false") == "true" {
			co.IgnoreTlog = true
			log.Printf("warning: cannot load Rekor public keys (%v); COSIGN_IGNORE_TLOG=true so skipping tlog verification", e)
		} else {
			return fmt.Errorf("cannot load Rekor public keys (needed to verify bundle): %w (set COSIGN_IGNORE_TLOG=true to skip tlog verification)", e)
		}
	}

	_, _, err = cosign.VerifyImageSignatures(ctx, ref, co)
	if err != nil {
		return fmt.Errorf("verify failed for %q: %w", img, err)
	}
	return nil

}

```


## Tài liệu onboarding: Hàm `VerifyImageSignature(image string) error`

Tài liệu này giải thích **hàm xác thực chữ ký Cosign** cho container image. Mục tiêu là để người mới vào team đọc là hiểu ngay: hàm làm gì, cần gì để chạy, lỗi thường gặp là gì, và cách debug.

---

## 1) Hàm này dùng để làm gì?

`VerifyImageSignature(image string) error` dùng để:

- Nhận vào **container image reference** (ví dụ `repo:tag` hoặc `repo@sha256:digest`)
- Dùng **Cosign public key** (`cosign.pub`) để kiểm tra image đó **đã được ký** hay chưa
- (Tuỳ cấu hình) kiểm tra thêm **transparency log (Rekor / tlog)** để tăng độ tin cậy
- Nếu hợp lệ → trả `nil`
- Nếu không hợp lệ → trả về `error` (để webhook chặn Pod)

Nói đơn giản: **“Image chạy trong Pod phải có chữ ký đúng theo public key của công ty.”**

---

## 2) Input đúng: `image` phải có dạng nào?

Hàm mong muốn `image` là **image thật để chạy container**, ví dụ:

- Theo tag:  
  `shieldxbot/backend_example:v1.0.0`
- Theo digest (khuyến nghị nhất trong production):  
  `shieldxbot/backend_example@sha256:383e2666e9e30a8d...`

### Không được dùng `.sig`
Cosign khi ký image sẽ tạo “artifact” trong registry có tên kiểu:

- `repo:sha256-<digest>.sig`

Đây là **signature artifact**, không phải image để deploy.  
Nếu bạn đưa `.sig` vào `spec.containers[].image` hoặc đưa vào hàm verify, bạn đang verify sai đối tượng.

Hàm có đoạn chặn rõ ràng:

```go
if strings.HasSuffix(img, ".sig") && strings.Contains(img, ":sha256-") {
  return fmt.Errorf("image looks like a cosign signature artifact tag ...")
}
```

---

## 3) Tóm tắt luồng xử lý (flow) của hàm

### Bước 1: Tạo context có timeout
```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
```

Ý nghĩa:
- Tránh treo vô hạn khi registry chậm / mạng có vấn đề.
- Sau 10 giây mà chưa xong thì fail.

### Bước 2: Làm sạch input và validate
```go
img := strings.TrimSpace(image)
if img == "" { return fmt.Errorf("empty image") }
```

Ý nghĩa: input rỗng → fail ngay.

### Bước 3: Chặn case “.sig artifact”
```go
if strings.HasSuffix(img, ".sig") && strings.Contains(img, ":sha256-") {
  return fmt.Errorf("... verify the real image tag/digest ...")
}
```

Ý nghĩa:
- `.sig` là signature artifact, không phải image chạy container.

### Bước 4: Parse image reference
```go
ref, err := name.ParseReference(image)
```

Ý nghĩa:
- Validate cú pháp image reference.
- Nếu sai format (ví dụ thiếu repo, ký tự lạ) → fail.

### Bước 5: Load Cosign public key
```go
pubKeyPath := getenv("COSIGN_PUB_KEY", "./cosign.pub")
verifier, err := signature.LoadPublicKey(ctx, pubKeyPath)
```

Ý nghĩa:
- Public key là “điểm tin cậy” để verify chữ ký.
- Nếu file key không tồn tại/không đọc được → fail.

### Bước 6: Cấu hình `CheckOpts` và Rekor keys (tlog)
```go
co := &cosign.CheckOpts{SigVerifier: verifier}

if rekorPubs, e := cosign.GetRekorPubs(ctx); e == nil {
  co.RekorPubKeys = rekorPubs
} else {
  if getenv("COSIGN_IGNORE_TLOG", "false") == "true" {
    co.IgnoreTlog = true
  } else {
    return fmt.Errorf("cannot load Rekor public keys ...")
  }
}
```

Ý nghĩa:
- Cosign verify “bundle/tlog offline” cần **Rekor public keys** để trust transparency log.
- Nếu load được Rekor keys → verify đầy đủ (tốt nhất).
- Nếu không load được:
  - **Dev/debug**: cho phép `COSIGN_IGNORE_TLOG=true` để bỏ qua tlog.
  - **Production**: nên fail để tránh verify “thiếu phần trust”.

### Bước 7: Verify signatures
```go
_, _, err = cosign.VerifyImageSignatures(ctx, ref, co)
```

Ý nghĩa:
- Đây là bước “verify thật sự”.
- Nếu không có chữ ký phù hợp / chữ ký không khớp key / các kiểm tra bundle fail → trả error.

---

## 4) Environment variables (cực quan trọng)

### `COSIGN_PUB_KEY`
- Mục đích: chỉ định đường dẫn public key.
- Mặc định: `./cosign.pub`

Khuyến nghị khi chạy trong Kubernetes:
- Mount key vào 1 path cố định, ví dụ `/cosign/cosign.pub`
- Set env: `COSIGN_PUB_KEY=/cosign/cosign.pub`

### `COSIGN_IGNORE_TLOG`
- Mục đích: bỏ qua verification transparency log (tlog).
- Mặc định: `false`
- Chỉ nên bật trong **dev/debug**, không khuyến nghị production:
  - `COSIGN_IGNORE_TLOG=true`

---

## 5) Lỗi thường gặp & cách xử lý nhanh

### (A) `empty image`
- Pod spec thiếu image hoặc parse sai object.
- Fix: kiểm tra parsing PodSpec / containers.

### (B) “image looks like a cosign signature artifact tag (ends with .sig)”
- Bạn đang đưa `.sig` vào làm image.
- Fix: dùng `repo:tag` hoặc `repo@sha256:...` (image thật).

### (C) `no trusted rekor public keys provided`
- Bạn verify bằng library nhưng chưa có Rekor keys.
- Fix: đảm bảo chạy được `cosign.GetRekorPubs(ctx)` (như trong code), hoặc tạm `COSIGN_IGNORE_TLOG=true` để debug.

### (D) `no matching signatures`
- Image chưa được ký bằng public key này, hoặc bạn verify nhầm repo/tag/digest.
- Fix:
  - Kiểm tra image đã ký chưa: `cosign tree <image>`
  - Verify bằng CLI: `cosign verify --key cosign.pub <image>`

---

## 6) Cách test nhanh bằng Cosign CLI (đối chiếu với webhook)

Trên máy dev/Jenkins (có `cosign.pub`):

- Verify tag:
  - `cosign verify --key cosign.pub shieldxbot/backend_example:v1.0.0`
- Verify digest:
  - `cosign verify --key cosign.pub shieldxbot/backend_example@sha256:<digest>`

Nếu CLI PASS mà webhook FAIL:
- thường là **webhook lấy sai image string**, hoặc **key path/env khác**, hoặc **tlog keys** chưa load được.

---

## 7) Ghi chú bảo mật / best practice

- Production nên dùng **digest pinning**: `repo@sha256:...` để tránh tag bị đổi trỏ sang image khác.
- Không nên bật `COSIGN_IGNORE_TLOG=true` trong production trừ khi bạn hiểu rõ trade-off.
- Log nên in `uid/pod/image` để debug, nhưng tránh log quá nhiều dữ liệu nhạy cảm.

---
 