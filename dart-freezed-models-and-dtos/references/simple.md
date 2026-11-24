# Simple model and DTO

Use this example for a shared entity with scalar fields: models belong in `lib/domain/models/`, DTOs in `lib/data/dtos/`. Replace the illustrative `User` and package `example_app` as needed. For feature-specific entities, add matching `{feature_name}` subdirectories as in the [nested example](nested.md).

## Model: lib/domain/models/user.dart

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';

@freezed
abstract class User with _$User {
  const factory User({
    required int id,
    required String name,
    int? age,
  }) = _User;
}
```

The model has no JSON serialization, private constructor, or custom methods.

## DTO: lib/data/dtos/user_dto.dart

```dart
import 'package:example_app/domain/models/user.dart';
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_dto.freezed.dart';
part 'user_dto.g.dart';

@freezed
abstract class UserDto with _$UserDto {
  const UserDto._();

  @JsonSerializable(includeIfNull: false)
  const factory UserDto({
    required int id,
    required String name,
    int? age,
  }) = _UserDto;

  factory UserDto.fromJson(Map<String, dynamic> json) =>
      _$UserDtoFromJson(json);

  factory UserDto.fromDomain(User model) => UserDto(
    id: model.id,
    name: model.name,
    age: model.age,
  );

  User toDomain() => User(id: id, name: name, age: age);
}
```

Freezed generates immutable data; json_serializable generates JSON conversion. The private constructor enables the instance method toDomain(). includeIfNull: false is required because age is nullable.

Field names match the source JSON/Swagger and use camelCase, so no JsonKey or fieldRename is needed.

## Round trip

After generation (imports omitted):

```dart
const user = User(id: 1, name: 'Anna', age: 25);
final dto = UserDto.fromDomain(user);
final json = dto.toJson();
final restored = UserDto.fromJson(json).toDomain();
```

Expected json:

```json
{
  "id": 1,
  "name": "Anna",
  "age": 25
}
```

restored equals user by Freezed value equality. Null age is omitted:

```dart
final jsonWithoutAge = UserDto.fromDomain(
  const User(id: 2, name: 'Ivan'),
).toJson();
```

Expected jsonWithoutAge:

```json
{
  "id": 2,
  "name": "Ivan"
}
```

An absent optional age key decodes to null. includeIfNull: false controls writing only.

Run [generation and validation](../SKILL.md#generation-and-validation) after source changes.
