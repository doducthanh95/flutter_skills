---
name: ios-foundations
description: Kiến thức nền tảng iOS gồm Swift, UIKit, và SwiftUI; dùng khi cần tóm tắt nhanh, code mẫu thực chiến, hoặc đối chiếu với codebase iOS của user.
version: 1.1.0
tags: [ios, swift, uikit, swiftui, foundation, cheat-sheet]
---

# iOS Foundations

## Mục đích
Skill này lưu các kiến thức nền tảng đã học về:
- Swift language fundamentals
- UIKit architecture and UI patterns
- SwiftUI declarative UI and data flow
- Các pattern thực chiến trong codebase iOS nội bộ: coordinator, networking, storage, event bus, resources

Dùng khi user muốn:
- cheat sheet
- giải thích nhanh concepts
- code mẫu ngắn
- đối chiếu với project iOS hiện tại
- quyết định nên dùng UIKit hay SwiftUI
- map kiến trúc SDK/module theo codebase thực tế

## 1) Swift core knowledge

### Tư duy chính
- `let` = constant, `var` = variable
- Swift ưu tiên an toàn kiểu và bất biến
- Type inference mạnh, nhưng có thể khai báo kiểu rõ ràng
- Không có implicit type conversion; phải convert tường minh
- Optional là trung tâm của Swift (`String?`, `if let`, `guard let`)
- Swift hỗ trợ:
  - functions/closures
  - tuples
  - enums
  - structs/classes
  - protocols/extensions
  - generics
  - error handling
  - concurrency (`async/await`, `Task`, actors)

### Swift code mẫu
```swift
let name = "VPBank"
var count = 1
count += 1

let width = 94
let label = "Width: \(width)"

if let value = Optional("Hello") {
    print(value)
}
```

### Optional / error / async mẫu
```swift
func fetchUser() async throws -> String {
    try await Task.sleep(nanoseconds: 100_000_000)
    return "thanh"
}

Task {
    do {
        let user = try await fetchUser()
        print(user)
    } catch {
        print(error)
    }
}
```

## 2) UIKit core knowledge

### Tư duy chính
- UIKit là framework UI imperative/event-driven cho iOS/iPadOS/tvOS
- `UIView` hiển thị nội dung
- `UIViewController` quản lý màn hình và vòng đời UI
- `UIWindow` là container gốc
- Navigation thường qua `UINavigationController` / modal presentation
- UI phải update trên main thread
- Auto Layout là chuẩn layout chính
- UIKit phù hợp với app legacy, app lớn, app có nhiều custom UI / xib / custom transition / coordinator

### UIKit lifecycle quan trọng
- `viewDidLoad()` → setup UI, bind data
- `viewWillAppear()` → chuẩn bị hiển thị
- `viewDidAppear()` → start animation / analytics
- `viewWillDisappear()` → dừng task nếu cần
- `deinit` → cleanup

### UIKit code mẫu
```swift
final class ProfileViewController: UIViewController {
    private let titleLabel: UILabel = {
        let label = UILabel()
        label.text = "Hello UIKit"
        label.font = .boldSystemFont(ofSize: 28)
        label.textAlignment = .center
        label.numberOfLines = 0
        return label
    }()

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground
        view.addSubview(titleLabel)
        titleLabel.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            titleLabel.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            titleLabel.centerYAnchor.constraint(equalTo: view.centerYAnchor)
        ])
    }
}
```

### Main-thread rule
```swift
DispatchQueue.main.async {
    self.titleLabel.text = "Updated"
}
```

### UIKit best practices
- Tách UI / business logic / navigation
- Dùng coordinator/router cho app lớn
- Dùng custom cells cho list phức tạp
- Dùng xib/storyboard chỉ khi team/project đã theo convention đó
- Luôn chú ý accessibility và localization

## 3) SwiftUI core knowledge

