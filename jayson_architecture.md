
# Flutter Clean Architecture Documentation (AI Prompt Style)

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
