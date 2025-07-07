
# Flutter Clean Architecture Documentation

This document explains the clean architecture structure used in a Flutter project with concrete examples. This is designed to help AI or developers understand how to generate code in this structure.

---

## 1. ENTITY

- Entities are immutable classes that extend `Equatable`.
- Used for comparison and business logic.
- Must not contain any logic or annotations.

```dart
import 'package:equatable/equatable.dart';

class RegionEntity extends Equatable {
  final int id;
  final String title;
  final String slug;

  const RegionEntity({this.id = -1, this.title = '', this.slug = ''});

  @override
  List<Object?> get props => [id, title, slug];
}
```

---

## 2. MODEL

- Models extend Entity.
- Annotated with `@JsonSerializable`.
- Contain `fromJson` and `toJson` methods for serialization.

```dart
import 'package:json_annotation/json_annotation.dart';
import 'package:marjonsuper/features/auth/domain/entity/region_entity.dart';

part 'region_model.g.dart';

@JsonSerializable(fieldRename: FieldRename.snake)
class RegionModel extends RegionEntity {
  const RegionModel({super.id, super.title, super.slug});

  factory RegionModel.fromJson(Map<String, dynamic> json) =>
      _$RegionModelFromJson(json);

  Map<String, dynamic> toJson() => _$RegionModelToJson(this);
}
```

---

## 3. ENTITY CONVERTER

- Used in other models when embedding an entity that must be parsed via its model.
- Ensures consistent parsing.

```dart
class RegionEntityConverter implements JsonConverter<RegionEntity, Map<String, dynamic>> {
  const RegionEntityConverter();

  @override
  RegionEntity fromJson(Map<String, dynamic> json) {
    return RegionModel.fromJson(json);
  }

  @override
  Map<String, dynamic> toJson(RegionEntity object) {
    return {
      'id': object.id,
      'title': object.title,
      'slug': object.slug,
    };
  }
}
```

---

## 4. DATASOURCE

- Abstract and Implementation are defined in the same file.
- Uses Dio + helper methods like `handleRequest()` or `handleVoidRequest()`.

### Abstract:

```dart
abstract class AuthenticationDatasource {
  Future<void> login({required String phone, required String password});
  Future<CompanyInfoModel> getCompanyInfo({required String fullName, required String tin});
}
```

### Implementation:

```dart
class AuthenticationDatasourceImpl implements AuthenticationDatasource {
  final Dio dio;

  AuthenticationDatasourceImpl(this.dio);

  @override
  Future<void> login({required String phone, required String password}) {
    return handleVoidRequest(() => dio.post('/auth/login/', data: FormData.fromMap({
      'phone_number': phone,
      'password': password,
    })).then((res) async {
      await StorageRepository.putString(StorageKeys.accessToken, res.data['access']);
      await StorageRepository.putString(StorageKeys.refreshToken, res.data['refresh']);
      return res;
    }));
  }

  @override
  Future<CompanyInfoModel> getCompanyInfo({required String fullName, required String tin}) {
    return handleRequest(
      () => dio.get('auth/super-app/i-hamkor/', queryParameters: {
        'full_name': fullName,
        'tin': tin,
      }),
      (json) => CompanyInfoModel.fromJson(json),
    );
  }
}
```

---

## 5. REPOSITORY

### Abstract Repository (in `domain/repository/`)

```dart
abstract class AuthenticationRepository {
  Future<Either<Failure, void>> login({required String phone, required String password});
  Future<Either<Failure, CompanyInfoEntity>> getCompanyInfo({required String fullName, required String tin});
}
```

### Implementation (in `data/repository/`)

```dart
class AuthenticationRepositoryImpl extends AuthenticationRepository {
  final AuthenticationDatasource datasource;

  AuthenticationRepositoryImpl({required this.datasource});

  @override
  Future<Either<Failure, void>> login({required String phone, required String password}) async {
    try {
      await datasource.login(phone: phone, password: password);
      return Right(null);
    } catch (e) {
      return Left(handleException(e));
    }
  }

  @override
  Future<Either<Failure, CompanyInfoEntity>> getCompanyInfo({required String fullName, required String tin}) async {
    try {
      final result = await datasource.getCompanyInfo(fullName: fullName, tin: tin);
      return Right(result);
    } catch (e) {
      return Left(handleException(e));
    }
  }
}
```

---

✅ You must use `handleRequest()` and `handleVoidRequest()` for request abstraction.

🛠 `Either<Failure, Result>` pattern must be used in repositories.

🧱 Keep all implementations consistent for scalability and testing.
# UseCase and Bloc Structure in Clean Architecture

