# Integer timestamps

Use this example when JSON/Swagger defines a date encoded as an integer. Not every int is a date. Here created_at is a nullable Unix timestamp in **milliseconds**.

LogEntry and the package example_app are illustrative. Shared paths are shown; for a feature, add matching {feature_name} subdirectories under lib/domain/models/ and lib/data/dtos/ and update imports.

## Model: lib/domain/models/log_entry.dart

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'log_entry.freezed.dart';

@freezed
abstract class LogEntry with _$LogEntry {
  const factory LogEntry({
    required int id,
    DateTime? createdAt,
  }) = _LogEntry;
}
```

The model stores DateTime? without JSON annotations or custom methods.

## DTO: lib/data/dtos/log_entry_dto.dart

```dart
import 'package:example_app/domain/models/log_entry.dart';
import 'package:freezed_annotation/freezed_annotation.dart';

part 'log_entry_dto.freezed.dart';
part 'log_entry_dto.g.dart';

@freezed
abstract class LogEntryDto with _$LogEntryDto {
  const LogEntryDto._();

  @JsonSerializable(
    includeIfNull: false,
    fieldRename: FieldRename.snake,
  )
  const factory LogEntryDto({
    required int id,
    @JsonKey(
      fromJson: _timestampFromJson,
      toJson: _timestampToJson,
    )
    DateTime? createdAt,
  }) = _LogEntryDto;

  factory LogEntryDto.fromJson(Map<String, dynamic> json) =>
      _$LogEntryDtoFromJson(json);

  factory LogEntryDto.fromDomain(LogEntry model) =>
      LogEntryDto(id: model.id, createdAt: model.createdAt);

  LogEntry toDomain() => LogEntry(id: id, createdAt: createdAt);
}

DateTime? _timestampFromJson(int? value) => value == null
    ? null
    : DateTime.fromMillisecondsSinceEpoch(value, isUtc: true);

int? _timestampToJson(DateTime? value) => value?.millisecondsSinceEpoch;
```

Converters are private top-level functions in the DTO file, outside its class; do not use static.

FieldRename.snake maps createdAt to created_at without JsonKey.name. includeIfNull: false omits null dates. Domain mappings pass DateTime directly; integers appear only in JSON.

## Round trip

Input:

```json
{
  "id": 1,
  "created_at": 1704067200000
}
```

After generation (imports omitted):

```dart
final dto = LogEntryDto.fromJson({
  'id': 1,
  'created_at': 1704067200000,
});
final model = dto.toDomain();
final json = LogEntryDto.fromDomain(model).toJson();
```

model.createdAt equals DateTime.utc(2024, 1, 1); json contains the original integer 1704067200000.

Both `{"id": 2}` and `{"id": 2, "created_at": null}` decode to a null date. Encoding produces:

```json
{
  "id": 2
}
```

Timestamp 0 means 1970-01-01T00:00:00Z; it must not become null.

## Required date

For a non-nullable field, replace DateTime? createdAt with required DateTime createdAt in both classes, retain JsonKey on the DTO field, and replace the converters with:

```dart
DateTime _timestampFromJson(int value) =>
    DateTime.fromMillisecondsSinceEpoch(value, isUtc: true);

int _timestampToJson(DateTime value) => value.millisecondsSinceEpoch;
```

Replace the nullable pair; do not declare two functions with the same name. If no other fields are nullable, includeIfNull: false is no longer required.

For required and nullable dates using the same unit in one DTO, keep the non-nullable pair and add top-level wrappers:

```dart
DateTime? _nullableTimestampFromJson(int? value) =>
    value == null ? null : _timestampFromJson(value);

int? _nullableTimestampToJson(DateTime? value) =>
    value == null ? null : _timestampToJson(value);
```

Reference wrappers in the nullable field's JsonKey and the main pair on required fields. Keep includeIfNull: false on a DTO with nullable fields.

## Seconds

If the contract specifies **seconds**, replace the nullable converters with:

```dart
DateTime? _timestampFromJson(int? value) => value == null
    ? null
    : DateTime.fromMillisecondsSinceEpoch(
        value * Duration.millisecondsPerSecond,
        isUtc: true,
      );

int? _timestampToJson(DateTime? value) => value == null
    ? null
    : value.millisecondsSinceEpoch ~/ Duration.millisecondsPerSecond;
```

1704067200 now decodes to DateTime.utc(2024, 1, 1) and encodes back to 1704067200. Do not mix seconds and milliseconds. Integer seconds do not preserve subsecond precision. Determine the unit from the contract, not digit count.

## Exceptional JSON key

If fieldRename cannot produce the required key, combine the name and converters in one annotation. For an existing createdAt field with the JSON key timestamp:

```dart
@JsonKey(
  name: 'timestamp',
  fromJson: _timestampFromJson,
  toJson: _timestampToJson,
)
DateTime? createdAt,
```

When Dart names are unconstrained, follow the skill's source-name rules first.

Run [generation and validation](../SKILL.md#generation-and-validation) after source changes.
