---
name: Flutter Clean Architecture
description: |
  Scaffold Flutter packages/features theo Clean Architecture + BLoC pattern.
  Dựa trên cấu trúc thực tế từ dự án, sử dụng Core SDK với BaseWidget, BaseCubit,
  DioUtils, GetIt DI, và OneRouter. Hỗ trợ auto-generate code bằng script init_feature.sh.
version: 1.0.0
tags: [flutter, clean-architecture, bloc, dependency-injection, routing]
---

# Flutter Clean Architecture Skill

## Mục đích
- Tạo feature/package Flutter theo Clean Architecture
- Tích hợp với Core SDK (BaseWidget, BaseCubit, DioUtils, OneRouter)
- Dependency Injection với GetIt
- State management với BLoC/Cubit
- Code generation với json_serializable và build_runner

## Khi nào dùng
- Tạo feature mới trong dự án Flutter modular
- Cần cấu trúc Clean Architecture chuẩn
- Dự án có Core SDK với BaseWidget, routing system
- Cần tích hợp API, state management, navigation

## Cấu trúc thư mục chuẩn

```
lib/
├── feature/
│   └── <feature_name>/
│       ├── data/
│       │   ├── models/              # Data models với json_serializable
│       │   │   ├── <feature>_model.dart
│       │   │   ├── <feature>_model.g.dart
│       │   │   └── sample_model.dart
│       │   ├── repositories_impl/   # Repository implementations
│       │   │   └── <feature>_repository_impl.dart
│       │   ├── helpers/             # Request helpers
│       │   │   └── <feature>_request_helper.dart
│       │   ├── mock_json/           # Mock data
│       │   │   └── <feature>_sample.json
│       │   └── endpoint.dart        # API endpoints (nếu có nhiều)
│       ├── domain/
│       │   ├── entities/            # Business entities
│       │   │   └── sample_entity.dart
│       │   ├── repositories/        # Repository interfaces
│       │   │   └── <feature>_repository.dart
│       │   └── usecases/            # Business logic
│       │       └── <feature>_usecase.dart
│       ├── presentation/
│       │   ├── cubit/               # State management
│       │   │   └── <feature>_cubit.dart
│       │   ├── ui/                  # UI components
│       │   │   ├── <feature>_page.dart
│       │   │   └── widgets/
│       │   └── bloc_observer.dart
│       ├── event/                   # Custom events
│       │   └── <feature>_event.dart
│       ├── helpers/                 # Helper functions
│       │   └── helpers.dart
│       ├── di.dart                  # Dependency injection
│       └── router_composer.dart     # Router registration
├── router.dart                      # Route definitions
├── sdk.dart                         # SDK initialization
├── translate/                       # i18n
│   ├── vi.dart
│   ├── en.dart
│   ├── ja.dart
│   ├── ko.dart
│   └── zh.dart
├── theme/
│   └── theme.dart
└── pubspec.yaml
```

## Các layer chính

### 1. Data Layer
**Models** - JSON serializable data models
```dart
import 'package:json_annotation/json_annotation.dart';

part '<feature>_model.g.dart';

@JsonSerializable()
class FeatureModel {
  final String data;
  final String? optionalField;
  
  FeatureModel({required this.data, this.optionalField});
  
  factory FeatureModel.fromJson(Map<String, dynamic> json) => 
      _$FeatureModelFromJson(json);
  Map<String, dynamic> toJson() => _$FeatureModelToJson(this);
}
```

**Repository Implementation** - Gọi API qua DioUtils
```dart
import 'package:core/core.dart';
import '../models/<feature>_model.dart';
import '../../domain/repositories/<feature>_repository.dart';

class FeatureRepositoryImpl implements FeatureRepository {
  @override
  Future<BaseResponseModel<FeatureModel>> fetchData({
    required BaseRequestModel model,
  }) async {
    final response = await DioUtils.instance.request(model: model)
        as BaseResponseModel<dynamic>;
    
    return response.model<FeatureModel>(
      modelConverter: FeatureModel.fromJson,
    );
  }
  
  // Với list
  @override
  Future<BaseResponseModel<List<FeatureModel>>> fetchList({
    required BaseRequestModel model,
  }) async {
    final response = await DioUtils.instance.request(model: model)
        as BaseResponseModel<dynamic>;
    
    return response.listModel<FeatureModel>(
      modelConverter: FeatureModel.fromJson,
    );
  }
}
```

