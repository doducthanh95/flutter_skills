---
name: authen-style-architecture-bootstrap
description: |
  Chuẩn khởi tạo project/module theo kiến trúc kiểu Authen/Core/Resource của VPBank.
  Dùng khi tạo mới feature/module iOS/SDK phải tham chiếu Core repo có sẵn, tự tạo Resource nội bộ,
  tuân thủ đúng Coordination/Coordinator hiện hữu và hỗ trợ đa ngôn ngữ qua translation layer.
version: 1.0.0
tags: [ios, authen, coordinator, resource, localization, clean-architecture, bootstrap, vpbank]
---

# Authen-Style Architecture Bootstrap

Skill này dùng để khởi tạo một project/module mới theo kiến trúc thực tế giống Authen trong repo VPBank.

Mục tiêu:
- Không tự tạo Core mới; phải tham chiếu Core repo đã tồn tại.
- Có thể tự tạo Resource riêng trong repo mới.
- Tuân thủ đúng khai báo Coordination/Coordinator hiện hữu.
- Hỗ trợ đa ngôn ngữ qua translation layer mở sẵn, không hardcode text trong UI.
- Khi có điểm chưa chắc, phải dừng và confirm với người dùng trước khi đi tiếp.

## Khi nào dùng
- Tạo mới feature/module iOS theo chuẩn Authen/Core/Resource.
- Tái cấu trúc một module cũ để khớp chuẩn kiến trúc Authen.
- Scaffold project có Coordinator + DI + UI + Localization + Resource.
- Kiểm tra xem module mới có phụ thuộc đúng vào Core repo hay không.

## Nguyên tắc không được phá vỡ

### 0) Làm tuần tự, không đoán
- Khi khởi tạo project/module mới, phải đi theo đúng thứ tự:
  1. xác nhận input
  2. xác định dependency graph
  3. scaffold structure
  4. tạo coordinator/coordination
  5. tạo domain/data/presentation
  6. tạo DI
  7. tạo localization/resource nếu cần
  8. verify build/run
- Nếu bất kỳ bước nào chưa rõ, phải dừng và confirm với người dùng trước khi đi tiếp.
- Không tự suy đoán route, naming, dependency, translation function, hay resource scope.

### 1) Không tạo Core mới
- Project mới không được scaffold lại Core.
- Core phải được tham chiếu từ repo/Core project có sẵn trước đó.
- Mọi thứ liên quan đến:
  - coordinator/navigation foundation
  - networking base
  - storage
  - event bus
  - base UI
  - translation/theme foundation
  đều phải dùng từ Core hiện hữu.

### 2) Resource có thể tự tạo trong repo
- Repo/module mới được phép tự tạo Resource riêng.
- Resource riêng phải theo convention của dự án, thường gồm:
  - assets
  - colors/themes
  - localization files
  - shared UI components
  - loading / popup / empty / retry views
  - utility helpers
- Không tự ý thay đổi cách Core/Resource giao tiếp nếu chưa xác nhận.
- Nếu chưa chắc Resource là optional hay bắt buộc cho project cụ thể, phải hỏi lại trước khi scaffold.

### 3) Coordination/Coordinator là bắt buộc
- Điều hướng phải đi qua Coordination/Coordinator contract đang có.
- Không hardcode route flow trong ViewController nếu kiến trúc hiện hữu yêu cầu Router/Coordinator.
- Tên route, package/domain, screen mapping phải nhất quán với naming convention của repo.
- Nếu chưa chắc route nào thuộc domain nào: dừng và confirm.

### 4) Đa ngôn ngữ bắt buộc qua translation layer
- Không hardcode text trực tiếp trong UI.
- Text hiển thị phải đi qua hàm mở sẵn của translation layer.
- Nếu chưa biết function/extension chính xác đang dùng trong repo, phải xác minh trước khi scaffold.

### 5) Không đoán bừa khi thiếu dữ liệu
- Nếu chưa rõ:
  - module thuộc Core hay Resource hay Feature
  - route/domain name
  - naming convention
  - dependency direction
  - translation entry point
  thì phải hỏi lại người dùng trước khi tạo file hoặc viết code.

## Quy trình chuẩn khi tạo project/module mới

### Bước 1: Xác nhận đầu vào
Cần xác định rõ:
- tên feature/module
- package/domain name
- có dùng Core repo nào
- có tự tạo Resource riêng hay dùng Resource chung
- route/coordinator domain
- translation keys cần có

Nếu chưa đủ dữ liệu, dừng và confirm.

