---
name: dart-list-blocs
description: Create or update Dart and Flutter BLoCs for lists (including pagination) or on-demand loading of other data. Uses bloc, flutter_bloc, bloc_concurrency, Freezed, and injectable.
---

# Dart List BLoCs

Adapt the [list template](references/items-bloc.md) for full lists or other data. Use the [pagination template](references/items-pagination-bloc.md) for incremental loading with `loadPage()`.

## Structure

- Check the target package's `pubspec.yaml`, nearby BLoCs, DI, and generator settings. Reuse the project's repository, models, and error contracts. If the repository is missing, describe the required contract in the reply. Without a project, provide source and commands.
- Keep new BLoC, event, and state files together under `lib/application/{feature}/`. For multiple BLoCs in a feature, use `lib/application/{feature}/{bloc_name}/`, where `{bloc_name}` is the BLoC filename without `.dart`.
- Use `Bloc<Event, State>` from `package:bloc/bloc.dart`, `on<E>`, and Freezed events and states. Use an immutable data class with `copyWith` for a fixed set of fields and a union for distinct cases. On Dart 3, use `abstract class` for a Freezed data class and `sealed class` for a union. Match the project's Dart and Freezed versions.
- Declare event, state, and matching `*.freezed.dart` parts in the BLoC file. Point both source parts back to it with `part of`. Use public variants for pattern matching outside the library.

## Event handling

- Use `sequential()` for ordered work, `droppable()` to ignore events while busy, `restartable()` when only the latest result matters, and `concurrent()` for independent work. BLoC defaults to concurrent handling unless overridden.
- A transformer applies to one `on<E>` registration. Register the common event type and dispatch within that handler when variants share a queue, cancellation, or drop policy.
- Await work that emits states and map errors according to the feature contract. With `restartable()`, use `emit.isDone` to guard emissions and side effects after an await. Use repository or client cancellation to stop the underlying request when needed.

## DI and Flutter

- Annotate the BLoC with `@injectable` for factory registration and inject dependencies through its constructor. Use the existing DI setup. If DI is missing, keep the annotation and report that registration requires project setup. Put per-event arguments in events and use `@factoryParam` for instance arguments when needed.
- If Flutter wiring is requested, use an existing widget or show a snippet. `BlocProvider(create: ...)` owns and closes new instances. With `BlocProvider.value`, the existing owner handles disposal. Use `BlocBuilder` for rendering and `BlocListener` for one-off effects. In pure Dart, the owner calls `close()`.

## Generation and validation

- Add compatible direct dependencies as needed (`bloc`, `bloc_concurrency`, `freezed_annotation`, `injectable`, plus `flutter_bloc` or `get_it` when imported). Add `freezed`, `injectable_generator`, and `build_runner` as dev dependencies when needed.
- Update `*.freezed.dart` only through generation. Edit and format the source files.
- After creating a BLoC or changing generator inputs, run `dart run build_runner build --delete-conflicting-outputs` from the target package root. Verify the Freezed output and the factory if DI is configured. Report any generation failure or blocker.
- Follow project validation instructions. Typical Dart commands are `dart format .`, `dart analyze`, and `dart test`. Use `flutter analyze` and `flutter test` for Flutter projects. Pass changed source paths to `dart format` if formatting the whole directory would touch generated files.