### Tư duy chính
- SwiftUI là declarative UI framework
- Bạn mô tả UI mong muốn, framework tự render
- View là value type và được recompute khi state thay đổi
- UI đi theo state/data, không phải imperative update thủ công
- SwiftUI rất hợp để iterate nhanh, compose view nhỏ, và xây layout hiện đại

### SwiftUI building blocks
- `View` là đơn vị cơ bản
- container views: `VStack`, `HStack`, `ZStack`, `List`, `ScrollView`
- modifiers tạo style/layout
- state-driven rendering qua `@State`, `@Binding`, `@StateObject`, `@ObservedObject`, `@Environment`
- navigation hiện đại qua `NavigationStack` + `navigationDestination`
- data modeling có thể kết hợp SwiftData

### SwiftUI code mẫu
```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack(spacing: 16) {
            Text("Count: \(count)")
                .font(.title)
            Button("+1") {
                count += 1
            }
        }
        .padding()
    }
}
```

### Layout / container mẫu
```swift
VStack(alignment: .leading, spacing: 12) {
    HStack {
        Text("Balance")
        Spacer()
        Text("VND")
    }
    Divider()
}
.padding()
.background(.white)
.clipShape(RoundedRectangle(cornerRadius: 16))
```

### Navigation mẫu
```swift
NavigationStack {
    List {
        NavigationLink("Profile", value: "profile")
    }
    .navigationDestination(for: String.self) { route in
        if route == "profile" {
            Text("Profile")
        }
    }
}
```

### Async/state update mẫu
```swift
struct LoadView: View {
    @State private var loading = false
    @State private var message = ""

    var body: some View {
        Button("Load") {
            Task { @MainActor in
                loading = true
                defer { loading = false }
                message = await fetchMessage()
            }
        }
    }
}
```

## 4) UIKit vs SwiftUI: khi nào dùng

### Ưu tiên UIKit khi:
- project legacy đã dùng UIKit nhiều
- cần control layout / gesture / animation / custom transitions mạnh
- đang theo coordinator + xib + MVC/MVVM cũ
- cần tương thích với module/native SDK sẵn có

### Ưu tiên SwiftUI khi:
- build màn hình mới nhanh
- UI đơn giản đến trung bình
- muốn state-driven UI
- cần preview và iteration nhanh
- app mới hoặc kiến trúc mới theo declarative style

### Kết hợp cả hai
- Có thể nhúng UIKit trong SwiftUI hoặc ngược lại
- Thực tế thường dùng hybrid migration

## 5) Kiến thức áp dụng vào project iOS của user

Trong codebase iOS Account / native module của user, các pattern sau rất quan trọng:
- `UIViewController` cho màn hình
- `ViewModel` cho state/data logic
- `Coordinator` cho routing
- `Combine` hoặc async/await cho binding/flow
- custom cells / xib cho list
- QR rendering / custom drawing
- localization En/Vi
- resource management qua assets / strings / fonts

### Quy tắc khi đọc/sửa code
- Nếu là UI screen: tìm ViewController
- Nếu là state: tìm ViewModel / ObservableObject / @Published
- Nếu là điều hướng: tìm Coordinator / Router
- Nếu là list UI: tìm custom cell / table / collection view datasource
- Nếu là dữ liệu: tìm model / entity / use case / service

## 6) Common pitfalls
- Quên main-thread khi update UI
- Swift optionals chưa unwrap an toàn
- ViewController ôm quá nhiều logic
- SwiftUI state đặt sai chỗ dẫn đến render lại không đúng
- Navigation trong SwiftUI bị lẫn imperative style cũ
- Dùng frame thay vì Auto Layout trong UIKit khi không cần thiết
- Quên accessibility/localization

## 7) Quick recall checklist
- Swift: let/var, optional, protocols, generics, async/await
- UIKit: UIView, UIViewController, Auto Layout, lifecycle, main thread
- SwiftUI: declarative, state-driven, containers, navigation, data flow
- Hybrid iOS app: chọn framework theo màn hình + mức độ control + kiến trúc hiện tại

