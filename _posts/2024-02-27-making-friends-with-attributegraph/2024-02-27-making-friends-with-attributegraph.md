---
layout: post
title: "Making Friends with AttributeGraph"
attribution: "I am grateful to <a href=\"https://shadowfacts.net\">Shadowfacts</a> for their review and feedback on an early draft of this post."
---

If you've used [SwiftUI](https://developer.apple.com/xcode/swiftui/) for long enough, you've probably noticed that the public Swift APIs it provides are really only half the story. Normally inconspicuous unless something goes *exceedingly* wrong, the private framework called AttributeGraph tracks almost every single aspect of your app from behind the scenes to make decisions on when things need to be updated. It would not be much of an exaggeration to suggest that this C++ library is actually what runs the show, with SwiftUI just being a thin veneer on top to draw some platform-appropriate controls and provide a stable interface to program against. True to its name, AttributeGraph provides the foundation of what a declarative UI framework needs: a graph of attributes that tracks data dependencies.

Mastering how these dependencies work is crucial to writing advanced SwiftUI code. Unfortunately, being a private implementation detail of a closed-source framework means that searching for AttributeGraph online usually only yields results from people desperate for help with their crashes. (Being deeply unpleasant to reverse-engineer definitely doesn't help things, though [some](https://www.fivestars.blog/articles/lets-build-state/) [have](https://medium.com/@kateinoigakukun/inside-swiftui-how-state-implemented-92a51c0cb5f6) [tried](https://kyleye.top/posts/demystify-attributegraph-1/).) Apple [has](https://developer.apple.com/videos/play/wwdc2020/10040) [several](https://developer.apple.com/videos/play/wwdc2021/10022) [videos](https://developer.apple.com/videos/play/wwdc2023/10160/) that go over the high-level design, but unsurprisingly they shy away from mentioning the existence of AttributeGraph itself. Other developers do, but [only fleetingly](https://chris.eidhof.nl/presentations/day-in-the-life/).

This puts us in a real bind! We can `Self._printChanges()` all day and still not understand what is going on, especially if problems we have relate to *missing* updates rather than too many of them. To be honest, figuring out what AttributeGraph is doing internally is not all that useful unless it is not working correctly. We aren't going to be calling those private APIs anyways, at least not easily, so there's not much point exploring them. What's more important is understanding what SwiftUI does and how the dependencies need to be set up to support that. We can take a leaf out of the generative AI playbook and go with the approach of just making guesses as how things are implemented. Unlike AI, we can also test our theories. We won't know whether our speculation is right, but we can definitely check to make sure we're not wrong!

{% include aside.html type="Warning" content="If it isn't clear already, what follows is conjecture about how a private implementation detail of SwiftUI works. It is, to my knowledge, accurate; however, it is extremely susceptible to deducing implementation details. Be cautious before relying on any of this information!" %}

## Introducing `Setting`

As we explore how SwiftUI propagates changes, it will be very helpful to have an real example to work with. Say hello to `Setting`, a simple `UserDefaults` wrapper similar to [`AppStorage`](https://developer.apple.com/documentation/swiftui/appstorage):

```swift
@propertyWrapper
struct Setting<T> {
	init(_ key: String, defaultValue: T)
	var wrappedValue: T { get set }
	var projectedValue: Binding<T> { get }
	var isSet: Bool { get }
	func reset()
}
```

We haven't implemented it yet, but it should be fairly straightforward to see how you might use this:

<!-- I should really send a PR to Rouge shouldn't I -->

```swift
@Setting("favoriteNumber", defaultValue: 42)
var favoriteNumber

favoriteNumber = 69
print(favoriteNumber) // 69
print($favoriteNumber.isSet) // true
$favoriteNumber.reset()
print($favoriteNumber.isSet) // false
print(favoriteNumber) // 42
```

Our API is slightly more explicit about how unset values should work, but it is otherwise almost identical to Apple's: we even expose a `Binding` as the `projectedValue` so `Setting` can be used directly with SwiftUI controls. Here's what a preliminary implementation might look like. None of it should be particularly surprising:

```swift
@propertyWrapper
struct Setting<T> {
	let key: String
	let defaultValue: T

	init(_ key: String, defaultValue: T) {
		self.key = key
		self.defaultValue = defaultValue
	}

	var wrappedValue: T {
		get {
			UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
		}
		set {
			UserDefaults.standard.setValue(newValue, forKey: key)
		}
	}

	var projectedValue: Binding<T> {
		Binding(
			get: {
				wrappedValue
			},
			set: {
				wrappedValue = $0
			}
		)
	}

	var isSet: Bool {
		UserDefaults.standard.value(forKey: key) != nil
	}

	func reset() {
		UserDefaults.standard.removeObject(forKey: key)
	}
}
```

This *almost* works. However, we have a compiler error in our implementation for `projectedValue`–it captures `self`, and when the `Binding` updates it tries to call `wrappedValue`'s setter. But since `self` is immutable, the compiler will not let us modify `wrappedValue`. Or will it?

We're not really modifying `self` here: the setter just writes to user defaults. In particular, it doesn't touch any members of the struct at all, so `self` doesn't change. Swift has a special way to accommodate this: a `nonmutating` setter. This is actually what `State` uses itself to let you assign to it, even in contexts where the view that owns it is not mutable: it stores its state outside of itself, just like we do. Let's update our implementation of `wrappedValue` to use this.

```swift
var wrappedValue: T {
	get {
		UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
	}
	nonmutating set {
		UserDefaults.standard.setValue(newValue, forKey: key)
	}
}
```

With that, the code compiles, so let's take it for a spin:

```swift
struct ContentView: View {
	@Setting("enabled", defaultValue: false)
	var enabled

	var body: some View {
		Toggle("Item", isOn: $enabled)
	}
}
```

## Handling updates

If you copied the code above into a test project of your own, you might have noticed that it doesn't exactly seem to work as we'd hope. You can toggle the switch all you want, but the UI doesn't update. What's going on? We made a `Binding` and everything!

With a debugger, it's easy to verify that the `Toggle` *is* actually invoking the `Binding` callback as it should. When we flip the switch, it calls through the set implementation, and from there to our `wrappedValue` setter, which writes to user defaults. In fact if we relaunch the app the UI will read the correct state that has been being saved in user defaults all this time, so the problem is in how SwiftUI updates the view rather than the backing store for the value. Since we have a debugger attached, we can see that the getter does get called a few times, which must be how the framework sets the initial state of the view. After that, though, the UI does not update, even though the getter returns the new value. Notably, `ContentView`'s body is not evaluated again, even though it should because of the new state.

### Examining `State`

If we replace `Setting` in our sample code with `State`, things work as expected. This is entirely unsurprising, because this is how we're *supposed* to do things. Somehow SwiftUI knows when `State`s change and triggers a view update in response. To do this, it must somehow be instrumenting stores to the underlying value…wait, this is exactly what property wrappers are for! When you call the setter you get a chance to run your own code, and `State` must use it to tell SwiftUI about the change. Putting aside our earlier conversation about `nonmutating`, `State` probably looks something like this:

```swift
struct State<Value> {
	var _value: Value

	var wrappedValue: Value {
		get {
			_value
		}
		set {
			_value = newValue
			SwiftUI._noteChanges(self)
		}
	}
}
```

Even though this explains why our code doesn't work, this is still a problem for us, because *we* can't call this method ourselves. It's internal to SwiftUI, and only `State` (and the other built-in types) know how to do it. However, we don't have to. The framework provides an affordance to let solve our issue: composing property wrappers.

### Composition with `DynamicProperty`

Since only built-in types know how to update the UI, SwiftUI allows us to extend this system with *composition*. If a property wrapper contains a `State` inside of itself, then mutating it can initiate a refresh. Let's make a few changes to the property wrapper (and skip the parts that remain unmodified):

```swift
@propertyWrapper
struct Setting<T> {
	let key: String
	let defaultValue: T
	
	@State
	var value: T?

	init(_ key: String, defaultValue: T) {
		self.key = key
		self.defaultValue = defaultValue
		self.value = UserDefaults.standard.object(forKey: key) as? T
	}

	var wrappedValue: T {
		get {
			value ?? defaultValue
		}
		nonmutating set {
			value = newValue
			UserDefaults.standard.setValue(newValue, forKey: key)
		}
	}
}
```

Note the addition of `value`, the new `State<T>` variable that we return instead. In the `wrappedValue` setter, we write to user defaults as before, but we also update `value`, which (as a `State`) can go and trigger a view refresh. Or, it would, but we're not quite done yet. SwiftUI doesn't know about the `State` we put inside our property wrapper yet: to let it peer inside and hook things up we need to conform `Setting` to the `DynamicProperty` protocol. We'll look into why this needs to be the case a little later, but with the change this code finally works. Toggling the switch updates the UI and it also writes to defaults, and this value persists across launches. Success!

### Abusing dependencies

Even though our design works, it's a little…unpleasant? The "source of truth" is in two places: in the `State` variable `value`, and in user defaults. `UserDefaults` is plenty fast and doesn't need us to cache values on its behalf. The old design we had, where defaults was the backing store and we talked to it directly, was definitely cleaner. If the `State` setter's side effects are all we need, can we clean the code up a little bit? What if we did this?

```swift
@propertyWrapper
struct Setting<T>: DynamicProperty {
	let key: String
	let defaultValue: T
	
	@State
	var value: T? = nil

	init(_ key: String, defaultValue: T) {
		self.key = key
		self.defaultValue = defaultValue
	}

	var wrappedValue: T {
		get {
			UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
		}
		nonmutating set {
			value = newValue
			UserDefaults.standard.setValue(newValue, forKey: key)
		}
	}
}
```

Even though we store things in `value` to trigger updates, we don't use it as the source of truth anymore. Instead, we just return whatever user defaults has in it. In the last version we were careful to have it match `value` at all times, so this should produce the same value.

If you try out this version, you'll find that it doesn't work anymore! The reason for this is actually quite subtle. If you've been following along so far and want a quick puzzle, see if you can debug what is going on before moving on to the next section. Hint: try printing `value` in `wrappedValue`'s getter.

---

The hint I gave was slightly misleading. Yes, printing `value` in `wrappedValue`'s getter does tell you that its value matches what is in user defaults (well, except until it is set for the first time, but you can add that code back in and check that it doesn't matter). But more importantly, **adding the print statement makes it work again!** If you remove the print statement, it stops working, and if you add it back, it works again.

If you test a little bit more, you'll see that it's not the print statement that's important, but the access of `value` itself. Even this code works:

```swift
get {
	_ = value
	return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
}
```

This is not a fluke of the optimizer. `State.wrappedValue`'s getter actually does something special: it returns the value, but it also notes to SwiftUI that the value has been "read". This is actually how SwiftUI optimizes view updates to only change parts of the UI that matter. When invoking a view's body, it must keep a list of all the state that is read during the pass. If any of the state changes in the future, it knows which views to update based on which ones access that state, and it will skip redrawing views that are unrelated. For us, this means that we cannot just set `value` in our setter: we also have to access it in the getter, to indicate to SwiftUI that we depend on it.

It's important to note at this point that this mechanism, while clever, is also somewhat crude. SwiftUI has no idea *what* we are doing with the value, only that we have accessed it. It cannot, because we can perform arbitrary computation with it to generate all of our "derived" properties. In fact the type of the `State` we update does not have to match the eventual dependent value (you can imagine a legitimate case where the `wrappedValue` was, say, the `description` of the `State`'s value). And we don't even have to do anything with the value we access. It's an access for the side effects of the access, and an update for the side effects of the update. This, you could argue, is a lot worse than our proper `State`-based solution, but it's far more flexible. Here's what it looks like (again, skipping redundant parts):

