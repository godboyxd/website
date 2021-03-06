In interactive UI applications, state is continually changing in response to user actions and application events. ReactiveUI enables you to express changes to application state as streams of values and combine and manipulate them using the powerful Reactive Extensions library.

The motivation is intuitive enough when you think about it. It's not hard to imagine that changes to a property can be considered events - that's how `INotifyPropertyChanged` works. From there, the same argument for using Rx over events applies. In the context of MVVM application design specifically, modelling property changes as observables leads to several advantages:

- The logic of an application can be defined in terms of changes to properties
- This logic can be composed and expressed declaratively, using the power of Rx operators
- Concepts like time and asynchronicity become easier to reason about, due to their first-class treatment in an Observable context.

ReactiveUI provides several variants of `WhenAny` to help you work with properties as an observable stream.

# What is WhenAny

`WhenAny` is a set of extension methods starting the prefix `WhenAny` that allow you to get notifications when property on objects change.

You can think of the `WhenAny` set of extension methods notifying you when one or many property values have changed.

`WhenAny` supports multiple property types including `INotifyPropertyChanged`, `DependencyProperty` and `BindableProperty`. 

It will check the property for support for each of those property types, and when you `Subscribe()` it will subscribe to the events offered by the applicable property notification mechanism.

`WhenAny` by default is just a wrapper around these property notification events, and won't store any values before a `Subscribe`. You can use techniques such as `Publish`, `Replay()` to get it to store these values.

You can also wrap the `WhenAny` in a `Observable.Defer` to avoid the value being calculated until a `Subscribe` has happened. This is useful for `ObservableAsPropertyHelper` when you using the defer feature.

# Basic syntax

The following examples demonstrate simple uses of `WhenAnyValue`, the `WhenAny` variant you are likely to use most frequently.

### Watching single property

This returns an observable that yields the current value of Foo each time it changes:

```cs
this.WhenAnyValue(x => x.Foo)
```

### Watching a number of properties

This returns an observable that yields a new `Color` with the latest RGB values each time any of the properties change. The final parameter is a selector describing how to combine the three observed properties:

```cs
this.WhenAnyValue(x => x.Red, x => x.Green, x => x.Blue, 
                  (r, g, b) => new Color(r, g, b));
```

### Watching a nested property

`WhenAny` variants can observe nested properties for changes, too:

```cs
this.WhenAnyValue(x => x.Foo.Bar.Baz);
```

# Idiomatic usage

Naturally, once you have an observable of property changes you can `Subscribe` to it in order to perform actions in response to the changed values. However, in many cases there may be a Better Way to achieve what you want. Below are some typical usages of the observables returned by the `WhenAny` variants:

### Exposing 'calculated' properties

In general, using `Subscribe` on a `WhenAny` observable (or any observable, for that matter) just to set a property is likely a code smell. Idiomatically, the `ToProperty` operator is used to create a 'read-only' calculated property that can be exposed to the rest of your application, only settable by the `WhenAny` chain that preceded it:

```cs
this.WhenAnyValue(x => x.SearchText, x => x.Length)
    .ToProperty(this, x => x.SearchTextLength, out _searchTextLength);
```

This initialises the `SearchTextLength` property (an [ObservableAsPropertyHelper](../oaph/) property) as a property that will be updated with the current search text length every time it changes. The property cannot be set in any other manner and raises change notifications, so can itself be used in a `WhenAny` expression or a binding. 

See the [ObservableAsPropertyHelper](../oaph/) section for more information on this pattern.

### Supporting validation as a `CanExecute` criteria

`WhenAny` can make specifying and adhering to validation logic clean and simple. Here, `WhenAnyValue` is used to observe the changing values of the `Username` and `Password` fields, and project whether the current pair of values is valid. This becomes the `canExecute` parameter for `CreateUserCommand`, preventing the user from proceeding until the validation conditions are met.

```cs
var canCreateUser = this.WhenAnyValue(
    x => x.Username, x => x.Password, 
    (user, pass) => 
        !string.IsNullOrWhiteSpace(user) && 
        !string.IsNullOrWhiteSpace(pass) && 
        user.Length >= 3 && 
        pass.Length >= 8)
    .DistinctUntilChanged();

CreateUserCommand = ReactiveCommand.CreateFromTask(CreateUser, canCreateUser); 
```

