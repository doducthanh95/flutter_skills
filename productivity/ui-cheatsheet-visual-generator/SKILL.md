---
name: ui-cheatsheet-visual-generator
description: Tạo cheat sheet trực quan cho framework UI/language, gồm tóm tắt kiến thức, code ví dụ theo từng phần, và xuất hình ảnh/sơ đồ kèm theo để học nhanh hoặc dùng lại sau.
version: 1.0.0
tags: [cheatsheet, ui, visual, code-example, diagram, documentation]
---

# UI Cheat Sheet Visual Generator

## Mục tiêu
Tạo một cheat sheet học nhanh, dễ nhìn, gồm:
- tóm tắt kiến thức theo từng mục
- code example ngắn cho từng mục
- hình ảnh/sơ đồ tương ứng để trực quan
- output cuối cùng có thể đọc nhanh và lưu dùng lại

## Khi nào dùng
- User yêu cầu "cheat sheet"
- User muốn nội dung "trực quan nhất"
- User muốn có code ví dụ từng phần
- User muốn hình ảnh/sơ đồ minh hoạ
- User muốn lưu kết quả để dùng lại sau

## Quy trình chuẩn

### 1) Xác định phạm vi
- Nếu là framework/language, chia thành 5–8 chủ đề chính
- Nếu là UI framework, ưu tiên:
  - core concept
  - lifecycle
  - layout
  - interaction
  - resources
  - best practices

### 2) Chốt format output
Luôn tạo 2 phần:
1. Text cheat sheet ngắn, dễ copy
2. 1 ảnh cheat sheet trực quan

Nếu chủ đề lớn, có thể thêm:
- nhiều ảnh theo từng phần
- flow diagram
- sequence diagram

### 3) Chọn ví dụ code
Mỗi phần nên có:
- 1 mô tả ngắn
- 1 code snippet nhỏ, đúng ngữ cảnh
- 1 lưu ý thực chiến

Nguyên tắc code:
- ngắn
- chạy được hoặc rất gần chạy được
- tập trung vào ý chính

### 4) Tạo hình ảnh trực quan
Ưu tiên dùng:
- schematic / box diagram
- flow arrows
- mini timeline
- code blocks giả lập trong ảnh

Nếu cần hình đẹp và chuẩn hơn, có thể dùng Excalidraw.
Nếu cần xuất ảnh nhanh, có thể dùng script vẽ PNG.

### 5) Trình bày kết quả
Luôn trả về:
- đường dẫn file ảnh
- tóm tắt ngắn nội dung ảnh
- nếu phù hợp, gợi ý bản nâng cấp

## Cấu trúc cheat sheet đề xuất

### Mẫu chuẩn cho 1 framework/language
- Title
- What it is
- Core building blocks
- Typical workflow
- Code examples
- Visual diagram
- Common mistakes
- Quick checklist

### Ví dụ cho UI framework
- UI architecture
- main thread rule
- lifecycle
- layout
- input/interaction
- resources
- accessibility
- code example
- diagram

## Tiêu chuẩn chất lượng
- Đọc trong 30–60 giây là nắm được ý chính
- Mỗi phần không quá dài
- Code không được quá rối
- Hình phải có nhãn rõ ràng
- Tính thực hành cao hơn tính lý thuyết

## Pitfalls
- Không nhồi quá nhiều chữ vào một ảnh
- Không dùng font quá nhỏ
- Không tạo diagram quá chi tiết gây rối
- Không để code block quá dài
- Không chỉ tóm tắt lý thuyết mà thiếu ví dụ

## Ghi nhớ cho phản hồi
Khi user yêu cầu cheat sheet, ưu tiên:
- nội dung trực quan
- code mẫu cho từng phần
- ảnh/sơ đồ tương ứng
- kết quả dễ lưu và dùng lại