```swift
@propertyWrapper
struct Setting<T>: DynamicProperty {
	// Dummy state that SwiftUI thinks we depend on
	@State
	var _update = false

	var wrappedValue: T {
		get {
			_ = _update
			return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
		}
		nonmutating set {
			_update.toggle()
			UserDefaults.standard.setValue(newValue, forKey: key)
		}
	}
}
```

{% include aside.html content="We chose a `State<Bool>` to update because it is small and easy to update. We do have to be a little careful here, though: if we try to be even cleverer by using `Void`, then SwiftUI actually gets out ahead of us and ignores our set because the new value we use (`()`) is equal to the old one and discards the update. In theory SwiftUI could get more \"memory\" and cache both the `true` and `false` states for `Bool` and prevent future updates, but we can work around this by using an `Int` and incrementing it each time instead (like a sequence number)." %}

## `State` lifecycle

Our `Setting` is pretty neat, but it's missing an important feature: user defaults can be updated externally, not just from the changes we make in the app. `UserDefaults` is [KVO compliant](https://developer.apple.com/documentation/foundation/userdefaults#2926902) to allow us to respond to these changes. Since we know how to set up dependencies ourselves, updating our code to handle this isn't too hard:

```swift
@propertyWrapper
struct Setting<T>: DynamicProperty {
	class Observer: NSObject {
		let key: String
		let _update: State<Bool>

		init(key: String, _update: State<Bool>) {
			self.key = key
			self._update = _update
			super.init()
			UserDefaults.standard.addObserver(self, forKeyPath: key, context: nil)
		}
		
		deinit {
			UserDefaults.standard.removeObserver(self, forKeyPath: key)
		}

		override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey: Any]?, context: UnsafeMutableRawPointer?) {
			_update.wrappedValue.toggle()
		}
	}
	
	let observer: Observer

	init(_ key: String, defaultValue: T) {
		self.key = key
		self.defaultValue = defaultValue
		self.observer = Observer(key: key, _update: __update)
	}
}
```