**Endpoint** - Định nghĩa API paths
```dart
class Endpoint {
  static const prefix = '';
  static const String getData = '$prefix/api/feature/get-data';
  static const String updateData = '$prefix/api/feature/update';
}
```

### 2. Domain Layer
**Repository Interface**
```dart
import 'package:core/core.dart';
import '../../data/models/<feature>_model.dart';

abstract class FeatureRepository {
  Future<BaseResponseModel<FeatureModel>> fetchData({
    required BaseRequestModel model,
  });
}
```

**Usecase** - Business logic
```dart
import 'package:core/core.dart';
import '../../data/models/<feature>_model.dart';
import '../repositories/<feature>_repository.dart';

class FeatureUsecase {
  final FeatureRepository repo;
  
  FeatureUsecase(this.repo);
  
  Future<BaseResponseModel<FeatureModel>> execute() async {
    final model = BaseRequestModel(
      path: '/feature/sample',
      method: DioMethod.get,
      domain: AppDomainType.api,
    );
    
    return repo.fetchData(model: model);
  }
  
  // POST với body
  Future<BaseResponseModel<FeatureModel>> create({
    required Map<String, dynamic> data,
  }) async {
    final model = BaseRequestModel(
      path: '/feature/create',
      method: DioMethod.post,
      domain: AppDomainType.api,
      data: data,
    );
    
    return repo.fetchData(model: model);
  }
}
```

### 3. Presentation Layer
**Cubit State** - Định nghĩa states
```dart
import 'package:core/core.dart';
import '../../data/models/<feature>_model.dart';

abstract class FeatureState {}

class FeatureInit extends FeatureState {}

class FeatureLoading extends FeatureState {}

class FeatureLoaded extends FeatureState {
  final List<FeatureModel>? data;
  FeatureLoaded({this.data});
}

class FeatureError extends FeatureState {
  final String? message;
  FeatureError({this.message});
}
```

**Cubit** - State management
```dart
import 'package:core/core.dart';
import '../../di.dart';
import '../../domain/usecases/<feature>_usecase.dart';

class FeatureCubit extends BaseCubit<FeatureState> {
  FeatureCubit() : super(FeatureInit());
  
  final usecase = sl.get<FeatureUsecase>();
  
  Future<void> loadData() async {
    emit(FeatureLoading());
    
    try {
      final result = await usecase.execute();
      
      if (!result.isSuccess) {
        emit(FeatureError(message: result.message));
        return;
      }
      
      emit(FeatureLoaded(data: result.data));
    } catch (e) {
      emit(FeatureError(message: e.toString()));
    }
  }
}
```

**Page** - UI với BlocBuilder
```dart
import 'package:core/core.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import '../../di.dart';
import '../cubit/<feature>_cubit.dart';

class FeaturePage extends BaseWidget {
  const FeaturePage({super.key});
  
  @override
  BaseWidgetState<FeaturePage> createState() => _FeaturePageState();
}

class _FeaturePageState extends BaseWidgetState<FeaturePage> {
  final _bloc = sl.get<FeatureCubit>();
  
  @override
  void initData() {
    WidgetsBinding.instance.addPostFrameCallback((_) {
      _bloc.loadData();
    });
  }
  
  @override
  void onDispose() {
    _bloc.close();
    super.onDispose();
  }
  
  @override
  Widget buildContent(BuildContext context) {
    return AppBarContainer(
      title: 'Feature Title',
      child: _buildContent(),
    );
  }
  
  Widget _buildContent() {
    return BlocProvider<FeatureCubit>(
      create: (context) => _bloc,
      child: BlocListener<FeatureCubit, FeatureState>(
        listener: (context, state) {
          if (state is FeatureError) {
            AppModal().show(
              iconType: AppModalIconType.failure,
              description: state.message ?? 'Error',
            );
          }
        },
        child: BlocBuilder<FeatureCubit, FeatureState>(
          builder: (context, state) {
            switch (state) {
              case FeatureInit _:
              case FeatureLoading _:
                return _buildSkeleton();
              case FeatureLoaded _:
                return _buildList(state.data);
              case FeatureError _:
                return AppErrorRetryWidget(
                  onRetry: () => _bloc.loadData(),
                );
              default:
                return SizedBox.shrink();
            }
          },
        ),
      ),
    );
  }
  
  Widget _buildList(List<FeatureModel>? data) {
    if (data == null || data.isEmpty) {
      return Center(child: AppEmptyWidget());
    }
    
    return ListView.separated(
      padding: EdgeInsets.all(16),
      itemCount: data.length,
      separatorBuilder: (_, __) => SizedBox(height: 12),
      itemBuilder: (context, index) {
        final item = data[index];
        return ListTile(title: Text(item.data));
      },
    );
  }
  
  Widget _buildSkeleton() {
    return ListView.separated(
      padding: EdgeInsets.all(16),
      itemCount: 5,
      separatorBuilder: (_, __) => SizedBox(height: 12),
      itemBuilder: (_, __) => OneSkeleton(width: double.infinity, height: 60),
    );
  }
}
```