### Bước 2: Xác minh dependency graph
Kiểm tra module mới sẽ:
- reference Core repo có sẵn
- reference hoặc tự tạo Resource riêng
- không tạo Core duplicate
- không phá vỡ workspace/package integration

### Bước 3: Scaffold structure chuẩn
Tạo structure gợi ý:

```text
FeatureName/
  Data/
    Models/
    Repositories/
  Domain/
    Entities/
    Repositories/
    UseCases/
  Presentation/
    ViewControllers/
    ViewModels/
    Cells/
    Views/
  Coordinator/
  DI/
  Resources/
    Assets/
    Localization/
    Colors/
    Fonts/
```

Ghi chú:
- Nếu repo đã dùng naming khác, ưu tiên convention hiện hữu.
- Không ép structure mới nếu nó xung đột với project gốc.

### Bước 4: Tạo Coordinator/Coordination đúng chuẩn
- Khởi tạo coordinator/router cho feature mới.
- Đăng ký route mapping đúng domain/package.
- ViewController chỉ nhận input/params theo contract đã định.
- Nếu flow có deep link hoặc open module, phải đi qua path chuẩn của Core/Coordinator.

### Bước 5: Tạo Localization đúng chuẩn
- Tạo keys cho từng ngôn ngữ.
- Map text qua translation layer mở sẵn.
- Tách rõ:
  - key
  - value theo ngôn ngữ
  - package/module scope nếu có

### Bước 6: Tạo Resource riêng nếu cần
- Thêm assets, images, colors, fonts, common views vào Resource nội bộ.
- Nếu assets cần chia sẻ cho nhiều màn hình, gom vào Resource layer thay vì nhét trong screen.

### Bước 7: Tạo DI
- Repositories nên được đăng ký theo đúng kiểu singleton/factory của dự án.
- UseCase/ViewModel/Coordinator theo lifecycle phù hợp.
- Không để ViewController tự khởi tạo object nghiệp vụ nếu project có DI.

### Bước 8: Tạo màn hình và state
- ViewController/View chỉ lo UI.
- ViewModel/Cubit/Bloc xử lý state.
- UseCase xử lý business flow.
- Repository xử lý data access.

### Bước 9: Verify
- Kiểm tra import Core đúng.
- Kiểm tra Resource riêng build được.
- Kiểm tra route registration.
- Kiểm tra translation load được.
- Kiểm tra module build/run theo đúng workspace/project.

## Checklist bắt buộc trước khi hoàn tất
- [ ] Không tạo Core mới
- [ ] Core được reference từ repo có sẵn
- [ ] Resource nội bộ được tạo đúng convention
- [ ] Coordinator/Coordination khớp chuẩn repo
- [ ] Text UI đi qua translation layer
- [ ] DI đăng ký đầy đủ
- [ ] Build verified
- [ ] Nếu có điểm chưa chắc, đã hỏi lại người dùng

## Pitfalls thường gặp

1. Tự scaffold Core mới
- Sai: tạo Core duplicate trong module mới
- Đúng: reference Core repo có sẵn

2. Hardcode text UI
- Sai: đưa string trực tiếp vào label/button
- Đúng: dùng translation function/extension

3. Route lệch domain
- Sai: tự đặt route name khác convention
- Đúng: bám đúng khai báo Coordination/Coordinator

4. Nhầm Resource chung và Resource riêng
- Sai: nhét asset chung vào screen feature
- Đúng: nếu cần, tạo Resource riêng trong repo theo convention

5. Chưa chắc nhưng vẫn làm tiếp
- Sai: đoán naming/contract
- Đúng: dừng và confirm

## Cách làm việc khi gặp thiếu thông tin
Nếu gặp bất kỳ chỗ nào sau, phải hỏi lại:
- tên route
- tên package/domain
- function translate chính xác
- project này có dùng Resource chung hay tự tạo Resource riêng hoàn toàn
- dependency Core nào cần link
- coordinator nào là entry point

## Mẫu câu hỏi confirm nên dùng
- "Anh/chị xác nhận giúp tôi domain/route name chuẩn trước khi scaffold tiếp."
- "Phần translation layer đang dùng hàm mở nào, tôi cần xác minh trước khi tạo keys."
- "Project này sẽ reference Core repo nào, tôi cần link đúng dependency trước khi tạo structure."

## Kết quả mong muốn
Sau khi chạy skill này, module mới phải có:
- structure rõ ràng
- dependency đúng hướng
- coordinator đúng contract
- localization đúng cơ chế
- resource riêng nếu cần
- không tạo Core duplicate
- build có thể verify được