This guide explains how to implement UseCases and Bloc in a Clean Architecture approach, suitable for Flutter projects with a structured and testable layout.

---

## 📂 UseCase Layer

Each usecase encapsulates a single responsibility and communicates only with the repository.

**Abstract Base:**
```dart
abstract class UseCase<Type, Params> {
  Future<Either<Failure, Type>> call(Params params);
}
```

**Implementation Example:**
```dart
class RegisterUseCase extends UseCase<void, RegisterParams> {
  final AuthRepository repository;

  RegisterUseCase(this.repository);

  @override
  Future<Either<Failure, void>> call(RegisterParams params) {
    return repository.register(
      phone: params.phone,
      password: params.password,
      firstName: params.firstName,
      lastName: params.lastName,
      fullName: params.fullName,
      tin: params.tin,
      avatar: params.avatar,
      isMale: params.isMale,
      session: params.session,
    );
  }
}
```

---

## 🧠 Bloc Layer

Bloc listens to events, manages state, and calls the corresponding usecases.

**Event Base:**
```dart
abstract class AuthEvent {
  final VoidCallback? onSuccess;
  final Function(String errorMessage)? onError;

  AuthEvent({this.onSuccess, this.onError});
}
```

**Event Example:**
```dart
class GetSessionEvent extends AuthEvent {
  final VerifyType type;

  GetSessionEvent({
    required this.type,
    super.onSuccess,
    super.onError,
  });
}
```

**State Example:**
```dart
class AuthState extends Equatable {
  final FormzSubmissionStatus registerStatus;
  final String phone;
  final String firstName;
  final String lastName;
  final String fullName;
  final String tin;
  final bool isMale;
  final String session;
  final String selectedAvatarPath;
  final RegisterType registerType;

  const AuthState({
    this.registerStatus = FormzSubmissionStatus.initial,
    this.phone = '',
    this.firstName = '',
    this.lastName = '',
    this.fullName = '',
    this.tin = '',
    this.isMale = true,
    this.session = '',
    this.selectedAvatarPath = '',
    this.registerType = RegisterType.individual,
  });
```

**Bloc Example:**
```dart
on<RegisterEvent>(_onRegister);

Future<void> _onRegister(RegisterEvent event, Emitter<AuthState> emit) async {
  emit(state.copyWith(registerStatus: FormzSubmissionStatus.inProgress));
  final result = await _register(RegisterParams(
    phone: "+998${state.phone.replaceAll(' ', '')}",
    password: state.secondPassword,
    firstName: state.firstName,
    lastName: state.lastName,
    fullName: state.fullName,
    tin: state.tin,
    avatar: state.selectedAvatarPath,
    isMale: state.isMale,
    session: state.session,
    type: state.registerType,
  ));
  result.either(
    (l) => blocErrorHandler(l, emit, state.copyWith(registerStatus: FormzSubmissionStatus.failure), event.onError),
    (r) {
      event.onSuccess?.call();
      emit(state.copyWith(registerStatus: FormzSubmissionStatus.success));
    },
  );
}
```

---

## 📁 Folder Structure & File Naming Tree

```
lib/
└── features/
    └── feature/
        ├── data/
        │   ├── data_source/
        │   │   └── feature_data_source.dart
        │   ├── models/
        │   │   └── feature_model.dart
        │   │   └── ... other models related to this feature
        │   └── repositories/
        │       └── feature_repository_impl.dart
        ├── domain/
        │   ├── entities/
        │   │   ├── feature_entity.dart
        │   │   └── ... other entities related to this feature
        │   ├── repositories/
        │   │   └── feature_repository.dart
        │   └── usecases/
        │       ├── something_usecase.dart
        │       └── ... other usecases related to this feature
        └── presentation/
            ├── bloc/
            │   ├── feature_bloc.dart
            │   ├── feature_evant.dart
            │   └── feature_state.dart
            ├── mixins/
            │   └── feature_mixin.dart            # logic: event triggering, validation, navigation
            ├── pages/
            │   ├── something_page.dart
            │   └── ... other pages related to this feature
            ├── widgets/                          # Custom widgets used across the app
            │   ├── feature_list_item.dart
            │   └── ... other widgets related to this feature
            └── feature_screen.dart               # only UI layout, uses mixin
```

## 🎯 Purpose

To implement a UI screen (e.g., `AuthScreen`, `ProfileEditScreen`) where:
- All business/UI logic is abstracted into a Mixin.
- The UI code only reflects layout, styling, and widget structure.
- Bloc is used for state changes and action dispatching.
- Custom reusable widgets (like `WButton`, `TextFieldWithLabel`, `WRadio`, etc.) are used to maintain design consistency.