### 4. Dependency Injection
**di.dart**
```dart
import 'package:get_it/get_it.dart';
import 'data/repositories_impl/<feature>_repository_impl.dart';
import 'domain/repositories/<feature>_repository.dart';
import 'domain/usecases/<feature>_usecase.dart';
import 'presentation/cubit/<feature>_cubit.dart';

final sl = GetIt.instance;

class FeatureDI {
  static void register() {
    // Repository
    sl.registerLazySingleton<FeatureRepository>(
      () => FeatureRepositoryImpl(),
    );
    
    // Usecase
    sl.registerFactory(() => FeatureUsecase(sl()));
    
    // Cubit/Bloc
    sl.registerFactory(() => FeatureCubit());
  }
}
```

### 5. Router
**router.dart**
```dart
import 'package:flutter/material.dart';
import 'package:core/core.dart';
import 'feature/<feature>/presentation/ui/<feature>_page.dart';

enum EnumFeatureRouter { featurePage }

extension EnumFeatureRouterExt on EnumFeatureRouter {
  String router() => '<package_domain>/$name';
}

class FeatureRouter extends OneRouter {
  FeatureRouter() : super(domain: '<package_domain>');
  
  @override
  Map<String, WidgetBuilder> register() => {
    EnumFeatureRouter.featurePage.router(): 
        (_) => const FeaturePage(),
  };
}
```

**router_composer.dart**
```dart
import 'package:core/core.dart';
import 'package:<package>/router.dart';

class FeatureRouterComposer {
  static void register() {
    RouterManager.instance.addRouterModule(FeatureRouter());
  }
}
```

### 6. SDK Initialization
**sdk.dart**
```dart
import 'package:core/core.dart';
import 'package:<package>/feature/<feature>/di.dart';
import 'package:<package>/feature/<feature>/router_composer.dart';
import 'translate/vi.dart';
import 'translate/en.dart';

export 'package:<package>/sdk.dart';

extension StringSdk on String {
  String get trSdk =>
      OneTranslateController.translates('<package>')[this] ?? this;
}

class PackageSdk {
  static void init() {
    // Register translations
    OneTranslateController.instance.addOneTranslate([
      PackageVi(),
      PackageEn(),
    ]);
    
    // Register router
    FeatureRouterComposer.register();
    
    // Register DI
    FeatureDI.register();
    
    // Listen to deeplink events
    OneEventController.instance.getEvent((event) {
      if (event is! DeeplinkEvent) return;
      if (event.domain != '<package>') return;
      
      navigator?.pushNamed(
        '${event.domain}/${event.destination}',
        arguments: event.param,
      );
    });
  }
}
```

## Sử dụng script init_feature.sh

Script tự động tạo toàn bộ cấu trúc:

```bash
# Trong thư mục package
./init_feature.sh <feature_name>

# Ví dụ
./init_feature.sh account_notification
```

Script sẽ:
1. Tạo cấu trúc thư mục đầy đủ
2. Generate boilerplate code (models, repositories, usecases, cubit, page)
3. Tạo DI và router
4. Inject vào sdk.dart
5. Thêm dependencies vào pubspec.yaml
6. Chạy build_runner tự động

## Các pattern quan trọng

### 1. BaseRequestModel patterns
```dart
// GET request
final model = BaseRequestModel(
  path: '/api/endpoint',
  method: DioMethod.get,
  domain: AppDomainType.api,
);

// POST with body
final model = BaseRequestModel(
  path: '/api/endpoint',
  method: DioMethod.post,
  domain: AppDomainType.api,
  data: {'key': 'value'},
);

// PUT with headers
final model = BaseRequestModel(
  path: '/api/endpoint',
  method: DioMethod.put,
  domain: AppDomainType.api,
  headers: {'X-Device-Id': AppManager().getDeviceId},
  data: [
    {'accountNumber': '123', 'action': 'REGISTER'}
  ],
);
```

