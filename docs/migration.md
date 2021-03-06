# Migration Guide

?> **Tip**: Please refer to the [release log](https://github.com/felangel/bloc/releases) for more information regarding what changed in each release.

## v6.1.0

### package:flutter_bloc

#### ❗context.bloc and context.repository are deprecated in favor of context.read and context.write

##### Rationale

`context.read`, `context.watch`, and `context.select` were added to align with the existing [provider](https://pub.dev/packages/provider) API which many developers are familiar and to address issues that have been raised by the community. To improve the safety of the code and maintain consistency, `context.bloc` was deprecated because it can be replaced with either `context.read` or `context.watch` dependending on if it's used directly within `build`.

**context.watch**

`context.watch` addresses the request to have a [MultiBlocBuilder](https://github.com/felangel/bloc/issues/538) because we can watch several blocs within a single `Builder` in order to render UI based on multiple states:

```dart
Builder(
  builder: (context) {
    final stateA = context.watch<BlocA>().state;
    final stateB = context.watch<BlocB>().state;
    final stateC = context.watch<BlocC>().state;

    // return a Widget which depends on the state of BlocA, BlocB, and BlocC
  }
);
```

**context.select**

`context.select` allows developers to render/update UI based on a part of a bloc state and addresses the request to have a [simpler buildWhen](https://github.com/felangel/bloc/issues/1521).

```dart
final name = context.select((UserBloc bloc) => bloc.state.user.name);
```

The above snippet allows us to access and rebuild the widget only when the current user's name changes.

**context.read**

Even though it looks like `context.read` is identical to `context.bloc` there are some subtle but significant differences. Both allow you to access a bloc with a `BuildContext` and do not result in rebuilds; however, `context.read` cannot be called directly within a `build` method. There are two main reasons to use `context.bloc` within `build`:

1. **To access the bloc's state**

```dart
@override
Widget build(BuildContext context) {
  final state = context.bloc<MyBloc>().state;
  return Text('$state');
}
```

The above usage is error prone because the `Text` widget will not be rebuilt if the state of the bloc changes. In this scenario, either a `BlocBuilder` or `context.watch` should be used.

```dart
@override
Widget build(BuildContext context) {
  final state = context.watch<MyBloc>().state;
  return Text('$state');
}
```

or

```dart
@override
Widget build(BuildContext context) {
  return BlocBuilder<MyBloc, MyState>(
    builder: (context, state) => Text('$state'),
  );
}
```

!> Using `context.watch` at the root of the `build` method will result in the entire widget being rebuilt when the bloc state changes. If the entire widget does not need to be rebuilt, either use `BlocBuilder` to wrap the parts that should rebuild, use a `Builder` with `context.watch` to scope the rebuilds, or decompose the widget into smaller widgets.

2. **To access the bloc so that an event can be added**

```dart
@override
Widget build(BuildContext context) {
  final bloc = context.bloc<MyBloc>();
  return RaisedButton(
    onPressed: () => bloc.add(MyEvent()),
    ...
  )
}
```

The above usage is inefficient because it results in a bloc lookup on each rebuild when the bloc is only needed when the user taps the `RaisedButton`. In this scenario, prefer to use `context.read` to access the bloc directly where it is needed (in this case, in the `onPressed` callback).

```dart
@override
Widget build(BuildContext context) {
  return RaisedButton(
    onPressed: () => context.read<MyBloc>().add(MyEvent()),
    ...
  )
}
```

**Summary**

**v6.0.x**

```dart
@override
Widget build(BuildContext context) {
  final bloc = context.bloc<MyBloc>();
  return RaisedButton(
    onPressed: () => bloc.add(MyEvent()),
    ...
  )
}
```

**v6.1.x**

```dart
@override
Widget build(BuildContext context) {
  return RaisedButton(
    onPressed: () => context.read<MyBloc>().add(MyEvent()),
    ...
  )
}
```

?> If accessing a bloc to add an event, perform the bloc access using `context.read` in the callback where it is needed.

**v6.0.x**

```dart
@override
Widget build(BuildContext context) {
  final state = context.bloc<MyBloc>().state;
  return Text('$state');
}
```

**v6.1.x**

```dart
@override
Widget build(BuildContext context) {
  final state = context.watch<MyBloc>().state;
  return Text('$state');
}
```

?> Use `context.watch` when accessing the state of the bloc in order to ensure the widget is rebuilt when the state changes.

## v6.0.0

### package:bloc

#### ❗BlocObserver onError takes Cubit

##### Rationale

Due to the integration of `Cubit`, `onError` is now shared between both `Bloc` and `Cubit` instances. Since `Cubit` is the base, `BlocObserver` will accept a `Cubit` type rather than a `Bloc` type in the `onError` override.

**v5.x.x**

```dart
class MyBlocObserver extends BlocObserver {
  @override
  void onError(Bloc bloc, Object error, StackTrace stackTrace) {
    super.onError(bloc, error, stackTrace);
  }
}
```

**v6.0.0**

```dart
class MyBlocObserver extends BlocObserver {
  @override
  void onError(Cubit cubit, Object error, StackTrace stackTrace) {
    super.onError(cubit, error, stackTrace);
  }
}
```

#### ❗Bloc does not emit last state on subscription

##### Rationale

This change was made to align `Bloc` and `Cubit` with the built-in `Stream` behavior in `Dart`. In addition, conforming this the old behavior in the context of `Cubit` led to many unintended side-effects and overall complicated the internal implementations of other packages such as `flutter_bloc` and `bloc_test` unnecessarily (requiring `skip(1)`, etc...).

**v5.x.x**

```dart
final bloc = MyBloc();
bloc.listen(print);
```

Previously, the above snippet would output the initial state of the bloc followed by subsequent state changes.

**v6.x.x**

In v6.0.0, the above snippet does not output the initial state and only outputs subsequent state changes. The previous behavior can be achieved with the following:

```dart
final bloc = MyBloc();
print(bloc.state);
bloc.listen(print);
```

?> **Note**: This change will only affect code that relies on direct bloc subscriptions. When using `BlocBuilder`, `BlocListener`, or `BlocConsumer` there will be no noticeable change in behavior.

### package:bloc_test

#### ❗MockBloc only requires State type

##### Rationale

It is not necessary and eliminates extra code while also making `MockBloc` compatible with `Cubit`.

**v5.x.x**

```dart
class MockCounterBloc extends MockBloc<CounterEvent, int> implements CounterBloc {}
```

**v6.0.0**

```dart
class MockCounterBloc extends MockBloc<int> implements CounterBloc {}
```

#### ❗whenListen only requires State type

##### Rationale

It is not necessary and eliminates extra code while also making `whenListen` compatible with `Cubit`.

**v5.x.x**

```dart
whenListen<CounterEvent,int>(bloc, Stream.fromIterable([0, 1, 2, 3]));
```

**v6.0.0**

```dart
whenListen<int>(bloc, Stream.fromIterable([0, 1, 2, 3]));
```

#### ❗blocTest does not require Event type

##### Rationale

It is not necessary and eliminates extra code while also making `blocTest` compatible with `Cubit`.

**v5.x.x**

```dart
blocTest<CounterBloc, CounterEvent, int>(
  'emits [1] when increment is called',
  build: () async => CounterBloc(),
  act: (bloc) => bloc.add(CounterEvent.increment),
  expect: const <int>[1],
);
```

**v6.0.0**

```dart
blocTest<CounterBloc, int>(
  'emits [1] when increment is called',
  build: () => CounterBloc(),
  act: (bloc) => bloc.add(CounterEvent.increment),
  expect: const <int>[1],
);
```

#### ❗blocTest skip defaults to 0

##### Rationale

Since `bloc` and `cubit` instances will no longer emit the latest state for new subscriptions, it was no longer necessary to default `skip` to `1`.

**v5.x.x**

```dart
blocTest<CounterBloc, CounterEvent, int>(
  'emits [0] when skip is 0',
  build: () async => CounterBloc(),
  skip: 0,
  expect: const <int>[0],
);
```

**v6.0.0**

```dart
blocTest<CounterBloc, int>(
  'emits [] when skip is 0',
  build: () => CounterBloc(),
  skip: 0,
  expect: const <int>[],
);
```

The initial state of a bloc or cubit can be tested with the following:

```dart
test('initial state is correct', () {
  expect(MyBloc().state, InitialState());
});
```

#### ❗blocTest make build synchronous

##### Rationale

Previously, `build` was made `async` so that various preparation could be done to put the bloc under test in a specific state. It is no longer necessary and also resolves several issues due to the added latency between the build and the subscription internally. Instead of doing async prep to get a bloc in a desired state we can now set the bloc state by chaining `emit` with the desired state.

**v5.x.x**

```dart
blocTest<CounterBloc, CounterEvent, int>(
  'emits [2] when increment is added',
  build: () async {
    final bloc = CounterBloc();
    bloc.add(CounterEvent.increment);
    await bloc.take(2);
    return bloc;
  }
  act: (bloc) => bloc.add(CounterEvent.increment),
  expect: const <int>[2],
);
```

**v6.0.0**

```dart
blocTest<CounterBloc, int>(
  'emits [2] when increment is added',
  build: () => CounterBloc()..emit(1),
  act: (bloc) => bloc.add(CounterEvent.increment),
  expect: const <int>[2],
);
```

!> `emit` is only visible for testing and should never be used outside of tests.

### package:flutter_bloc

#### ❗BlocBuilder bloc parameter renamed to cubit

##### Rationale

In order to make `BlocBuilder` interoperate with `bloc` and `cubit` instances the `bloc` parameter was renamed to `cubit` (since `Cubit` is the base class).

**v5.x.x**

```dart
BlocBuilder(
  bloc: myBloc,
  builder: (context, state) {...}
)
```

**v6.0.0**

```dart
BlocBuilder(
  cubit: myBloc,
  builder: (context, state) {...}
)
```

#### ❗BlocListener bloc parameter renamed to cubit

##### Rationale

In order to make `BlocListener` interoperate with `bloc` and `cubit` instances the `bloc` parameter was renamed to `cubit` (since `Cubit` is the base class).

**v5.x.x**

```dart
BlocListener(
  bloc: myBloc,
  listener: (context, state) {...}
)
```

**v6.0.0**

```dart
BlocListener(
  cubit: myBloc,
  listener: (context, state) {...}
)
```

#### ❗BlocConsumer bloc parameter renamed to cubit

##### Rationale

In order to make `BlocConsumer` interoperate with `bloc` and `cubit` instances the `bloc` parameter was renamed to `cubit` (since `Cubit` is the base class).

**v5.x.x**

```dart
BlocConsumer(
  bloc: myBloc,
  listener: (context, state) {...},
  builder: (context, state) {...}
)
```

**v6.0.0**

```dart
BlocConsumer(
  cubit: myBloc,
  listener: (context, state) {...},
  builder: (context, state) {...}
)
```

---

## v5.0.0

### package:bloc

#### ❗initialState has been removed

##### Rationale

As a developer, having to override `initialState` when creating a bloc presents two main issues:

- The `initialState` of the bloc can be dynamic and can also be referenced at a later point in time (even outside of the bloc itself). In some ways, this can be viewed as leaking internal bloc information to the UI layer.
- It's verbose.

**v4.x.x**

```dart
class CounterBloc extends Bloc<CounterEvent, int> {
  @override
  int get initialState => 0;

  ...
}
```

**v5.0.0**

```dart
class CounterBloc extends Bloc<CounterEvent, int> {
  CounterBloc() : super(0);

  ...
}
```

?> For more information check out [#1304](https://github.com/felangel/bloc/issues/1304)

#### ❗BlocDelegate renamed to BlocObserver

##### Rationale

The name `BlocDelegate` was not an accurate description of the role that the class played. `BlocDelegate` suggests that the class plays an active role whereas in reality the intended role of the `BlocDelegate` was for it to be a passive component which simply observes all blocs in an application.

!> There should ideally be no user-facing functionality or features handled within `BlocObserver`.

**v4.x.x**

```dart
class MyBlocDelegate extends BlocDelegate {
  ...
}
```

**v5.0.0**

```dart
class MyBlocObserver extends BlocObserver {
  ...
}
```

#### ❗BlocSupervisor has been removed

##### Rationale

`BlocSupervisor` was yet another component that developers had to know about and interact with for the sole purpose of specifying a custom `BlocDelegate`. With the change to `BlocObserver` we felt it improved the developer experience to set the observer directly on the bloc itself.

?> This changed also enabled us to decouple other bloc add-ons like `HydratedStorage` from the `BlocObserver`.

**v4.x.x**

```dart
BlocSupervisor.delegate = MyBlocDelegate();
```

**v5.0.0**

```dart
Bloc.observer = MyBlocObserver();
```

### package:flutter_bloc

#### ❗BlocBuilder condition renamed to buildWhen

##### Rationale

When using `BlocBuilder`, we previously could specify a `condition` to determine whether the `builder` should rebuild.

```dart
BlocBuilder<MyBloc, MyState>(
  condition: (previous, current) {
    // return true/false to determine whether to call builder
  },
  builder: (context, state) {...}
)
```

The name `condition` is not very self-explanatory or obvious and more importantly, when interacting with a `BlocConsumer` the API became inconsistent because developers can provide two conditions (one for `builder` and one for `listener`). As a result, the `BlocConsumer` API exposed a `buildWhen` and `listenWhen`

```dart
BlocConsumer<MyBloc, MyState>(
  listenWhen: (previous, current) {
    // return true/false to determine whether to call listener
  },
  listener: (context, state) {...},
  buildWhen: (previous, current) {
    // return true/false to determine whether to call builder
  },
  builder: (context, state) {...},
)
```

In order to align the API and provide a more consistent developer experience, `condition` was renamed to `buildWhen`.

**v4.x.x**

```dart
BlocBuilder<MyBloc, MyState>(
  condition: (previous, current) {
    // return true/false to determine whether to call builder
  },
  builder: (context, state) {...}
)
```

**v5.0.0**

```dart
BlocBuilder<MyBloc, MyState>(
  buildWhen: (previous, current) {
    // return true/false to determine whether to call builder
  },
  builder: (context, state) {...}
)
```

#### ❗BlocListener condition renamed to listenWhen

##### Rationale

For the same reasons as described above, the `BlocListener` condition was also renamed.

**v4.x.x**

```dart
BlocListener<MyBloc, MyState>(
  condition: (previous, current) {
    // return true/false to determine whether to call listener
  },
  listener: (context, state) {...}
)
```

**v5.0.0**

```dart
BlocListener<MyBloc, MyState>(
  listenWhen: (previous, current) {
    // return true/false to determine whether to call listener
  },
  listener: (context, state) {...}
)
```

### package:hydrated_bloc

#### ❗HydratedStorage and HydratedBlocStorage renamed

##### Rationale

In order to improve code reuse between [hydrated_bloc](https://pub.dev/packages/hydrated_bloc) and [hydrated_cubit](https://pub.dev/packages/hydrated_cubit), the concrete default storage implementation was renamed from `HydratedBlocStorage` to `HydratedStorage`. In addition, the `HydratedStorage` interface was renamed from `HydratedStorage` to `Storage`.

**v4.0.0**

```dart
class MyHydratedStorage implements HydratedStorage {
  ...
}
```

**v5.0.0**

```dart
class MyHydratedStorage implements Storage {
  ...
}
```

#### ❗HydratedStorage decoupled from BlocDelegate

##### Rationale

As mentioned earlier, `BlocDelegate` was renamed to `BlocObserver` and was set directly as part of the `bloc` via:

```dart
Bloc.observer = MyBlocObserver();
```

The following change was made to:

- Stay consistent with the new bloc observer API
- Keep the storage scoped to just `HydratedBloc`
- Decouple the `BlocObserver` from `Storage`

**v4.0.0**

```dart
BlocSupervisor.delegate = await HydratedBlocDelegate.build();
```

**v5.0.0**

```dart
HydratedBloc.storage = await HydratedStorage.build();
```

#### ❗Simplified Initialization

##### Rationale

Previously, developers had to manually call `super.initialState ?? DefaultInitialState()` in order to setup their `HydratedBloc` instances. This is clunky and verbose and also incompatible with the breaking changes to `initialState` in `bloc`. As a result, in v5.0.0 `HydratedBloc` initialization is identical to normal `Bloc` initialization.

**v4.0.0**

```dart
class CounterBloc extends HydratedBloc<CounterEvent, int> {
  @override
  int get initialState => super.initialState ?? 0;
}
```

**v5.0.0**

```dart
class CounterBloc extends HydratedBloc<CounterEvent, int> {
  CounterBloc() : super(0);

  ...
}
```
