# Runbook: Quy trình ký ảnh container bằng Cosign

Tài liệu này hướng dẫn cách quản lý khóa và thực hiện ký số (signing) cho ảnh container sử dụng công cụ **Cosign** (Sigstore).

---

## 1. Cách sinh cặp khóa Cosign bằng Docker
Để tạo cặp khóa bí mật và công khai mới mà không cần cài đặt Cosign trực tiếp trên máy host Windows, bạn có thể chạy lệnh qua Docker:

```bash
docker run --rm -v ${PWD}:/keys -w /keys -e COSIGN_PASSWORD="<nhập_mật_khẩu_ở_đây>" ghcr.io/sigstore/cosign/cosign:v2.2.3 generate-key-pair
```

Kết quả thu được:
* `cosign.key`: Khóa bí mật (Dùng để ký ảnh). **Tuyệt đối không lưu khóa này lên Git public**.
* `cosign.pub`: Khóa công khai (Dùng để xác thực ảnh trong cụm Kubernetes). Lưu tại thư mục `signing/cosign.pub` để GitOps đồng bộ.

---

## 2. Cấu hình GitHub Secrets cho CI/CD
Để workflow tự động ký ảnh sau khi build, bạn phải đưa thông tin khóa và mật khẩu lên **GitHub Repository Secrets**:

1. Vào repository trên GitHub -> chọn **Settings** -> **Secrets and variables** -> **Actions**.
2. Thêm Secret **`COSIGN_PRIVATE_KEY`**: Sao chép toàn bộ nội dung của tệp `cosign.key` (bao gồm cả dòng đầu và dòng cuối bắt đầu bằng `-----`).
3. Thêm Secret **`COSIGN_PASSWORD`**: Mật khẩu bạn đã nhập lúc sinh khóa ở bước 1.

---

## 3. Quy trình ký ảnh thủ công (Manual Sign)
Nếu cần ký thủ công một ảnh container bằng khóa cục bộ:

```bash
# Đăng nhập vào GHCR
docker login ghcr.io -u <username>

# Thực hiện ký số
docker run --rm -v ${PWD}:/keys -w /keys -e COSIGN_PASSWORD="<mật_khẩu>" ghcr.io/sigstore/cosign/cosign:v2.2.3 sign --key cosign.key ghcr.io/<tài-khoản>/w10-api:<phiên-bản>
```

---

## 4. Xác minh chữ ký của ảnh (Verification)
Để kiểm tra xem ảnh đã được ký hợp lệ bằng khóa công khai tương ứng hay chưa:

```bash
docker run --rm -v ${PWD}:/keys -w /keys ghcr.io/sigstore/cosign/cosign:v2.2.3 verify --key cosign.pub ghcr.io/<tài-khoản>/w10-api:<phiên-bản>
```
Nếu thành công, Cosign sẽ trả về mã JSON chứa thông tin chi tiết về chữ ký và xác nhận hợp lệ.
