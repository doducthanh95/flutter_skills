---
name: figma-upload-automation
description: |
  Quy trình đã thử để tự động tạo/upload mockup vào Figma bằng Figma REST API,
  những lỗi gặp phải (404, 403 khi gọi endpoint images), và các giải pháp thay thế
  (upload qua storage + dùng imageRef, hoặc Figma Plugin để import SVG như nodes).
version: 0.1.1
tags: [figma, design, automation, api]
---

Mục đích
- Ghi lại các bước thực tế đã thử khi tự động đưa assets (SVG/PNG) vào file Figma
  thông qua REST API, cùng các lỗi, nguyên nhân khả dĩ và giải pháp thay thế.

Khi nào dùng
- Khi cần tích hợp pipeline CI/CD để publish design assets lên Figma.
- Khi muốn tự động hoá import SVGs thành nodes trong file Figma (trên môi trường
  mà REST API hỗ trợ upload trực tiếp).

Tổng quan các bước đã thử (kịch bản thực tế)
1) Xác thực token
   - Lệnh mẫu:
     curl -s -H "X-FIGMA-TOKEN: <token>" https://api.figma.com/v1/me
   - Kết quả mong đợi: trả về {id, email, handle, img_url} → token hợp lệ.

2) Kiểm tra file / quyền
   - Lệnh mẫu:
     curl -s -H "X-FIGMA-TOKEN: <token>" https://api.figma.com/v1/files/:file_key
   - Kết quả thực tế trong thử nghiệm: GET /v1/files/:key trả về JSON file (lastModified, components, styles). Kiểm tra trường "role" (owner/editor/reader) rất quan trọng.

3) Tạo assets local
   - Lưu màn hình mockup thành SVG/PNG.
   - Ví dụ: luong_thanh_toan_dien_screen_1_customer_id.svg ... _4_success.svg
   - Đóng gói thành ZIP khi cần chuyển cho người dùng.

4) Thử upload images qua REST API
   - Các endpoint đã thử (nhiều biến thể):
     * POST /v1/files/:file_key/images (multipart/form-data)
     * POST /v1/images
     * POST /v1/files  (tạo file mới)
   - Các biến thể field-name đã thử: images[], images[0], images[], image=@..., images=@...
   - Lệnh mẫu:
     curl -s -X POST "https://api.figma.com/v1/files/:file_key/images" \
       -H "X-FIGMA-TOKEN: <token>" \
       -F "images[]=@screen1.svg" -F "images[]=@screen2.svg"

   - Kết quả thực tế: mọi lần POST lên các endpoint trên đều trả 404 {"status":404,"err":"Not found"} hoặc {"status":404,"error":true,"message":"not found"} trong ngữ cảnh thử với Personal Access Token và file key cụ thể. Trong khi đó GET /v1/me và GET /v1/files/:key vẫn thành công.

   - Ghi chú: lỗi 404 chỉ xảy ra với các thao tác POST (tạo file/upload image); điều này cho thấy giới hạn/quyền trên endpoint hoặc thay đổi API chứ không phải token hoàn toàn vô hiệu.

5) Các bước troubleshooting đã làm
   - Kiểm tra token (GET /me) → OK
   - Kiểm tra file existence & role (GET /files/:key) → OK
   - Thử nhiều biến thể multipart/form-data key names
   - Thử tạo file mới bằng POST /v1/files (nhận 404)
   - Thử gọi /v1/files/ (list) => 404

Nguyên nhân khả dĩ
- Token (PAT) hợp lệ nhưng có thể không có quyền write cho endpoint nhất định (ví dụ token tạo chỉ dùng cho đọc hoặc file thuộc project có policy giới hạn API uploads).
- Một số endpoint của Figma REST có thể không cho phép tạo file/upload trong ngữ cảnh account/token hiện tại.
- Endpoint hoặc tham số multipart có thể đã thay đổi — cần tham khảo docs mới nhất.
- File có thể thuộc Team/Org có cấu hình chặn API ghi.

Giải pháp thay thế (recommended)
- Dùng Figma Plugin: plugin chạy trong context Figma có quyền tạo nodes, import SVG như vector và gán thành các Frame/Component. Đây là cách ổn định nhất để tự động "chèn" SVG vào file thông qua code.
- Dùng hosted images: upload assets lên S3 hoặc storage tạm thời, rồi sử dụng API nếu hỗ trợ imageRef/URL để tham chiếu (tùy endpoint Figma có hỗ trợ hay không).
- Nếu automation trên client-side được chấp nhận: tạo script AppleScript/Automator/cliclick trên máy người dùng để mở Figma desktop (hoặc app khác như Pencil) và import file tự động.
- Nếu không thể tự động, export ZIP và cung cấp cho designer import thủ công.

Snippet thực tế đã dùng (đã thử và ghi nhận lỗi)
- Kiểm tra token:
  curl -s -H "X-FIGMA-TOKEN: <token>" https://api.figma.com/v1/me

- Đọc file:
  curl -s -H "X-FIGMA-TOKEN: <token>" https://api.figma.com/v1/files/VXrseadPbyTwF4rasnpPKj

- Thử upload images (ví dụ đã nhận 404):
  curl -s -X POST "https://api.figma.com/v1/files/VXrseadPbyTwF4rasnpPKj/images" \
    -H "X-FIGMA-TOKEN: <token>" \
    -F "images[]=@luong_thanh_toan_dien_screen_1_customer_id.svg" \
    -F "images[]=@luong_thanh_toan_dien_screen_2_bill.svg"

- Thử tạo file mới (nhận 404):
  curl -s -X POST "https://api.figma.com/v1/files" -H "X-FIGMA-TOKEN: <token>" -H "Content-Type: application/json" -d '{"name":"Luong Thanh Toan Dien - Hermes"}'

Ghi chú bảo mật
- Không lưu token trong skill. Các lệnh chỉ là ví dụ mẫu.

Recommendations để cải thiện flow tự động
- Nếu mục tiêu là đưa SVG vào file Figma programmatically cho production pipeline:
  1) Xây 1 Figma Plugin (JS) chạy bằng token người dùng trong context file hoặc được cài trong team — plugin có thể fetch SVG từ URL hoặc lấy payload và tạo các vector nodes/frames.
  2) Hoặc host các assets (S3) và sử dụng API theo docs nếu Figma có endpoint hỗ trợ imageRef -> node creation.
  3) Tự động hoá client-side: cung cấp AppleScript/Automator script để import file vào Figma Desktop (phù hợp cho workflow cá nhân/designer).

Các bài học từ thử nghiệm
- Luôn verify token bằng GET /v1/me rồi kiểm tra role trên file trước khi cố gắng upload.
- Nếu POST upload luôn trả 404 nhưng GET file OK, nghĩ tới policy/project-level restrictions hoặc endpoint không khả dụng cho token loại đó.
- Biện pháp bền vững: plugin > hosted images + reference > REST direct upload.

Kết luận
- Lưu các mẫu lệnh, lỗi, và giải pháp thay thế vào skill này để tái sử dụng khi tự động hoá upload assets vào Figma.
- Nếu cần, tiếp theo có thể thêm ví dụ code cho 1 Figma Plugin (JS) import SVG từ URL, hoặc AppleScript mẫu để import SVG vào Figma Desktop.