### 2. Response handling
```dart
// Single object
return response.model<FeatureModel>(
  modelConverter: FeatureModel.fromJson,
);

// List
return response.listModel<FeatureModel>(
  modelConverter: FeatureModel.fromJson,
);

// Check success
if (!result.isSuccess) {
  emit(FeatureError(message: result.message));
  return;
}
```

### 3. Navigation
```dart
// Push to route
navigator?.pushNamed(
  EnumFeatureRouter.featurePage.router(),
  arguments: args,
);

// With arguments
navigator?.pushNamed(
  CoreRouterEnum.changePasswordPage.router(),
);
```

### 4. Translation
```dart
// Trong sdk.dart định nghĩa extension
extension StringSdk on String {
  String get trSdk =>
      OneTranslateController.translates('<package>')[this] ?? this;
}

// Sử dụng
Text('feature_title'.trSdk)
Text('common_text'.trCore)
Text('app_text'.trApp)

// Với params
'error_message'.trParam('<package>', ['param1', 'param2'])
```

### 5. Loading & Modals
```dart
// Show loading
AppLoading().show();
AppLoading().hide();

// Show modal
AppModal().show(
  iconType: AppModalIconType.failure,
  description: 'Error message',
  button1: AppModalButton(title: 'close'.trCore),
);
```

## Checklist tạo feature mới

- [ ] Chạy `./init_feature.sh <feature_name>`
- [ ] Sửa model trong `data/models/<feature>_model.dart`
- [ ] Thêm endpoint vào `data/endpoint.dart` (nếu nhiều API)
- [ ] Implement business logic trong `domain/usecases/<feature>_usecase.dart`
- [ ] Định nghĩa states trong `presentation/cubit/<feature>_cubit.dart`
- [ ] Implement UI trong `presentation/ui/<feature>_page.dart`
- [ ] Thêm translation vào `translate/vi.dart`, `translate/en.dart`
- [ ] Test navigation bằng DeeplinkEvent
- [ ] Chạy `dart run build_runner build --delete-conflicting-outputs`
- [ ] Verify DI registration trong sdk.dart

## Lưu ý quan trọng

1. **BaseWidget lifecycle**:
   - `initData()`: Gọi sau khi widget mount
   - `onDispose()`: Cleanup resources
   - `buildContent()`: Build UI chính

2. **Cubit cleanup**:
   Luôn close cubit trong `onDispose()`:
   ```dart
   @override
   void onDispose() {
     _bloc.close();
     super.onDispose();
   }
   ```

3. **BlocProvider position**:
   - Provider ở ngoài cùng
   - Listener cho side effects
   - Builder cho UI updates

4. **State management**:
   - Init: Initial state
   - Loading: Đang fetch data
   - Loaded: Data thành công
   - Error: Có lỗi xảy ra

5. **Response parsing**:
   - `response.model<T>()`: Single object
   - `response.listModel<T>()`: List objects
   - Luôn check `result.isSuccess` trước khi dùng data

## Pitfalls thường gặp

1. **Quên close BLoC/Cubit** → Memory leak
   - Fix: Luôn close trong onDispose()

2. **Không check isSuccess** → Crash khi API fail
   - Fix: Kiểm tra result.isSuccess trước

3. **Quên addPostFrameCallback trong initData** → Race condition
   - Fix: Wrap API call trong WidgetsBinding.instance.addPostFrameCallback

4. **BlocProvider không đúng vị trí** → BlocBuilder không nhận state
   - Fix: Provider phải wrap Listener và Builder

5. **Quên run build_runner** → Model .g.dart không có
   - Fix: `dart run build_runner build --delete-conflicting-outputs`

6. **Package name sai trong rootBundle.loadString**
   - Fix: `packages:<package_name>/feature/...`

## Tài liệu tham khảo
- Core SDK: `packages/core/lib/core/`
- BaseWidget: `packages/core/lib/core/base_widget/`
- BaseCubit: `packages/core/lib/core/base_bloc/`
- Router: `packages/core/lib/core/one_router/`
- Example: `packages/settings/`
