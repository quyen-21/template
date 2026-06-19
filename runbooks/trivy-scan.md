# Runbook: Quét lỗ hổng bảo mật ảnh container bằng Trivy

Tài liệu này hướng dẫn vận hành hệ thống quét lỗ hổng (vulnerability scanning) cho ảnh container sử dụng **Trivy** trong quy trình CI/CD và trên máy cục bộ.

---

## 1. Tổng quan hoạt động
Trivy được tích hợp trực tiếp vào GitHub Actions (`.github/workflows/build-push.yml`) nhằm kiểm tra các lỗ hổng bảo mật trước khi cho phép cập nhật ứng dụng trên môi trường Kubernetes.

* **Thời điểm chạy**: Tự động khi push mã nguồn lên nhánh `main` (có thay đổi tại thư mục `src/api/`).
* **Tiêu chí chặn**: Pipeline sẽ dừng và báo lỗi đỏ nếu tìm thấy lỗ hổng mức độ **HIGH** hoặc **CRITICAL**.
* **Cơ chế hoạt động**: Quét trực tiếp ảnh đã build trên Docker Registry (GHCR) trước khi thực hiện commit mã phiên bản mới vào kho GitOps.

---

## 2. Hướng dẫn quét thủ công tại máy cục bộ (Local Run)

Để kiểm tra lỗ hổng của ảnh container cục bộ trước khi push, chạy lệnh Trivy thông qua Docker:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:0.50.1 image <tên-ảnh-container>:<tag>
```

**Ví dụ**:
```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:0.50.1 image ghcr.io/quyen-21/w10-api:0.0.1
```

Để chỉ quét các lỗ hổng nghiêm trọng mức High/Critical và bỏ qua các lỗi chưa có bản vá từ nhà cung cấp (unfixed):
```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:0.50.1 image --severity HIGH,CRITICAL --ignore-unfixed ghcr.io/quyen-21/w10-api:0.0.1
```

---

## 3. Cách xem và khắc phục lỗ hổng
Khi bước quét Trivy trong CI/CD thất bại (màu đỏ):
1. **Xem Log**: Mở tab *Actions* trên GitHub, click vào workflow bị fail và chọn bước `Run Trivy vulnerability scanner`.
2. **Xem danh sách CVE**: Trivy sẽ hiển thị bảng chứa mã CVE, thư viện bị lỗi, phiên bản hiện tại, và phiên bản đã sửa (Fixed Version).
3. **Khắc phục**:
   * Cập nhật phiên bản của thư viện/base image bị lỗi trong `Dockerfile` hoặc file quản lý thư viện của dự án (ví dụ: `package.json`, `go.mod`, v.v.).
   * Tiến hành build và chạy thử lại pipeline.
