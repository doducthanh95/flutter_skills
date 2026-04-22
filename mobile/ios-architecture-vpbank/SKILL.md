---
name: ios-architecture-vpbank
description: |
  Chuẩn kiến trúc iOS theo style VPBank/Authen để khởi tạo module/project mới.
  Bắt buộc tuân thủ Coordination/Coordinator hiện hữu, Resource là optional, repository implementation
  phải gọi NetworkUtils.shared.api.request, và mọi dependency dùng trong Data/Domain/ViewModel phải được
  inject qua constructor rồi register trong DI.
version: 1.0.0
tags: [ios, vpbank, authen, coordinator, resource, localization, clean-architecture, dependency-injection]
---

# iOS Architecture VPBank

Skill này dùng để khởi tạo hoặc refactor project/module iOS theo kiến trúc thực tế của VPBank/Authen.

## Mục tiêu
- Tuân thủ đúng Coordination/Coordinator hiện hữu.
- Resource chỉ là optional, không bắt buộc.
- Repository implementation bắt buộc gọi `NetworkUtils.shared.api.request`.
- Không gọi trực tiếp singleton class từ Data/Domain/ViewModel.
- Nếu cần dùng một service/manager/class nào đó, phải inject qua constructor và register trong DI.
- Hỗ trợ đa ngôn ngữ qua translation layer, không hardcode text UI.
- Nếu chưa chắc về contract/naming/dependency thì phải dừng và confirm với người dùng.

## Khi nào dùng
- Tạo module iOS mới theo chuẩn VPBank/Authen.
- Refactor module cũ để bám theo pattern hiện hữu.
- Tạo screen/flow mới có Coordinator + DI + Data/Domain/ViewModel.
- Kiểm tra dependency injection, networking, localization, routing trước khi scaffold.

## Rule cứng

### 1) Resource là optional
- Repo/module mới có thể có hoặc không có `Resources/`.
- Chỉ tạo Resource khi thực sự cần:
  - assets
  - localization
  - colors/fonts
  - shared UI components
  - loading / popup / empty / retry views
- Nếu project không cần Resource riêng, không bắt buộc scaffold.

### 2) Không dùng Core reference bắt buộc
- Skill này không ép project phải tự tạo Core mới.
- Không tự scaffold Core duplicate trong repo mới.
- Nếu project cần các foundation có sẵn từ Core hoặc module nền khác, phải link đúng dependency hiện hữu của workspace/repo.
- Nếu chưa rõ dependency nào là chuẩn, phải hỏi lại trước khi tạo file.

### 3) Repository implementation bắt buộc gọi network qua NetworkUtils.shared.api.request
- Tất cả repository implementation phải đi qua API request layer chuẩn.
- Không tự gọi API theo kiểu khác nếu repository của project đã chuẩn hóa qua `NetworkUtils.shared.api.request`.
- Data layer là nơi duy nhất xử lý API orchestration, mapping DTO/response, error translation.

Ví dụ pattern:
```swift
final class FeatureRepositoryImpl: FeatureRepository {
    private let networkUtils: NetworkUtils

    init(networkUtils: NetworkUtils) {
        self.networkUtils = networkUtils
    }

    func fetchFeature(request: BaseRequestModel) async throws -> FeatureModel {
        let response = try await networkUtils.api.request(request)
        return try map(response)
    }
}
```

### 4) Không gọi trực tiếp singleton class từ Data/Domain/ViewModel
- Không được viết kiểu:
  - `SomeManager.shared.doSomething()`
  - `SomeService.instance.fetch()`
  - `NetworkUtils.shared...` trực tiếp trong ViewModel/Domain/Data nếu nó không đi qua constructor injection của module
- Mọi dependency phải đi qua constructor injection.
- Object nào cần dùng thì phải tạo ở DI, sau đó truyền xuống:
  - Coordinator
  - ViewModel
  - UseCase
  - Repository implementation

### 5) Dependency phải đăng ký trong DI
- Nếu một class/service/manager cần dùng ở Data/Domain/ViewModel, phải register trong DI.
- ViewController/ViewModel không được tự khởi tạo singleton “ngầm” trong thân hàm nghiệp vụ.
- DI là composition root của module.

### 6) Coordination/Coordinator là bắt buộc
- Điều hướng phải đi qua route/coordinator contract đang có.
- Không hardcode navigation logic rải rác trong screen.
- Route name, package name, domain name phải khớp convention của repo.
- Nếu chưa chắc, phải confirm.

### 7) Localization bắt buộc qua translation layer
- Không hardcode text trong UI.
- Tất cả text phải gọi qua hàm mở trong translation layer.
- Nếu không biết extension/function chính xác, phải xác minh trước.

### 8) Không đoán bừa khi thiếu dữ liệu
Nếu chưa rõ:
- route name
- package/domain name
- translation function
- dependency graph
- resource có cần hay không
- class nào phải inject từ DI
thì phải dừng và hỏi người dùng.

## Quy trình chuẩn khi tạo module/project mới

### Bước 1: Xác nhận đầu vào
Cần biết rõ:
- feature/module name
- route/domain name
- module có cần Resource riêng hay không
- dependency nào được inject
- repository sẽ gọi API nào
- translation keys cần tạo

### Bước 2: Xác định dependency graph
- Liệt kê rõ class nào được tạo mới.
- Liệt kê rõ class/service nào phải inject.
- Xác định repository implementation gọi network service nào.
- Xác định coordinator entry point.

### Bước 3: Scaffold structure chuẩn
Template gợi ý:

