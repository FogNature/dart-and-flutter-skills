# List loading

For a single BLoC in the `items` feature, use `lib/application/items/`. The example assumes an existing DI-registered `ItemsRepository` with `Future<List<String>> loadItems()`. Use the project's import and contract.

## `items_bloc.dart`

```dart
import 'package:bloc/bloc.dart';
import 'package:bloc_concurrency/bloc_concurrency.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:injectable/injectable.dart';

import 'items_repository.dart';

part 'items_bloc.freezed.dart';
part 'items_event.dart';
part 'items_state.dart';

@injectable
class ItemsBloc extends Bloc<ItemsEvent, ItemsState> {
  final ItemsRepository _repository;

  ItemsBloc(this._repository) : super(const ItemsState.initial()) {
    on<_Init>(_onInit, transformer: droppable());
  }

  Future<void> _onInit(_Init event, Emitter<ItemsState> emit) async {
    emit(const ItemsState.loading());
    try {
      final items = await _repository.loadItems();
      emit(ItemsState.dataLoaded(List.unmodifiable(items)));
    } catch (error) {
      emit(ItemsState.failure(error));
    }
  }
}
```

## `items_event.dart`

```dart
part of 'items_bloc.dart';

@freezed
sealed class ItemsEvent with _$ItemsEvent {
  const factory ItemsEvent.init() = _Init;
}
```

## `items_state.dart`

```dart
part of 'items_bloc.dart';

@freezed
sealed class ItemsState with _$ItemsState {
  const factory ItemsState.initial() = ItemsInitial;
  const factory ItemsState.loading() = ItemsLoading;
  const factory ItemsState.dataLoaded(List<String> items) = ItemsDataLoaded;
  const factory ItemsState.failure(Object error) = ItemsFailure;
}
```

## Existing Flutter widget

When wiring is requested, wrap the existing subtree in `BlocProvider` and use the project's DI accessor.

```dart
BlocProvider<ItemsBloc>(
  create: (_) => GetIt.I<ItemsBloc>()..add(const ItemsEvent.init()),
  child: existingChild,
)
```
