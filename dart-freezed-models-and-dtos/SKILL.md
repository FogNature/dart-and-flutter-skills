---
name: dart-freezed-models-and-dtos
description: Create or update Dart/Flutter domain models with Freezed and DTOs with Freezed and json_serializable, including JSON serialization and fromDomain/toDomain mapping.
---

# Dart Freezed models and DTOs

Keep domain models separate from transfer/storage formats. Models use only Freezed; DTOs use Freezed and json_serializable.

## Project context and file layout

Read project instructions (including AGENTS.md if present), related models, DTOs, usages, and tests before editing. Check pubspec.yaml for the package name and compatible dependencies:

- Dependencies: freezed_annotation, json_annotation.
- Dev dependencies: freezed, json_serializable, build_runner.

Use these locations:

| Scope | Models and enums | DTOs |
| --- | --- | --- |
| Shared | `lib/domain/models/` | `lib/data/dtos/` |
| Feature | `lib/domain/models/{feature_name}/` | `lib/data/dtos/{feature_name}/` |

Use the same `snake_case` feature name in both layers, following existing feature boundaries. Import shared models rather than duplicating them. Do not move unrelated files. Package imports omit `lib/`, e.g. `package:example_app/domain/models/users/user.dart`.

Examples use the placeholder package `example_app` and illustrative entities. Adapt package, feature, entity names, and fields to the task; do not add example entities unless requested.

## Field names

- Always use `camelCase` in models and DTOs.
- When deriving fields from JSON or Swagger/OpenAPI, preserve source names that already use camelCase. Use matching names in the model and DTO.
- Convert snake_case keys to camelCase fields, e.g. `group_name` to `groupName`, and configure `FieldRename.snake` on the DTO.
- If a Dart name cannot match the JSON key directly or through fieldRename, add `@JsonKey(name: 'exact_json_key')` to the DTO field. Avoid arbitrary renaming and unnecessary annotations.
- Keep JSON annotations out of Freezed model classes.

## Domain models

Use `@freezed`, an `abstract class`, its generated mixin, and a `const factory` declaring fields. Use entity names and snake_case filenames, e.g. `User` in `user.dart`.

Include only `part 'user.freezed.dart';`. Do not add a private constructor, custom methods/getters, business logic, JSON annotations, `*.g.dart`, or JSON/domain conversion methods. Freezed-generated methods are allowed.

## DTOs

Use the entity name plus `Dto`, e.g. `UserDto` in `user_dto.dart`. Require:

- `@freezed`, an abstract class, its mixin, and a const factory declaring fields.
- Both `part 'user_dto.freezed.dart';` and `part 'user_dto.g.dart';`.
- A private constructor: `const UserDto._();`.
- `factory UserDto.fromJson(Map<String, dynamic> json) => _$UserDtoFromJson(json);`.
- `factory UserDto.fromDomain(User model)` explicitly mapping model fields.
- `User toDomain()` constructing the model from the current DTO.

Let json_serializable generate `toJson()`; do not implement it manually. DTOs may import models; models must not import DTOs. Freezed integrates JSON generation through fromJson and the g.dart part. JSON annotations are available through freezed_annotation.

Preserve required/nullable types and intentional defaults. Do not invent missing data. Map nested models through their DTOs, lists with `map(...).toList()`, and nullable values with null checks.

## JSON configuration

Place one `@JsonSerializable(...)` on the main `const factory DtoName({...})` that declares fields, not on the private constructor, fromJson, or fromDomain. Combine all applicable settings:

| Condition | Required setting |
| --- | --- |
| Any list, including scalar lists | `explicitToJson: true` |
| Any nullable field (`Type?`) | `includeIfNull: false` |
| JSON field names use snake_case | `fieldRename: FieldRename.snake` |

Also use explicitToJson for nested DTOs without lists when toJson must return nested JSON maps. Configure each nested DTO independently. includeIfNull controls output only; preserve input required/null/default semantics.

For exceptional field names, use `@JsonKey(name: ...)`; it overrides fieldRename for that field. Preserve existing JSON keys and backward compatibility, using contract-appropriate defaults or explicit conversions when necessary.

