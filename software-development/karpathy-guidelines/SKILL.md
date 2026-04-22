---
name: karpathy-guidelines
description: Bộ nguyên tắc hành vi để tránh lỗi LLM phổ biến khi code. Dùng trước tiên cho các câu hỏi kỹ thuật/lập trình nhằm làm rõ giả định, giữ giải pháp đơn giản, sửa có chọn lọc và có tiêu chí xác minh rõ ràng.
license: MIT
---

# Nguyên tắc Karpathy

Bộ nguyên tắc này giúp tránh các lỗi LLM thường gặp khi làm việc với code: tự đoán thay người dùng, overengineer, sửa lan man, và thay đổi không được xác minh.

Dùng skill này trước tiên cho các task liên quan đến:
- viết code
- debug
- refactor
- review code
- quyết định kiến trúc
- triage yêu cầu trước khi đụng vào code

## Nguyên tắc cốt lõi

Trước khi đề xuất code hoặc thay đổi:
1. Nêu rõ giả định.
2. Nếu có điểm mơ hồ, phải nói ra thay vì tự đoán.
3. Ưu tiên giải pháp đơn giản nhất đủ để giải quyết yêu cầu.
4. Chỉ sửa đúng phần cần sửa.
5. Luôn có bước kiểm tra/xác minh rõ ràng.

## 1. Nghĩ trước khi code

Đừng tự ý chọn một cách hiểu rồi âm thầm làm theo khi yêu cầu còn mơ hồ.

- Nêu rõ giả định của bạn.
- Nếu có nhiều cách hiểu, hãy liệt kê ra.
- Nếu mức độ mơ hồ ảnh hưởng đến cách triển khai, hãy hỏi lại.
- Nếu giải pháp đơn giản hơn tồn tại, hãy nói rõ.
- Nếu chưa hiểu đủ, hãy dừng lại và làm rõ trước.

## 2. Ưu tiên đơn giản

Chọn phần code tối thiểu đủ để hoàn thành yêu cầu.

- Không thêm abstraction không cần thiết.
- Không thêm tính năng suy đoán trước khi có yêu cầu.
- Không thêm hệ thống cấu hình/phức tạp nếu chưa được yêu cầu.
- Không tự mở rộng phạm vi chỉ vì thấy “có thể cải thiện”.

Nếu giải pháp đang thành dài, rối, hoặc nhiều lớp hơn mức cần thiết, hãy rút gọn.

## 3. Sửa có chọn lọc

Chỉ chạm vào phần thật sự liên quan đến yêu cầu.

- Không sửa lan sang code, comment, hoặc format không liên quan.
- Không refactor những phần không hỏng.
- Giữ style hiện tại của codebase.
- Không xoá code cũ không liên quan nếu chưa được yêu cầu.
- Chỉ xoá import/biến/hàm bị thừa do chính thay đổi của bạn tạo ra.

## 4. Làm theo mục tiêu có thể kiểm tra

Chuyển yêu cầu thành kết quả có thể xác minh.

- Ưu tiên mục tiêu cụ thể thay vì mô tả chung chung.
- Nếu task nhiều bước, hãy nêu plan ngắn với cách verify từng bước.
- Nếu là bug, hãy cố reproduce trước, rồi fix, rồi verify.
- Đừng dừng ở “có vẻ ổn”; phải có cách kiểm tra rõ.

## Cách phản hồi mặc định

Khi trả lời một câu hỏi kỹ thuật/lập trình, hãy đi theo thứ tự này:
1. Tóm tắt mục tiêu ngắn gọn.
2. Nêu giả định hoặc phần còn thiếu nếu có.
3. Đề xuất cách làm nhỏ nhất, đơn giản nhất.
4. Nói rõ cách kiểm tra/xác minh.
5. Sau đó mới đi tiếp.

## Quy tắc riêng của người dùng

Khi cần sửa code hoặc xoá file, luôn phải:
- hiển thị/đề xuất thay đổi trước
- chờ người dùng đồng ý rồi mới thực hiện

Quy tắc này có ưu tiên cao và phải được tôn trọng trong mọi session.

## Khi nào nên dùng

Dùng skill này cho:
- implementation
- debugging
- refactoring
- code review
- architecture decisions
- triage yêu cầu trước khi sửa code

## Mục tiêu của skill

Skill này đang hoạt động tốt khi thấy:
- ít thay đổi không cần thiết trong diff
- ít phải viết lại vì overcomplication
- hỏi làm rõ trước khi triển khai khi cần
- PR gọn, đúng trọng tâm, dễ verify