### Invoking commands

[Commands](./commands) are often bound to buttons or controls in the view that can be triggered by the user. However, it often makes sense to perform work in response to changes in property values. For example, a 'live search' feature may be designed to perform searches as the user types into a textbox, after a small delay is detected. `WhenAny` in conjunction with the `InvokeCommand` operator can be used to achieve this.

```cs
// In the ViewModel.
this.WhenAnyValue(x => x.SearchText)
    .Where(x => !String.IsNullOrWhiteSpace(x))
    .Throttle(TimeSpan.FromSeconds(.25))
    .InvokeCommand(SearchCommand);

// In the View.
this.Bind(ViewModel, vm => vm.SearchText, v => v.SearchTextField.Text);
```

In addition to being able to simply and declaratively handle search throttling, building the search execution logic on top of the property change has made it easy to keep all the logic in the viewmodel - all the view needs to do is bind a control to the property.

### Performing view-specific transforms as an input to `BindTo`

Ideally, controls on your view bind directly to properties on your viewmodel. In cases where you need to convert a viewmodel value to a view-specific value (e.g. `bool` to `Visibility`), you should register a `BindingConverter`. Still, you may come across a situation in which you want to perform a transformation in the view directly. Here, we observe the `ShowToolTip` property of the viewmodel, transform the `true`/`false` values to `1` and `0` respectively, then bind the result to the `ToolTipLabel`'s alpha property. 

```cs
// In the View.
ViewModel.WhenAnyValue(x => x.ShowToolTip)
         .Select(show => show ? 1f : 0f)
         .BindTo(this, x => x.ToolTipLabel.Alpha);
```

# Variants of `WhenAny`
 
Several variants of `WhenAny` exist, suited for different scenarios.

### WhenAny vs WhenAnyValue

`WhenAnyValue` covers the most common usage of `WhenAny`, and is a useful shortcut in many cases. The following two statements are equivalent and return an observable that yields the updated value of `SearchText` on every change:

- `this.WhenAny(x => x.SearchText, x => x.Value)`
- `this.WhenAnyValue(x => x.SearchText)`

When needing to observe one or many properties for changes, `WhenAnyValue` is quick to type and results in simpler looking code. Working with `WhenAny` directly gives you access to the `ObservedChange<,>` object that ReactiveUI produces on each property change. This is typically useful for framework code or extension methods. `ObservedChange` exposes the following properties:

* `Value` - the updated value
* `Sender` - the object whose has property changed 
* `Expression` - the expression that changed.

At the risk of extreme repetition - use `WhenAnyValue` unless you know you need `WhenAny`. 

### WhenAnyObservable

`WhenAnyObservable` acts a lot like the Rx operator `CombineLatest`, in that it watches one or multiple observables and allows you to define a projection based on the latest value from each. `WhenAnyObservable` differs from `CombineLatest` in that its parameters are expressions, rather than direct references to the target observables. The impact of this difference is that the watch set up by `WhenAnyObservable` is not tied to the specific observable instances present at the time of subscription. That is, the observable pointed to by the expression can be replaced later, and the results of the new observable will still be captured. 

An example of where this can come in handy is when a view wants to observe an observable on a viewmodel, but the viewmodel can be replaced during the view's lifetime. Rather than needing to resubscribe to the target observable after every change of viewmodel, you can use `WhenAnyObservable` to specify the 'path' to watch. This allows you to use a single subscription in the view, regardless of the life of the target viewmodel. 

# Additional Considerations

Using `WhenAny` variants is fairly straightforward. However, there are a few aspects of their behaviour that are worth highlighting.

### `INotifyPropertyChanged` is required

Watched properties must implement ReactiveUI's `RaiseAndSetIfChanged` or raise the standard `INotifyPropertyChanged` events. If you attempt to use `WhenAny` on a property without either of these in place, `WhenAny` will produce the current value of the property upon subscription, and nothing thereafter. Additionally, a warning will be issued at run time (ensure you have registered a service for `ILogger` to see this).