## 8) Learned from the current iOS codebase

### Core SDK pattern
- `CoreSDK` là singleton entrypoint của SDK.
- `register(isShowLog:partner:)` set partner và init state mặc định như dark mode.
- `CoreSDK.coordinator(packageName:)` parse package name rồi lấy coordinator từ `CoordinatorController`.

### Coordinator / navigation
- `CoreCoordinator` khai báo `packageName`, `routers`, `getViewController(...)`.
- `CoordinatorController` giữ danh sách coordinator và một `UINavigationController` global.
- Các helper navigation quan trọng:
  - `pushNamed`
  - `pushReplacementNamed`
  - `pop`
  - `popUntil`
  - `popToRoot`
- Khi pop ra khỏi module, SDK có thể bắn `CloseSdkEvent` qua `EventBus`.

### Base UI layer
- `BaseViewController` hỗ trợ 3 kiểu load:
  - xib
  - storyboard
  - programmatic
- Base VC có `params`, `packageName`, `setupUI()`, `bindViewModel()`.
- Dark mode được đọc từ `StorageClient` và có thể ép `.light` nếu module không support.
- `BaseViewModel` dùng Combine `PassthroughSubject` và các hook lifecycle: ready/active/inactive/release.

### Networking layer
- `NetworkClient` là URLSession-based client, không phải chỉ wrapper mỏng.
- Có interceptor chain với các hook:
  - `adapt`
  - `process`
  - `onError`
  - `shouldRetry`
  - `didReceive`
- `TokenInterceptor` tự gắn Bearer token và queue request khi refresh token.
- `CertificatePinningInterceptor` support pinning theo certificate hoặc public key.
- Response/error models tách rõ:
  - `BaseRequestModel`
  - `BaseResponseModel<T>`
  - `BaseResultModel`
  - `BaseErrorModel<E>`

### Storage layer
- `StorageClient` là facade, gộp 3 backend:
  - `KeychainStorage`
  - `UserDefaultsStorage`
  - `MemoryStorage`
- `StorageKey` chuẩn hóa key theo token/theme/language/userName/type.
- Dùng `Keychain` cho dữ liệu nhạy cảm, `UserDefaults` cho config, `Memory` cho state tạm.

### EventBus / deeplink / translate / theme
- `EventBus` dùng Combine `PassthroughSubject<OneEvent, Never>`.
- Event tiêu biểu:
  - `LogoutEvent`
  - `TokenExpiredEvent`
  - `DeepLinkEvent`
  - `ChangeLanguageEvent`
  - `CloseSdkEvent`
- `DeepLink` tạo và parse link kiểu `Vpbank://{partner}?domain=...&version=...&destination=...`.
- `ThemeController` registry theme theo `packageName + type`.
- `TranslateController` registry translate theo `domain + language` và bắn event khi đổi ngôn ngữ.

### Resources module
- `resources/Resoures/Resoures` là UI kit + asset kit dùng chung.
- `ResourceAssets` load image qua bundle nội bộ và expose shortcut `UIImage` helpers.
- Nhóm quan trọng:
  - `DesignSystems` cho popup/calendar/message/no-data/retry/skeleton
  - `Share` cho loading/OTP/common views/events
  - `Utils` cho money reader/date formatter/dropdown/extensions
  - `LocalPodLibs/Lottie` là engine animation source nội bộ

### Kinh nghiệm áp dụng vào dự án iOS
- Nếu sửa màn hình: tìm `BaseViewController` hoặc subclass của nó.
- Nếu sửa điều hướng: tìm `CoordinatorController` / router dictionary.
- Nếu sửa API: tìm `NetworkClient` + interceptor + request/response models.
- Nếu sửa lưu trạng thái: ưu tiên `StorageClient` thay vì gọi thẳng Keychain/UserDefaults.
- Nếu sửa cross-module communication: dùng `EventBus`.
- Nếu thêm UI dùng chung: đặt vào `resources` để reuse theo asset/component.