---

## 🔧 How to Write `ExampleScreen`

### 1. **Screen Widget**: Only display UI with layout and custom widget composition.
- Use `BlocSelector` or `BlocBuilder` only for extracting values.
- No logic or `context.read<>.add(...)` calls inside `build()`.

### 2. **Mixin**: Handles all tap, onChanged, validation, and Bloc event dispatch.
- Keeps `State` class clean.
- Makes screen reusable and easier to test.

---

## 🧪 Example Use: `AuthScreen`

### ➤ Event dispatching (from mixin)

```dart
void onLogin() {
  if (context.read<AuthBloc>().state.phone.isEmpty) {
    context.showPopUp(status: PopUpStatus.error, message: 'Phone required');
    return;
  }
  context.read<AuthBloc>().add(LoginEvent(
    onSuccess: () => Navigator.push(...),
    onError: (msg) => context.showErrorPopUp(message: msg),
  ));
}
```

---

If you follow this pattern:
✅ Your screens remain clean  
✅ Your logic stays testable  
✅ Reusability and clarity improve dramatically
## 🧠 General Principles
- Follow Clean Architecture: separate domain, data, and presentation layers.
- State management is done via `Bloc` and `FormzSubmissionStatus`.
- All asynchronous logic (API calls) must be handled in UseCases and Repositories.
- Pagination must support `fetchMore`, `next`, and `List<Entity>` with `GenericPagination<Model>` response.
- UI logic like button taps must go into mixins if needed.
- Avoid duplication, use `BlocSelector`, `BlocBuilder` and reuse widgets like `Paginator`.

---

## 🔁 Pagination Guidelines

### Datasource
```dart
Future<GenericPagination<NotificationModel>> getNotifications({String? next});
```

### Repository
```dart
Future<Either<Failure, GenericPagination<NotificationEntity>>> getNotifications({String? next});
```

### UseCase
```dart
class GetNotificationsUseCase extends UseCase<GenericPagination<NotificationEntity>, String?> {
  final NotificationRepository repository;

  GetNotificationsUseCase({required this.repository});

  @override
  Future<Either<Failure, GenericPagination<NotificationEntity>>> call(String? next) {
    return repository.getNotifications(next: next);
  }
}
```

### Bloc Events
```dart
class GetNotifications extends NotificationEvent {}
class GetMoreNotifications extends NotificationEvent {}
```

### Bloc State
```dart
final List<NotificationEntity> notifications;
final String? notificationNext;
final bool notificationsFetchMore;
final FormzSubmissionStatus notificationsStatus;
```

### Bloc Logic
```dart
on<GetNotifications>((event, emit) async {
  emit(state.copyWith(notificationsStatus: FormzSubmissionStatus.inProgress));
  final result = await getNotificationsUseCase.call(null);
  result.either(
    (l) => emit(state.copyWith(notificationsStatus: FormzSubmissionStatus.failure, errorMessage: l.errorMessage)),
    (r) => emit(state.copyWith(
      notifications: r.results,
      notificationNext: r.next,
      notificationsFetchMore: r.next != null,
      notificationsStatus: FormzSubmissionStatus.success,
    )),
  );
});

on<GetMoreNotifications>((event, emit) async {
  final result = await getNotificationsUseCase.call(state.notificationNext);
  result.either(
    (l) => emit(state),
    (r) => emit(state.copyWith(
      notifications: [...state.notifications, ...r.results],
      notificationNext: r.next,
      notificationsFetchMore: r.next != null,
    )),
  );
});
```

---

## 🖼 UI: Notifications Screen with Paginator

### BlocBuilder
```dart
BlocBuilder<NotificationBloc, NotificationState>(
  builder: (context, state) {
    if (state.notificationsStatus.isInProgress) {
      return ShimmerList(); // show loading
    } else if (state.notificationsStatus.isSuccess) {
      if (state.notifications.isEmpty) {
        return NotificationNoData(); // show empty
      } else {
        return Paginator(
          itemCount: state.notifications.length,
          itemBuilder: (context, index) => NotificationItem(
            title: state.notifications[index].notification.title,
            ...
            onTab: () => Navigator.push(...),
          ),
          separatorBuilder: (_, __) => const SizedBox(height: 12),
          fetchMoreFunction: () => context.read<NotificationBloc>().add(GetMoreNotifications()),
          hasMoreToFetch: state.notificationsFetchMore,
          paginatorStatus: state.notificationsStatus,
        );
      }
    } else {
      return FailScreen(message: state.errorMessage); // show error
    }
  },
);
```

