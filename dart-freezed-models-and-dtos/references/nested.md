# Nested objects, lists, and JSON keys

This example uses the feature `users`: models belong in `lib/domain/models/users/`, DTOs in `lib/data/dtos/users/`. Replace `users` and the package `example_app` as needed.

Reuse the definitions of User and UserDto from the [simple example](simple.md), including includeIfNull: false on UserDto. For this alternative layout, place them in `lib/domain/models/users/user.dart` and `lib/data/dtos/users/user_dto.dart`, updating the model import in UserDto to `package:example_app/domain/models/users/user.dart`. Do not duplicate shared models.

The illustrative UserGroup contract uses group_name, users, and group_owner. Dart fields use camelCase and the DTO uses FieldRename.snake. This contract treats a missing users list as empty; do not assume that default for other contracts.

## Model: lib/domain/models/users/user_group.dart

```dart
import 'package:example_app/domain/models/users/user.dart';
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_group.freezed.dart';

@freezed
abstract class UserGroup with _$UserGroup {
  const factory UserGroup({
    required String groupName,
    @Default(<User>[]) List<User> users,
    User? groupOwner,
  }) = _UserGroup;
}
```

The model holds User values and has no JSON annotations or custom methods.

## DTO: lib/data/dtos/users/user_group_dto.dart

```dart
import 'package:example_app/data/dtos/users/user_dto.dart';
import 'package:example_app/domain/models/users/user_group.dart';
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_group_dto.freezed.dart';
part 'user_group_dto.g.dart';

@freezed
abstract class UserGroupDto with _$UserGroupDto {
  const UserGroupDto._();

  @JsonSerializable(
    explicitToJson: true,
    includeIfNull: false,
    fieldRename: FieldRename.snake,
  )
  const factory UserGroupDto({
    required String groupName,
    @Default(<UserDto>[]) List<UserDto> users,
    UserDto? groupOwner,
  }) = _UserGroupDto;

  factory UserGroupDto.fromJson(Map<String, dynamic> json) =>
      _$UserGroupDtoFromJson(json);

  factory UserGroupDto.fromDomain(UserGroup model) => UserGroupDto(
    groupName: model.groupName,
    users: model.users.map((user) => UserDto.fromDomain(user)).toList(),
    groupOwner: model.groupOwner == null
        ? null
        : UserDto.fromDomain(model.groupOwner!),
  );

  UserGroup toDomain() => UserGroup(
    groupName: groupName,
    users: users.map((user) => user.toDomain()).toList(),
    groupOwner: groupOwner?.toDomain(),
  );
}
```

Combine settings on the main factory constructor:

- explicitToJson: true is required for users and turns nested DTOs into JSON maps.
- includeIfNull: false is required for nullable groupOwner and omits its key when null.
- FieldRename.snake maps groupName to group_name and groupOwner to group_owner.

These settings apply to this DTO. Nested UserDto has its own includeIfNull: false, so null age is also omitted.

## Round trip

After generation (imports omitted):

```dart
const user = User(id: 1, name: 'Anna');
const group = UserGroup(
  groupName: 'Developers',
  users: [user],
  groupOwner: user,
);

final dto = UserGroupDto.fromDomain(group);
final json = dto.toJson();
final restored = UserGroupDto.fromJson(json).toDomain();
```

Expected json:

```json
{
  "group_name": "Developers",
  "users": [
    {"id": 1, "name": "Anna"}
  ],
  "group_owner": {"id": 1, "name": "Anna"}
}
```

restored equals group by value. Minimal input and output:

```dart
final empty = UserGroupDto.fromJson({'group_name': 'Guests'}).toDomain();
final emptyJson = UserGroupDto.fromDomain(empty).toJson();
```

empty has groupName Guests, an empty users list, and null groupOwner. Expected emptyJson:

```json
{
  "group_name": "Guests",
  "users": []
}
```

The group_owner key is absent.

## Exceptional JSON key

If an existing Dart field must remain groupOwner but the external contract requires primary_contact, replace its DTO declaration with:

```dart
@JsonKey(name: 'primary_contact') UserDto? groupOwner,
```

JsonKey.name overrides FieldRename.snake for that field, for both reading and writing. The model and domain mappings retain groupOwner. Without a Dart naming constraint, prefer primaryContact in both classes and FieldRename.snake to avoid an unnecessary annotation.

Run [generation and validation](../SKILL.md#generation-and-validation) after source changes.