```text
FeatureName/
├── Coordinator/
│   └── FeatureCoordinator.swift
├── DI/
│   └── FeatureDI.swift
├── Domain/
│   ├── Entities/
│   │   └── FeatureEntity.swift
│   ├── Repositories/
│   │   └── FeatureRepository.swift
│   └── UseCases/
│       └── FeatureUseCase.swift
├── Data/
│   ├── Models/
│   │   └── FeatureModel.swift
│   └── Repositories/
│       └── FeatureRepositoryImpl.swift
├── Presentation/
│   ├── ViewControllers/
│   │   └── FeatureViewController.swift
│   ├── ViewModels/
│   │   └── FeatureViewModel.swift
│   ├── Cells/
│   └── Views/
└── Resources/               # optional
    ├── Assets/
    ├── Localization/
    ├── Colors/
    ├── Fonts/
    └── Views/
```

### Bước 4: Tạo Coordinator
- Khởi tạo route mapping theo contract hiện hữu.
- Coordinator chỉ lo điều hướng.
- Nếu module có deep link/open-module flow, phải đi qua đường dẫn chuẩn.

### Bước 5: Tạo Data/Domain/Presentation
- Domain:
  - Entity
  - Repository protocol
  - UseCase
- Data:
  - Model
  - Repository implementation
- Presentation:
  - ViewModel state
  - ViewController/UI binding

### Bước 6: Tạo DI
- Register repository implementation.
- Register usecase/viewmodel.
- Register dependencies cần inject.
- Không để ViewController tự lấy singleton nghiệp vụ.

### Bước 7: Tạo Localization nếu cần
- Tạo key/value theo từng ngôn ngữ.
- Text UI phải lấy từ translation layer.

### Bước 8: Resource optional
- Nếu module cần asset/shared UI, thêm Resource riêng.
- Nếu không cần, không ép tạo.

### Bước 9: Verify
- Kiểm tra import/dependency đúng.
- Kiểm tra repository dùng `NetworkUtils.shared.api.request`.
- Kiểm tra DI có register mọi dependency.
- Kiểm tra coordinator/route mapping.
- Kiểm tra localization load được.
- Kiểm tra build/run nếu workspace cho phép.

## Template class examples

### Coordinator example
```swift
import UIKit

final class FeatureCoordinator: CoreCoordinator {
    let packageName: String = "feature"

    var routers: [String: RouteHandler] = [
        FeatureRoute.home.rawValue: { params in
            FeatureViewController.create(params: params, packageName: "feature")
        }
    ]
}
```

### DI example
```swift
final class FeatureDI {
    static let shared = FeatureDI()

    private init() {}

    func register(
        networkUtils: NetworkUtils,
        analytics: AnalyticsService
    ) {
        // register repository
        // register usecase
        // register viewmodel
        // inject networkUtils, analytics via constructor
    }
}
```

### Domain entity example
```swift
struct FeatureEntity {
    let id: String
    let title: String
}
```

### Repository protocol example
```swift
protocol FeatureRepository {
    func fetchFeature() async throws -> FeatureEntity
}
```

### UseCase example
```swift
final class FeatureUseCase {
    private let repository: FeatureRepository

    init(repository: FeatureRepository) {
        self.repository = repository
    }

    func execute() async throws -> FeatureEntity {
        try await repository.fetchFeature()
    }
}
```

### Repository implementation example
```swift
final class FeatureRepositoryImpl: FeatureRepository {
    private let networkUtils: NetworkUtils

    init(networkUtils: NetworkUtils) {
        self.networkUtils = networkUtils
    }

    func fetchFeature() async throws -> FeatureEntity {
        let request = BaseRequestModel()
        let response = try await networkUtils.api.request(request)
        return try map(response)
    }
}
```

### ViewModel example
```swift
final class FeatureViewModel {
    enum State {
        case idle
        case loading
        case loaded(FeatureEntity)
        case failed(String)
    }

    private let useCase: FeatureUseCase
    var onStateChanged: ((State) -> Void)?

    init(useCase: FeatureUseCase) {
        self.useCase = useCase
    }

    func load() {
        onStateChanged?(.loading)
    }
}
```

### ViewController example
```swift
import UIKit

final class FeatureViewController: BaseViewController {
    private let viewModel: FeatureViewModel

    init(viewModel: FeatureViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        bindViewModel()
        viewModel.load()
    }

    private func bindViewModel() {
        viewModel.onStateChanged = { state in
            // update UI
        }
    }
}
```

## Common pitfalls
- Tự dùng singleton trong ViewModel/Domain/Data thay vì inject qua constructor.
- Repository implementation không đi qua `NetworkUtils.shared.api.request`.
- Hardcode text UI.
- Route/coordinator không theo contract hiện hữu.
- Tạo Resource dù project không cần.
- Chưa rõ dependency nhưng vẫn scaffold tiếp.

## Checklist
- [ ] Coordinator đúng contract
- [ ] DI register tất cả dependency
- [ ] Data/Domain/ViewModel không tự gọi singleton trực tiếp
- [ ] Repository implementation gọi `NetworkUtils.shared.api.request`
- [ ] Resource chỉ tạo khi cần
- [ ] Localization qua translation layer
- [ ] Không hardcode text
- [ ] Verify build/run nếu workspace cho phép

## Khi nào phải dừng và confirm
- Chưa rõ function translation
- Chưa rõ route/domain name
- Chưa rõ dependency cần inject
- Chưa rõ repository API contract
- Chưa rõ Resource có cần hay không
- Chưa chắc coordinator entry point