### `WhenAny` has cold observable and behavioural semantics

`WhenAny` is a purely cold Observable, which eventually directly connects to UI component events. For events such as `DependencyProperties`, this could potentially be a minor place to optimize, via `Publish`. When chaining to `ToProperty` (another cold operator), the target `ObservableAsPropertyHelper` must be read (`.Value`) or observed (e.g. used in a binding or used as part of another `WhenAny` with a subscription), for any part of the chain to execute. 

Additionally, `WhenAny` always provides you with the current value as soon as you subscribe to it - in this sense it is effectively a `BehaviorSubject`.

### `WhenAny` will not propagate `NullReferenceException`s within the watched expression

`WhenAny` will only send notifications if reading the given expression would not throw a `NullReferenceException`. Consider the following code:

```cs
this.WhenAny(x => x.Foo.Bar.Baz, _ => "Hello!")
    .Subscribe(x => Console.WriteLine(x));

// Example 1
this.Foo.Bar.Baz = null;
>>> Hello!

// Example 2: Nothing printed!
this.Foo.Bar = null;

// Example 3
this.Foo.Bar = new Bar() { Baz = "Something" };
>>> Hello!
```

* In Example 1, even though `Baz` is null, because the expression could be evaluated, you get a notification.

* In Example 2 however, evaluating this.Foo.Bar.Baz wouldn't give you null, it would crash. `WhenAny` therefore suppresses any notifications from being generated. Setting `Bar` to a new value generates a new notification.

### `WhenAny` only notifies on change of the output value

`WhenAny` only tells you when the final value of the input expression has changed. This is true even if the resulting change is because of an intermediate value in the expression chain. Here's an explaining example:

```cs
this.WhenAny(x => x.Foo.Bar.Baz, _ => "Hello!")
    .Subscribe(x => Console.WriteLine(x));

// Example 1
this.Foo.Bar.Baz = "Something";
>>> Hello!

// Example 2: Nothing printed!
this.Foo.Bar.Baz = "Something";

// Example 3: Still nothing
this.Foo.Bar = new Bar() { Baz = "Something" };

// Example 4: The result changes, so we print
this.Foo.Bar = new Bar() { Baz = "Else" };
>>> Hello!
```

Notably, in Example 3, even though the intermediate `Bar` object was replaced with a new instance, no change is fired - as the result of the full `Foo.Bar.Baz` expression has not changed.

### null propogation inside `WhenAnyValue`
`WhenAnyValue` is not directly able to perform null propogation due to the fact Expression's don't support this feature yet.

You can simulate null propogation support by chaining your WhenAnyValue() call to each property along the way. Here's an example:

```cs
this.WhenAnyValue(x => x.Foo, x => x.Foo.Bar, x => x.Foo.Bar.Baz, (foo, bar, baz) => foo?.Bar?.Baz)
    .Subscribe(x => Console.WriteLine(x));
```

# How WhenAny knows about different type of properties

`WhenAny` operators will look for services registered with Splat with the `ICreatesObservableForProperty` interface.

The interface has two methods, `GetAffinityForObject` and `GetNotificationForProperty`.

The first method `GetAffinityForObject` based on the property type, property name, and it should notify before or after the property change, will ask for a "vote" on how confident the property changed observable is at converting the value. A value of 0 from `GetAffinityForObject` means it cannot create a property changed observable at all. The system will then take votes from all registered `ICreatesObservableForProperty` interfaces and the one with the highest numeric vote wins. In the worst case a `POCOObservableForProperty` will always have a value greater than 0. This type of property changed observable will only get the initial value for the property and never update.

After the voting has finished it will call `GetNotificationForProperty` with the property type, name and if it should be before or after the property changed. It will then create the property changed observable.

When you are calling as part of a `WhenAny` chain it will get the property changed observable for each property along the chain.

So for instance `this.WhenAnyValue(x => x.Property1.Property2)` will get the property changed observables for the Property1, then for the Property2. So each can be a different type of observable property, for example a `INotifyPropertyChanged` and a `DependencyObject`.
