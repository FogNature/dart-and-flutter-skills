# Paginated list

Use `lib/application/items/` for a single BLoC or `lib/application/items/items_pagination_bloc/` when the feature has several.

The example assumes an existing DI-registered repository with `Future<({List<String> items, bool hasMore})> loadItems({required int page})` and starts at page 1. Use the project's first page and end-of-list signal. For cursor-based repositories, store and pass the cursor.

## `items_pagination_bloc.dart`

```dart
import 'package:bloc/bloc.dart';
import 'package:bloc_concurrency/bloc_concurrency.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:injectable/injectable.dart';

import 'items_repository.dart';

part 'items_pagination_bloc.freezed.dart';
part 'items_pagination_event.dart';
part 'items_pagination_state.dart';

@injectable
class ItemsPaginationBloc
    extends Bloc<ItemsPaginationEvent, ItemsPaginationState> {
  final ItemsRepository _repository;

  ItemsPaginationBloc(this._repository)
    : super(const ItemsPaginationState.initial()) {
    on<ItemsPaginationEvent>(_onEvent, transformer: droppable());
  }

  Future<void> _onEvent(
    ItemsPaginationEvent event,
    Emitter<ItemsPaginationState> emit,
  ) {
    return switch (event) {
      _Init() => _onInit(emit),
      _LoadPage() => _onLoadPage(emit),
    };
  }

  Future<void> _onInit(Emitter<ItemsPaginationState> emit) async {
    const pageNumber = 1;
    emit(const ItemsPaginationState.loading());
    try {
      final page = await _repository.loadItems(page: pageNumber);
      emit(
        ItemsPaginationState.dataLoaded(
          List.unmodifiable(page.items),
          page.hasMore ? pageNumber + 1 : null,
        ),
      );
    } catch (error) {
      emit(ItemsPaginationState.failure(error));
    }
  }

  Future<void> _onLoadPage(Emitter<ItemsPaginationState> emit) async {
    final pendingPage = switch (state) {
      ItemsPaginationDataLoaded(:final items, nextPage: final int pageNumber) =>
        (items, pageNumber),
      ItemsPaginationPageFailure(:final items, :final pageNumber) => (
        items,
        pageNumber,
      ),
      _ => null,
    };
    if (pendingPage == null) return;

    final (items, pageNumber) = pendingPage;
    emit(ItemsPaginationState.loadingPage(items));
    try {
      final page = await _repository.loadItems(page: pageNumber);
      emit(
        ItemsPaginationState.dataLoaded(
          List.unmodifiable([...items, ...page.items]),
          page.hasMore ? pageNumber + 1 : null,
        ),
      );
    } catch (error) {
      emit(ItemsPaginationState.pageFailure(items, pageNumber, error));
    }
  }
}
```

## `items_pagination_event.dart`

```dart
part of 'items_pagination_bloc.dart';

@freezed
sealed class ItemsPaginationEvent with _$ItemsPaginationEvent {
  const factory ItemsPaginationEvent.init() = _Init;
  const factory ItemsPaginationEvent.loadPage() = _LoadPage;
}
```

## `items_pagination_state.dart`

```dart
part of 'items_pagination_bloc.dart';

@freezed
sealed class ItemsPaginationState with _$ItemsPaginationState {
  const factory ItemsPaginationState.initial() = ItemsPaginationInitial;
  const factory ItemsPaginationState.loading() = ItemsPaginationLoading;
  const factory ItemsPaginationState.dataLoaded(
    List<String> items,
    int? nextPage,
  ) = ItemsPaginationDataLoaded;
  const factory ItemsPaginationState.failure(Object error) =
      ItemsPaginationFailure;
  const factory ItemsPaginationState.loadingPage(List<String> items) =
      ItemsPaginationLoadingPage;
  const factory ItemsPaginationState.pageFailure(
    List<String> items,
    int pageNumber,
    Object error,
  ) = ItemsPaginationPageFailure;
}
```

A shared `droppable()` handler prevents overlapping `init()` and `loadPage()` requests. `loadPage()` retries the failed page and stops when `nextPage` is null.

In the existing Flutter widget, dispatch `init()` on creation and `loadPage()` to load or retry the next page. Keep items visible during `loadingPage` and `pageFailure` with a footer spinner or retry action as needed.
