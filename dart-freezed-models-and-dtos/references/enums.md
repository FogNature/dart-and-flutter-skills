# String enums with JsonValue

Use an enum for a string field with a finite set of allowed values. This contract permits active, pending_review, and disabled. Annotate every enum value with its exact JSON string; json_serializable generates conversion.

The package example_app is a placeholder. Shared paths are shown; for a feature, add {feature_name} under lib/domain/models/ and lib/data/dtos/, keep the enum beside the model, and update imports.

## Enum: lib/domain/models/user_status.dart

```dart
import 'package:json_annotation/json_annotation.dart';

part 'user_status.g.dart';

@JsonEnum(alwaysCreate: true)
enum UserStatus {
  @JsonValue('active')
  active,
  @JsonValue('pending_review')
  pendingReview,
  @JsonValue('disabled')
  disabled,
}
```

Keep enum names in camelCase and exact contract strings in JsonValue. Annotate every value, including matching spellings. JSON values must be unique for unambiguous decoding.

The part directive includes the enum's generated file. JsonEnum(alwaysCreate: true) generates _$UserStatusEnumMap in this library even before a DTO uses the enum; see [JsonEnum.alwaysCreate](https://pub.dev/documentation/json_annotation/latest/json_annotation/JsonEnum/alwaysCreate.html).

JsonEnum enables generation; JsonValue defines serialized values. No custom fromJson/toJson methods, string-value field, or adapters are needed. build_runner creates user_status.g.dart; never write it manually.

## Model: lib/domain/models/user_account.dart

```dart
import 'package:example_app/domain/models/user_status.dart';
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_account.freezed.dart';

@freezed
abstract class UserAccount with _$UserAccount {
  const factory UserAccount({
    required int id,
    required UserStatus status,
  }) = _UserAccount;
}
```

JSON annotations belong to the separate enum file. The Freezed model has no JSON annotations or custom methods.

## DTO: lib/data/dtos/user_account_dto.dart

```dart
import 'package:example_app/domain/models/user_account.dart';
import 'package:example_app/domain/models/user_status.dart';
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_account_dto.freezed.dart';
part 'user_account_dto.g.dart';

@freezed
abstract class UserAccountDto with _$UserAccountDto {
  const UserAccountDto._();

  const factory UserAccountDto({
    required int id,
    required UserStatus status,
  }) = _UserAccountDto;

  factory UserAccountDto.fromJson(Map<String, dynamic> json) =>
      _$UserAccountDtoFromJson(json);

  factory UserAccountDto.fromDomain(UserAccount model) =>
      UserAccountDto(id: model.id, status: model.status);

  UserAccount toDomain() => UserAccount(id: id, status: status);
}
```

Both classes use the same enum; domain mappings pass it directly. No field-level JsonKey converter is needed: json_serializable reads JsonValue from the enum.

This DTO has no lists, nullable fields, or renamed JSON keys, so no extra JsonSerializable annotation is needed. FieldRename.snake changes field keys; JsonValue defines enum strings.

## Round trip

Input:

```json
{
  "id": 1,
  "status": "pending_review"
}
```

After generation (imports omitted):

```dart
final dto = UserAccountDto.fromJson({
  'id': 1,
  'status': 'pending_review',
});
final model = dto.toDomain();
final json = UserAccountDto.fromDomain(model).toJson();
```

model.status equals UserStatus.pendingReview; encoding writes pending_review. Check all allowed values in both directions, including active and disabled.

## Nullable and unknown values

For a nullable field, use UserStatus? in both classes and add JsonSerializable(includeIfNull: false) to the DTO's main factory constructor. Declare the field in both constructors as:

```dart
UserStatus? status,
```

Null or an absent optional key decodes to null; encoding omits the key. No extra converters or JsonKey annotations are needed.

Unknown strings must fail deserialization unless the contract defines a fallback. Do not invent an unknown value or substitute active.

Run [generation and validation](../SKILL.md#generation-and-validation) after editing source, enum values, or annotations. Wait for generation before analysis, tests, or using DTOs. One invocation generates user_status.g.dart beside its source and the model/DTO files. In a feature layout, the enum's generated file stays in the same feature subdirectory. Check the generated files; never edit them manually.
