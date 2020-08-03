---
layout: post
comments: true
title: Redux-like architecture with SwiftUI&#58; Side Effects, Middleware & Combine
image: /posts/2020-07-30-redux-like-architecture-with-swiftui-basics/header.png
tags:
  - development ios swift
---

In the [previous post]({{ site.url }}/redux-like-architecture-with-swiftui-basics/) we went through setting up the basic architecture for an iOS a Redux-like approach, and now we will be following up with additional optimizations and support for asynchronous functions or side effects.<!--more-->

So far, our app is aligned with Redux three principles:

* A single source of truth
* A read-only App State
* Mutating the state only through Reducer functions.

What happens if we want to load animal names from the network instead of a local array?

Sure, we could add all that logic and dependencies to our *appReducer*, and make it work for this small app, but it’s far from optimal and goes against Redux principles.

By definition, Reducers must be *pure functions*—functions that return the exact same output for given inputs, and should also be free of side effects like API calls or asynchronous tasks, so let's introduce the Middleware concept.

---

> This project is based on the code we wrote in the [previous post]({{ site.url }}/redux-like-architecture-with-swiftui-basics/).  Before starting, grab a clean copy from [this repository.](https://github.com/afterxleep/Redux-Architecture-SwiftUI-Basics)

---

## Middlewares
In Redux, a Middleware is an entity (function) that is executed when certain Actions go through the Store.   It receives a copy of the current AppState, performs some operations (API calls, or async tasks), and dispatches a new Action.

Middlewares take care of all the heavy load in the background (like API call, database fetches and updates, etc…) while Reducers simple deal with updating our state synchronously.  Let’s update the store to support Middlewares.

#### AppStore.swift
```swift
import Foundation

typealias Dispatcher<Action> = (Action) -> Void
typealias Middleware<State, Action> = (State, Action, @escaping Dispatcher<Action>) -> Void
typealias AppStore = Store<AppState, AppAction>

final class Store<State, Action>: ObservableObject {

    // Read only access to app state
    @Published private(set) var state: State

    private let reducer: Reducer<State, Action>
    let middlewares: [Middleware<State, Action>]

    init(initialState: State,
         reducer: @escaping Reducer<State, Action>,
         middlewares: [Middleware<State, Action>] = []) {
        self.state = initialState
        self.reducer = reducer
        self.middlewares = middlewares
    }

    // The dispatch function.
    func dispatch(_ action: Action) {
        reducer(&state, action)

        middlewares.forEach { middleware in
            middleware(state, action, dispatch)
        }
    }
}
```

I’ve created a couple new type aliases to define our Middleware and Dispatcher functions, and then added a Middleware array to our Store.  
This allows us to inject as many Middleware functions as we need on initialization.

Then, I’ve modified our dispatch function to iterate on the available Middlewares.   Each Middleware will receive a copy of our State, and action and the dispatch function.  Based on the action, it will then execute the tasks we need and dispatch a new action with the result.

So now, instead of generating animal names in our reducer, let’s go asynchronous and create a Service that simulates a network call, by generating an animal name after X(random) number of seconds.

#### AnimalService.swift
``` swift
import Foundation
import Combine

struct AnimalService {

    func generateAnimalInTheFuture() -> AnyPublisher<String, Never> {
        let animals = ["Cat", "Dog", "Crow", "Horse", "Iguana", "Cow", "Racoon"]
        let number = Double.random(in: 0..<5)
        return Future<String, Never> { promise in
            DispatchQueue.main.asyncAfter(deadline: .now() + number) {
                let result = animals.randomElement() ?? ""
                promise(.success(result))
            }
        }
        .eraseToAnyPublisher()
    }

}
```

Here we are using the Combine “[Future Publisher](https://developer.apple.com/documentation/combine/future)”  to create a promise, that is fulfilled after a random number of seconds with a random name.  As you can see, we are returning a publisher of type [AnyPublisher](https://developer.apple.com/documentation/combine/anypublisher) and not a String.    On completion, the Future publisher produces a single String value, and then finishes or fails.  In this case, it never fails, so we set it up as *<String, Never>*

By using AnyPublisher, instead of Publisher we wrap our publisher, so its details are not exposed, and also prevent its callers from accessing send(:) on it.     If you want to find out more about this, check out this [ Stack Overflow thread.](https://stackoverflow.com/questions/57690243/publisher-vs-anypublisher-in-combine)

Now that the Store is ready for async tasks, and we have our new service, let’s move forward with optimizing our Redux Stack and implementing our first Middleware.

## Let’s do some Optimizations
We could store everything in our AppState struct, and that works for a small app like this one, but as  apps grow you end up passing a huge amount of data around and things get out of hand.

#### AppState.swift
``` swift
import Foundation

struct AppState {
    var animal: AnimalState
}

struct AnimalState {
    var currentAnimal: String = ""
}
```

Here we are simply dividing our state into different pieces and composing them in AppState.   You can continue to add as many structs as needed to keep things clean and easy to understand.

Now let’s setup the Actions for the animal generator.   Since now we are fetching the data asynchronously, we’ll need two:  One that performs the fetch, and another that sets the result value in our State when the data arrives.

As we did with our State, let’s do some composition here too.

#### AppActions.swift
``` swift
import Foundation

enum AppAction {
    case animal(action: AnimalAction)
}

enum AnimalAction {
    case fetchAnimal
    case setCurrentAnimal(animal: String)
}
```

Following the same pattern, we can also extract the logic for our reducers.  

#### Reducers.swift
``` swift
import Foundation

typealias Reducer<State, Action> = (inout State, Action) -> Void

func appReducer(state: inout AppState, action: AppAction) -> Void {
    switch(action) {

    case .animal(let action):
        animalReducer(state: &state.animal, action: action)
    }

}

func animalReducer(state: inout AnimalState, action: AnimalAction) -> Void {
    switch(action) {
        case .fetchAnimal:
            break
        case .setCurrentAnimal(let animal):
            state.currentAnimal = animal
    }
}
```

With those changes in place we will keep things tidy and clean, when the app grows, and limit the data we pass around.     The app won’t compile as we have not updated our views, but let’s move onto the Middleware and we can update that later.

This is also a good time to reorganize our functions and Types into different files and groups.  Here’s an example that would work for most small to medium size apps.

[App groups]

## Implementing our Middleware
The animalMiddleware will receive a copy of the current State, an Action and a Dispatcher function.   We will be using the new service we’ve created, to fetch an animal, and then simply dispatch a new Action with the name when it becomes available.

#### AnimalMiddleware.swift
``` swift
import Foundation
import Combine

func animalMiddleware(service: AnimalService) -> Middleware<AppState, AppAction> {

    var cancellables: Set<AnyCancellable> = []

    return { state, action, dispatch in
        switch action {
            case .animal(action: .fetchAnimal):
                service.generateAnimalInTheFuture()
                    .sink { dispatch( .animal(action: .setCurrentAnimal(animal: $0))) }
                    .store(in: &cancellables)
            default:
                break
            }
        }

}
```

Pretty neat, huh?. Since we are using Combine, there is no need for callbacks or weird tricks to handle asynchronous operations and as our application grows, we can add more cases to our Middleware to intercept other actions.

Now let’s modify our Store initializer to inject our Middleware as a dependency.

#### ContentView.swift
``` swift
import SwiftUI

struct ContentView: View {
    let store = AppStore(initialState:
                            .init(animal: AnimalState()),
                      reducer: appReducer,
                      middlewares: [
                        animalMiddleware(service: AnimalService())
                      ])


    init() {
        store.dispatch(.animal(action: .fetchAnimal))
    }

    var body: some View {
        AnimalView()
            .environmentObject(store)
    }
}

struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
```

In essence, we’ve are injecting our new AnimalState and the Middleware in the initializer.  This approach makes our code really easy to test

We’ve also changed the dispatch call in our initializer to match the changes we did to our Store.

Now, let’s take care of updating the view.

#### AnimalView.swift
``` swift
import SwiftUI

struct AnimalView: View {
    @EnvironmentObject var store: AppStore

    func loadAnimal() {
        store.dispatch(.animal(action: .fetchAnimal))
    }

    var body: some View {
        VStack {
            Text(store.state.animal.currentAnimal).font(.system(.largeTitle)).padding()
            Button(“Tap me”, action: { self.loadAnimal() })
        }
    }
}
```

That’s it.  If you try build and run the app, it will launch, and after some seconds, a new Animal will appear on the screen, and the same will happen when you tap the button.

It’s all good, but the User Experience not so much.  It takes time to display an animal after you press the button, and you have no feedback on the screen.  So let’s update our UI to display a “Loading” message, while our animal is fetched.

The easiest way to do this, would be displaying a “Loading” message in the same label, while the animal name is being loaded, so let’s modify our animalReducer.

``` swift
import Foundation

func animalReducer(state: inout AnimalState, action: AnimalAction) -> Void {
    switch(action) {
        case .fetchAnimal:
            state.currentAnimal = "Loading..."
        case .setCurrentAnimal(let animal):
            state.currentAnimal = animal            
    }
}
```

As you can see, I’m just modifying our state when that actions arrives.  Now try and run the app again, and you should get something like this:

<img src="/assets/posts/2020-08-03-redux-like-architecture-with-swiftui-part2/result.gif" width="400">

## The Power of Middlewares
Having Middlewares in place, allows us to do all sorts of things in the application, as we can easily intercept any action that passes through our Store.

For example, creating a Middleware, that Logs every action to the console is as simple as writing a function.

``` swift
import Foundation

func logMiddleware() -> Middleware<AppState, AppAction> {

    return { state, action, dispatch in
        print(“Triggered action: \(action)”)
    }

}
```

And then simply add it to your store Initialization

``` swift
let store = AppStore(initialState: .init(
                            animal: AnimalState()
                        ),
                      reducer: appReducer,
                      middlewares: [
                        animalMiddleware(service: AnimalService()),
                        logMiddleware()
                      ])
```

Remember that every Middleware receives a full copy of you app’s state, and can trigger whatever action you need, so possibilities are limitless.

## Conclusion
We did a lot of things in this second part of the tutorial and now you have a really robust Architecture, that would allow you to write testable apps in no time.

Check out the resulting app for this post is [the repo](https://github.com/afterxleep/Redux-Architecture-SwiftUI-Part2)., where we will be talking making some improvements in the View Stack and implementing proper error handling, so our users can be informed if something has gone wrong.

I hope you enjoyed this tutorial.   If you have any questions or comments, feel free to ping me on [Twitter](https://twitter.com/afterxleep).

---
Posts in this series
1. [The Basics]({{ site.url }}/redux-like-architecture-with-swiftui-basics/)
2. [Side Effects, Middleware & Combine (This Post)]({{ site.url }}/redux-like-architecture-with-swiftui-middleware/)


---

Sources & Refs:
- [Three Principles - Redux](https://redux.js.org/introduction/three-principles)
- [Async Actions - Redux](https://redux.js.org/advanced/async-actions)
- [How to handle asynchronous operations with Redux in Swift](https://medium.com/swlh/how-to-handle-asynchronous-operations-with-redux-in-swift-495591d9df84)












