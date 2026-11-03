# BÁO CÁO KIỂM TRA VÀ TỐI ƯU HIỆU NĂNG PORTAL (SESSION 18 - BÀI 6)
## 1. Bối cảnh & Yêu cầu kỹ thuật

### Bối cảnh bài toán
Trang chủ `portal.greenlog.vn` tải mất tới **20 giây** mới hiện đầy đủ nội dung. Shipper phàn nàn không thấy hình ảnh bản đồ lộ trình dù kết nối mạng vẫn rất mạnh.

### Chức năng 1 (Phân tích hiệu năng)
* Mở Chrome DevTools (`F12`), chuyển sang tab **Network** và tích chọn ô **Disable cache** để phân tích thời gian nạp thực tế từ máy chủ.
* Quan sát biểu đồ **Waterfall**:
  * Các pha thời gian: **DNS Lookup**, **Initial Connection (TCP/TLS)**, **Waiting for server response (TTFB - Time to First Byte)**, và **Content Download**.
  * Phân tích thời gian nạp của từng gói tin HTML, CSS, JavaScript và Media.

### Chức năng 2 (Gỡ lỗi mạng)
* Lọc các Request bị tô đỏ với mã **Status 404 (Not Found)**: Xác định các đường dẫn hình ảnh bản đồ hoặc file cấu hình bị sai đường dẫn/mất file.
* Nhận diện các tệp hình ảnh có dung lượng quá lớn (vài chục MB ở định dạng thô PNG/RAW/BMP chưa tối ưu) làm kéo dài thanh thời gian Content Download trên Waterfall.

### Ràng buộc
Toàn bộ các tệp CSS và JavaScript quan trọng (Critical CSS/JS) phải được nạp thành công với mã **200 OK** trước khi Rendering Engine của trình duyệt thực hiện bước Parse DOM/CSSOM và tô vẽ giao diện (First Contentful Paint / Largest Contentful Paint).

---

## 2. Yêu cầu đầu ra

### Phần 1. Phân tích: Xác định "Nút thắt cổ chai" (Bottleneck) qua biểu đồ Waterfall

Khi kiểm tra trang chủ `portal.greenlog.vn` qua tab Network, hai nút thắt cổ chai chính được xác định:

1. **Nút thắt 1 - Tài nguyên Media quá nặng kéo dài dải Content Download (Màu xanh dương đậm):**
   * Xuất hiện file ảnh banner hoặc bản đồ nền có kích thước lên tới 15MB - 20MB. Dải thời gian tải nội dung (Content Download) kéo dài chiếm gần 15 - 18 giây tổng thời lượng tải trang.
2. **Nút thắt 2 - Request bị lỗi đứt link (Status 404 Not Found - Tô đỏ):**
   * File ảnh lộ trình `map-route-active.png` bị trả về mã 404 Not Found. Trình duyệt phải chờ hết thời gian phân giải/tìm kiếm tài nguyên rồi mới thất bại, gây ra hiện tượng không hiển thị bản đồ trên giao diện shipper.
3. **Nút thắt 3 - Render-blocking Resources:**
   * Các file CSS/JS chính chưa được tối ưu hoặc đặt sai vị trí chặn đứng tiến trình dựng cây DOM, khiến màn hình trắng xuất hiện trong nhiều giây đầu.

---

### Phần 2. Triển khai: Đề xuất quy trình 3 bước tối ưu toàn diện

```text
+---------------------------------------------------------------------------------+
| QUY TRÌNH 3 BƯỚC TỐI ƯU HIỆU NĂNG TẢI TRANG PORTAL GREENLOG                      |
+---------------------------------------------------------------------------------+
| Bước 1: Khắc phục lỗi đứt link 404 Not Found                                   |
|   -> Kiểm tra lại đường dẫn tương đối/tuyệt đối của file ảnh bản đồ             |
|   -> Cấu hình fallback ảnh mặc định hoặc cập nhật route tải bản đồ chính xác   |
+---------------------------------------------------------------------------------+
| Bước 2: Nén và chuyển đổi định dạng tài nguyên hình ảnh                         |
|   -> Chuyển từ PNG/JPG cồng kềnh sang định dạng thế hệ mới WebP / AVIF (giảm 70-80%)|
|   -> Sử dụng thuộc tính loading="lazy" để chỉ tải ảnh khi cuộn tới               |
+---------------------------------------------------------------------------------+
| Bước 3: Thiết lập cơ chế Browser Cache cho tài nguyên tĩnh                      |
|   -> Cấu hình Header Cache-Control: public, max-age=31536000, immutable trên NGINX |
|   -> Tận dụng ETag và cơ chế Last-Modified để phục vụ phản hồi 304 Not Modified  |
+---------------------------------------------------------------------------------+
```

#### Chi tiết cấu hình Web Server (NGINX):
```nginx
# Cấu hình nén Gzip và Cache-Control cho tài nguyên tĩnh
location ~* \.(jpg|jpeg|png|webp|avif|css|js|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, no-transform, max-age=31536000, immutable";
    access_log off;
}
```

---

### Phần 3. Kiểm thử: Tại sao mã HTTP 304 Not Modified là tín hiệu tốt cho hệ thống?

Khi người dùng tải lại trang và nhận được mã trạng thái **HTTP 304 Not Modified**:
* **Bản chất kỹ thuật:** 
  * Trình duyệt gửi request có chứa các header điều kiện: `If-None-Match` (kèm mã hash ETag của file đã lưu trước đó) hoặc `If-Modified-Since` (thời điểm tải file lần trước).
  * Web Server kiểm tra tài nguyên trên đĩa và thấy nội dung không hề thay đổi so với bản trình duyệt đang giữ.
  * Server **không gửi lại nội dung file (Response Body trống)** mà chỉ gửi về Header mã **304 Not Modified**.
* **Lý do đây là tín hiệu cực tốt cho băng thông và hiệu năng:**
  1. **Tiết kiệm tối đa băng thông (Bandwidth Saving):** Kích thước phản hồi giảm từ vài Megabyte xuống chỉ còn vài trăm byte (chỉ bao gồm HTTP headers).
  2. **Tốc độ phản hồi tức thì (Zero Download Time):** Trình duyệt lập tức lấy nội dung từ Disk Cache/Memory Cache để hiển thị, loại bỏ hoàn toàn giai đoạn Content Download.
  3. **Giảm tải cho máy chủ:** CPU và tài nguyên I/O của máy chủ không phải đọc đĩa và đẩy luồng dữ liệu lớn cho hàng ngàn shipper truy cập đồng thời.

---

## 3. Quy định nộp bài & Cấu trúc Repository

* **Tên Repository GitHub:** `IT108-K26_NhapMonCNTT_Session16_Ex06`
* **Cấu trúc thư mục:**
  ```text
  IT108-K26_NhapMonCNTT_Session16_Ex06/
  ├── README.md
  └── screenshots/
      ├── devtools_waterfall_bottleneck.png
      ├── devtools_status_404_red.png
      └── devtools_status_304_cache.png
  ```
* **Minh họa F12:** 
  * Mở DevTools (`F12`) $
ightarrow$ tab **Network**, chụp biểu đồ Waterfall hiển thị tài nguyên nạp.
  * Chụp các request 404 tô đỏ và request nhận mã 304 Not Modified.
  * Đặt repository ở chế độ **Public**, dán URL vào ô nộp bài trên LMS Portal.