Unfortunately, we can't use Swift's type-safe `KeyPath`-based callback API because `key` is a string, so we need a bit more boilerplate involving a `NSObject` subclass. We can pass it the `State` that we're using to manage updates and it can inform SwiftUI using that when the value changes. Otherwise, everything else can stay the same.

As you've probably guessed by now, this doesn't work. At least it doesn't break in-app behavior this time: it just doesn't update when changes happen externally. While the KVO callback does get invoked, poking `_update` from inside it does not have any visible effects.

### Reflection introspection

Even though `State` looks opaque to us, it must maintain some private data inside of itself. At the very least, it must have some sort of identity: even though it's a struct, it manages data that has a lifetime longer than any particular `State` instance. This state (pun unintended) is not accessible to us, but holds the key to the behavior we are seeing. Even though we are trying to update the "same" `State` in our code, something must be different about the two for one change to go through and another to get dropped.

Fortunately, Swift binaries typically contain fairly rich metadata that can help expose these internals for us. `Mirror` uses this to back its reflection APIs, but the debugger also knows how to read it, allowing us to poke around at the innards of types we don't own. Let's try that by printing `__update` (note the extra leading underscore to refer to the property wrapper itself) in `wrappedValue`, where we know it is ready to publish our changes:

```console
(lldb) po __update
▿ State<Bool>
- _value : false
▿ _location : Optional<AnyLocation<Bool>>
  ▿ some : <StoredLocation<Bool>: 0x600002f88c00>
```