## String enums

For a string field with a finite set of allowed values, create a separate enum file beside the domain model and use the same enum type in model and DTO.

- Obtain the full set from the contract or Swagger/OpenAPI enum; one sample string does not establish a closed set.
- Use UpperCamelCase for the type, camelCase for values, and snake_case for the file, e.g. `UserStatus.pendingReview` in `user_status.dart`.
- Import `package:json_annotation/json_annotation.dart`. Annotate every value with `@JsonValue('exact_contract_value')`, including matching spellings. Preserve case and use unique JSON values.
- Add `part 'user_status.g.dart';` and `@JsonEnum(alwaysCreate: true)` above the enum to generate its mapping even before a DTO uses it. JsonValue still defines each serialized value.
- Use enum-typed DTO fields directly. Do not add enum conversion JsonKey annotations, custom fromJson/toJson methods, jsonValue fields, or adapters. Do not serialize through index or toString().
- JSON annotations stay in the separate enum file; Freezed model classes remain free of JSON annotations and custom methods. fromDomain/toDomain pass enum values directly.
- For nullable enums, use `EnumName?` in both classes and includeIfNull: false on the DTO; no converters are needed.
- FieldRename changes field names, not enum values. JsonValue defines enum strings. JsonKey(name: ...) remains applicable to exceptional field keys.
- Unknown strings must fail deserialization unless the contract defines a fallback. Do not invent an unknown value or silently substitute a valid one.

## Integer timestamps

When the contract defines an integer as a date timestamp, use `DateTime` or `DateTime?` in both model and DTO. Convert only at the JSON boundary.

- Define private top-level functions `_timestampFromJson` and `_timestampToJson` in the DTO file, outside the class. Top-level Dart functions do not use `static`.
- Annotate the DTO field with `@JsonKey(fromJson: _timestampFromJson, toJson: _timestampToJson)`. Combine name and converters in one JsonKey if a custom key is needed.
- Determine seconds vs milliseconds from the contract; never guess from digit count. Clarify an unspecified unit before implementing conversion. Read and write the same unit.
- Decode with `DateTime.fromMillisecondsSinceEpoch(..., isUtc: true)`. Multiply seconds by `Duration.millisecondsPerSecond` before decoding; reverse this when encoding.
- Match converter nullability: int to DateTime or int? to DateTime?. Preserve null and configure includeIfNull: false for nullable fields.
- Pass DateTime directly in fromDomain/toDomain. Keep converters out of the model.

## Examples

Read only the reference needed for the task:

- [Simple](references/simple.md): shared model/DTO, scalar fields, omitted nulls.
- [Nested](references/nested.md): feature layout, lists, nested DTOs, snake_case and exceptional keys.
- [Enums](references/enums.md): JsonValue, enum g.dart generation, JSON round trip.
- [Timestamps](references/timestamps.md): nullable/required dates, seconds/milliseconds, custom keys.

## Generation and validation

After creating or editing models, DTOs, or enums, run from the Dart/Flutter project root:

```bash
dart run build_runner build --delete-conflicting-outputs
```

Wait for successful generation before analysis, tests, or using new DTOs. One build_runner invocation generates model, DTO, and enum files; no separate enum stage is required. Check that each enum has its g.dart file beside its source.

Never edit `*.freezed.dart` or `*.g.dart` manually. Fix source/configuration errors and regenerate. Review the diff, including generated files, for unexpected changes.

Follow project validation instructions. Typical Dart commands are `dart format .`, `dart analyze`, and `dart test`; use `flutter analyze` and `flutter test` for Flutter projects.

Add/update relevant tests when serialized data changes: JSON and domain round trips, nulls, nested objects, omitted null fields, snake_case keys, and JsonKey overrides. Test every enum value in both directions and unknown input. For timestamps, check round trips, zero, nullable/missing input, and the UTC instant. If data is persisted, check save/restore and older JSON compatibility. Report failed or unavailable commands; do not claim successful generation or validation without evidence.
