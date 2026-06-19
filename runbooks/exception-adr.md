# ADR: Quy trình ghi nhận ngoại lệ cho các lỗ hổng chưa có bản vá (Exception ADR)

* **Trạng thái**: Đã phê duyệt (Approved)
* **Ngày quyết định**: 2026-06-19

---

## 1. Bối cảnh (Context)
Hệ thống CI/CD được tích hợp công cụ quét lỗ hổng **Trivy** với mức chặn nghiêm ngặt (chặn toàn bộ image chứa CVE mức `HIGH` hoặc `CRITICAL`). 

Tuy nhiên, trong thực tế có những trường hợp:
1. Phát hiện lỗ hổng nhưng từ phía nhà phát hành thư viện gốc (vendor) **chưa tung ra bản vá (no fixed version available)**.
2. Ứng dụng không thực sự sử dụng tính năng/lớp code chứa lỗ hổng đó (rủi ro thấp).
3. Cần triển khai khẩn cấp và không thể chờ đợi bản cập nhật từ bên thứ ba.

Nếu tiếp tục chặn cứng pipeline sẽ gây tắc nghẽn quá trình phát triển phần mềm (block deployments).

---

## 2. Quyết định (Decision)
Chúng ta đồng ý cho phép sử dụng cơ chế **Ngoại lệ tạm thời (Temporary Exception)** để bỏ qua một số mã CVE cụ thể khi chạy Trivy quét ảnh, tuân thủ các quy tắc sau:

1. **Cơ chế kỹ thuật**: Sử dụng tệp tin `.trivyignore` được đặt ở thư mục gốc của repo hoặc thư mục dự án `src/api/`.
2. **Khai báo thông tin**: Mỗi CVE được loại trừ trong `.trivyignore` bắt buộc phải kèm theo:
   * Mã CVE cụ thể.
   * Lý do loại trừ (ví dụ: `vendor chưa có bản vá`, `không sử dụng module lỗi`, v.v.).
   * Thời hạn hiệu lực của ngoại lệ (thường tối đa là **30 ngày**).
3. **Quy trình phê duyệt**: Phải có sự đồng thuận từ bộ phận Security/Tech Lead trước khi đưa mã CVE vào danh sách bỏ qua.

---

## 3. Hướng dẫn thực hành (How-to)

### Bước 1: Tạo tệp cấu hình `.trivyignore`
Tạo file `.trivyignore` ở thư mục chứa mã nguồn cần quét (ví dụ: `src/api/.trivyignore`).

### Bước 2: Khai báo mã CVE cần bỏ qua
```text
# Bỏ qua lỗ hổng CVE-2024-XXXX do vendor chưa có bản vá. Hết hạn: 2026-07-19
CVE-2024-XXXX

# Bỏ qua lỗ hổng CVE-2023-YYYY do thư viện không sử dụng thực tế. Hết hạn: 2026-07-19
CVE-2023-YYYY
```

### Bước 3: Cấu hình workflow nhận file ignore
Khi chạy lệnh Trivy trong workflow, Trivy mặc định sẽ tự động tìm kiếm file `.trivyignore` ở thư mục quét. Để đảm bảo chắc chắn, có thể chỉ định tham số:
```yaml
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@v0.35.0
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.semver.outputs.version }}
          trivyignore: ./src/api/.trivyignore
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
```
*(Lưu ý: Tệp `.trivyignore` phải được commit lên Git cùng đợt code để quy trình CI nhận dạng).*