As expected, `State` is a composite of several properties. `_value` is fairly self-explanatory, and we guessed its existence earlier. However, the other property, `_location`, is a little more enigmatic. We can drill down into it further using LLDB (for example, we can see that there is a flag called `_wasRead`, as well as a connection to AttributeGraph), but before dive too deeply let's do the same dump in `Observer.observeValue(forKeyPath:of:change:context:)`:

```console
(lldb) po _update
▿ State<Bool>
  - _value : false
  - _location : nil
```

Aha! This time, `_location` is `nil`. This `_location` must (among other things) contain the connection back to SwiftUI used when notifying it of changes. Since it's not set at this point, there's nobody there to listen to our updates.

It's worth thinking about how this might happen. Even though both `Setting` and `Observer` are using the "same" `State`, they actually don't get a coherent view of `_update`. `State`s are structs as mentioned earlier, so the `Observer` gets a *copy* of the state when it is initialized, which is when `Setting` is initialized. If we put a breakpoint in that initializer, we can see that `_location` *is* `nil` at that point. `Observer`'s own local copy is made then, and never sees any changes. On the other hand, `Setting`'s `_update` also has a `nil` `_location` at this point, but sometime in the future this changes to something valid. Who is doing it? When, and how?

The answer, of course, is that SwiftUI does it for us. Shortly before a view's body is computed, it goes through and sets the `_location` on all relevant `State` variables, so that they are ready for dependency tracking. Typically, a property wrapper does not have the ability to grab context from outside of itself (for example, by looking up who owns it). SwiftUI can use reflection much like we did to discover `State` members that it needs to install `_location` on, sidestepping this issue. To discover `State` in nested types, it needs a little bit of help: this is why we had to add a `DynamicProperty` conformance earlier. In that case, it uses reflection to look for `DynamicProperty` members instead and then does a search for `State` inside of those.

SwiftUI does the same inside of our `Setting` property wrapper when the view that owns it goes for an update. This means that we need `Observer._update` to be set then, rather than in the initializer. Fortunately, the framework calls [`DynamicProperty.update()`](https://developer.apple.com/documentation/swiftui/dynamicproperty/update()-29hjo) at exactly the right point in time to let us do this. Let's update `Setting` to use it:


```swift
@propertyWrapper
struct Setting<T>: DynamicProperty {
	let key: String
	let defaultValue: T

	@State
	var _update = false

	class Observer: NSObject {
		let key: String
		var _update: State<Bool>!

		init(key: String) {
			self.key = key
			super.init()
			UserDefaults.standard.addObserver(self, forKeyPath: key, context: nil)
		}
		
		deinit {
			UserDefaults.standard.removeObserver(self, forKeyPath: key)
		}

		override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey: Any]?, context: UnsafeMutableRawPointer?) {
			_update.wrappedValue.toggle()
		}
	}
	
	let observer: Observer

	init(_ key: String, defaultValue: T) {
		self.key = key
		self.defaultValue = defaultValue
		self.observer = Observer(key: key)
	}
	
	func update() {
		observer._update = __update
	}
}
```

With that, external updates (e.g. using `defaults write`) update the view as well. With that, our `Setting` is functional and ready to use with SwiftUI. Here's the final version that we created, for reference:

{% capture preference_code %}
```swift
@propertyWrapper
struct Setting<T>: DynamicProperty {
	let key: String
	let defaultValue: T

	@State
	var _update = false

	class Observer: NSObject {
		let key: String
		var _update: State<Bool>!

		init(key: String) {
			self.key = key
			super.init()
			UserDefaults.standard.addObserver(self, forKeyPath: key, context: nil)
		}
		
		deinit {
			UserDefaults.standard.removeObserver(self, forKeyPath: key)
		}

		override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey: Any]?, context: UnsafeMutableRawPointer?) {
			_update.wrappedValue.toggle()
		}
	}
	
	let observer: Observer

	init(_ key: String, defaultValue: T) {
		self.key = key
		self.defaultValue = defaultValue
		self.observer = Observer(key: key)
	}

	var wrappedValue: T {
		get {
			_ = _update
			return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
		}
		nonmutating set {
			_update.toggle()
			UserDefaults.standard.setValue(newValue, forKey: key)
		}
	}

	var projectedValue: Binding<T> {
		Binding(
			get: {
				wrappedValue
			},
			set: {
				wrappedValue = $0
			}
		)
	}

	var isSet: Bool {
		UserDefaults.standard.value(forKey: key) != nil
	}

	func reset() {
		UserDefaults.standard.removeObject(forKey: key)
	}
	
	func update() {
		observer._update = __update
	}
}
```
{% endcapture %}

{% include aside.html type="`Preference.swift`" collapsible=true content=preference_code %}

While it is probably not worth using this code directly in your app (note the disclaimer above!), understanding how and why it works the way it does might be helpful in debugging your own SwiftUI projects.
