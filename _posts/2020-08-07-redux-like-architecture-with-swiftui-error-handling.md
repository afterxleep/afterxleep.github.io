---
layout: post
comments: true
title: Redux-like architecture with SwiftUI&#58; Error Handling
image: /posts/2020-08-07-redux-like-architecture-with-swiftui-error-handling/header.png
tags:
  - development ios swift
---

In the [previous post]({{ site.url }}/redux-like-architecture-with-swiftui-middleware/), we’ve implemented Middleware support to facilitate asynchronous operations and extended functionality, and today we will be following up with Error handling and improving our User Experience<!--more-->.

 >This project is based on the code we wrote in the [Side Effects post]({{ site.url }}/redux-like-architecture-with-swiftui-middleware/).  Before starting, grab a clean copy from [this repository.](https://github.com/afterxleep/Redux-Architecture-SwiftUI-SideEffects)

## Handling errors in our Middleware Functions
Let’s say, for example, a user taps the button, and the AnimalMiddleware tries to fetch an animal from the service, but in this case, it gets an error.  

When this happens, the UI will freeze, displaying the “Loading” message, and the user will be stuck waiting for a response.  

To fix things, we will break them first, so let's start by updating our AnimalService to fail randomly.

The first thing we need to do is define an enum that will hold the Error Types the service can return.  Then, update our Publisher declaration to return a Failure by replacing *<String, Never>* with *<String, AnimalServiceError>*

Then, we will add some simple logic to return errors at random.

#### AnimalService.swift
```swift
import Foundation
import Combine

enum AnimalServiceError: Error, CaseIterable {
    case unknown
    case networkError
}

struct AnimalService {

    var requestNumber: Int = 0

    func generateAnimalInTheFuture() -> AnyPublisher<String, AnimalServiceError> {
        let animals = ["Cat", "Dog", "Crow", "Horse", "Iguana", "Cow", "Racoon"]
        let number = Double.random(in: 0..<5)
        return Future<String, AnimalServiceError> { promise in
            DispatchQueue.main.asyncAfter(deadline: .now() + number) {
                let randomError = Int.random(in: 0..<2)
                if(randomError != 0) {
                    promise(.success( animals.randomElement() ?? ""))
                }
                promise(.failure(AnimalServiceError.allCases.randomElement()!))
            }
        }
        .eraseToAnyPublisher()
    }

}
```

## Setting up App State and Actions
Now that the service is ready, we need to set up our App State to maintain information about fetching Animals and tracking errors.

We will also take this opportunity to streamline some of the State Variables' names and actions for clarity.  This seems like changing a lot, but it's actually just changing variable names across existing files.  The compiler will guide you if you missed something.

Let’s update the AppState first:

#### AppState.swift
``` swift
import Foundation

struct AppState {
    var animal: AnimalState
}

struct AnimalState {
    var current: String = ""
    var fetchError: String?
    var fetchInProgress: Bool = false
}
```

And since we’ve added new State variables, we would also need to create the corresponding Actions.  

#### AnimalAction.swift
``` swift
import Foundation

enum AnimalAction {
    case fetch
    case fetchComplete(animal: String)
    case fetchError(error: Error?)
}
```

Then change the switch statement in the animalReducer:

#### AnimalReducer.swift
```swift
func animalReducer(state: inout AnimalState, action: AnimalAction) -> Void {

switch(action) {
        case .fetch:
            state.fetchError = nil
            state.fetchInProgress = true

        case .fetchComplete(let animal):
            state.fetchError = nil
            state.fetchInProgress = false
            state.current = animal

        case .fetchError(let error):
            state.fetchError = error
            state.fetchInProgress = false
    }
}
```

And the actions we are publishing at the Middleware:

#### AnimalMiddleware.swift
``` swift
func animalMiddleware(service: AnimalService) -> Middleware<AppState, AppAction> {

    return { state, action in
        switch action {

            case .animal(.fetch):
                return service.generateAnimalInTheFuture()
                    .subscribe(on: DispatchQueue.main)
                    .map { AppAction.animal(action: .fetchComplete(animal: $0 )) }                   
                    .eraseToAnyPublisher()

            default:                        
                break
            }

        return Empty().eraseToAnyPublisher()
    }
}
```

And lastly, the actions names we are dispatching from the Views.

#### AnimalView.swift
``` swift
import SwiftUI

struct AnimalView: View {
    @EnvironmentObject var store: AppStore

    func loadAnimal() {
        store.dispatch(.animal(action: .fetch))
    }

    var body: some View {
        VStack {
            Text(store.state.animal.current).font(.system(.largeTitle)).padding()
            Button("Tap me", action: { self.loadAnimal() })
        }
    }
}
```

#### ContentView.swift
``` swift
import SwiftUI

struct ContentView: View {
    let store = AppStore(initialState: .init(
                            animal: AnimalState()
                        ),
                      reducer: appReducer,
                      middlewares: [
                        animalMiddleware(service: AnimalService()),
                        logMiddleware()
                      ])

    init() {
        store.dispatch(.animal(action: .fetch))
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


## Catching errors with Combine
When using Combine, you are probably running lots of asynchronous code, and therefore you will need ways to handle and catch errors when they appear.  

Combine offers some operators to handle errors returned from upstream publishers, allowing you to recover from them.  Let’s try a couple!.

### .replaceError()
This operator is the easiest of all.  Every publisher that can fail (as our AnimalService Publisher) allows you to replace any error that occurs with a default value.   

Let's use *[replaceError](https://developer.apple.com/documentation/combine/publisher/replaceerror(with:))*, to set up a Default Action:

#### AnimalMiddleware.swift
``` swift
...

 case .animal(.fetchAnimal):
                return service.generateAnimalInTheFuture()
                    .subscribe(on: DispatchQueue.main)
                    .map { AppAction.animal(action: .setCurrentAnimal(animal: $0 )) }
                    .replaceError(with: AppAction.animal(action: .setCurrentAnimal(animal: "Oops!")))
                    .eraseToAnyPublisher()

```

As the *map* operator fails, we will return a new Action that has a default value of “Oops,” which will cause that text to render in the Label.

<img src="/assets/posts/2020-08-07-redux-like-architecture-with-swiftui-error-handling/result1.gif" width="400">

### .catch()
Since the AnimalService can fail with different types of errors, we might want to show different information to the user. Using *replaceError* is not enough.  

[Catch](https://developer.apple.com/documentation/combine/publishers/trydropwhile/catch(_:)) allows you to inspect the received error and return a new Publisher that will be passed downstream.

Let's update the Middleware to use *.catch().*  

``` swift
func animalMiddleware(service: AnimalService) -> Middleware<AppState, AppAction> {

    return { state, action in
        switch action {

        case .animal(.fetch):
                return service.generateAnimalInTheFuture()
                    .subscribe(on: DispatchQueue.main)
                    .map { AppAction.animal(action: .fetchComplete(animal: $0 )) }
                    .catch { (error: AnimalServiceError) -> Just<AppAction> in
                        switch(error) {
                        case .unknown:
                            return Just(AppAction.animal(action: .fetchComplete(animal: "Oops")))
                        case .networkError:
                            return Just(AppAction.animal(action: .fetchComplete(animal: "Network Failed")))
                        }
                    }
                    .eraseToAnyPublisher()

            default:

                break
            }

        return Empty().eraseToAnyPublisher()
    }
}
```

Note that we are returning a Publisher ([Just](https://www.google.com/search?client=safari&rls=en&q=just+publisher&ie=UTF-8&oe=UTF-8)), and not a plain Action, and switching the different error Types to return a slightly different one.

## Configuring the Views
Now that we have proper state variables to display everything we need in the View, let’s make some changes to improve the user experience:

1. After the user taps the button, we will display a ProgressView while an animal is loaded.
2. If there is an error, we will present an Alert, with a custom message depending on the error that occurred.

### Progress View
SwiftUI on iOS 14 (beta) supports the new ProgressView. (You had to use UIKit before), so let’s add it to our AnimalView.

#### AnimalView.swift
``` swift
import SwiftUI

struct AnimalView: View {
    @EnvironmentObject var store: AppStore

    func loadAnimal() {
        store.dispatch(.animal(action: .fetch))
    }

    var body: some View {
        VStack {
            if(store.state.animal.fetchInProgress) {
                ProgressView(“Fetching Animal…”)
            }
            else {
                Text(store.state.animal.current).font(.system(.largeTitle)).padding()
                Button(“Tap me”, action: { self.loadAnimal() })
            }            
        }
    }
}
```

I have added a new ProgressView and wrapped it around a condition, that observes the State Variable *fetchInprogress*.

Now, the only thing we need to do is change that variable to true while fetching an animal and reset it when done.  That happens in the Reducer.

#### AppReducer.swift
``` swift
import Foundation

func animalReducer(state: inout AnimalState, action: AnimalAction) -> Void {

    switch(action) {
        case .fetch:
            state.fetchInProgress = true

        case .fetchComplete(let animal):
            state.fetchInProgress = false
            state.current = animal

        case .fetchError(let error):
            state.fetchInProgress = false
            state.fetchError = ""

    }
}
```

We are getting there!

<img src="/assets/posts/2020-08-07-redux-like-architecture-with-swiftui-error-handling/result2.gif" width="400">


#### Alert Window
So far, we have decent error handling, but things can be a lot better.
Let’s stop displaying errors in the Label and show an Alert with details instead.

To present an Alert in SwiftUI, you have to use some view’s State property (a Bool).  When that property is true, the Alert will show, and tapping the Dismiss Button on it, will toggle the property back to false, hiding it.

In this case, we want to display an Alert based on whether the *FetchError* variable in our State is different than *nil*, so there a couple of things to consider.

1. Our State variable is not a Boolean (It’s a String), so we’ll need to add some logic.
2. We cannot use the *fetchError* variable directly because the State is read-only (remember?), and therefore the Alert dismiss action cannot change it.   That means we will have to dispatch an Action to reset it via a custom Binding.

Here’s how:

#### AnimalView.swift
``` swift
import SwiftUI

struct AnimalView: View {
    @EnvironmentObject var store: AppStore

    func loadAnimal() {
        store.dispatch(.animal(action: .fetch))
    }

    var body: some View {

        let shouldDisplayError =  Binding<Bool>(
            get: { store.state.animal.fetchError != nil },
            set: { _ in store.dispatch(.animal(action: .fetchError(error: nil))) }
        )

        VStack {
            if(store.state.animal.fetchInProgress) {
                ProgressView(“Fetching Animal…”)
            }
            else {
                Text(store.state.animal.current).font(.system(.largeTitle)).padding()
                Button(“Tap me”, action: { self.loadAnimal() })
            }            
        }
        .alert(isPresented: shouldDisplayError) {
            Alert(title: Text(“An error has Ocurred”),
                  message: Text(store.state.animal.fetchError ?? “”),
                  dismissButton: .default(Text(“Got it!”)))
        }
    }
}
```

What we did:

1. Created a *shouldDisplayError* custom binding.  When read, it returns the value of the State variable, and when set, it dispatches an action to reset the existing error.
2. Added an alert at the bottom, that uses the new custom binding in the *isPresented* property to show/hide the Alert.

Note that in the custom binding *set* parameter, we ignore whatever value comes from the Alert and are just dispatching an action.

``` swift
 set: { _ in store.dispatch(.animal(action: .fetchError(error: nil))) }
```

We are getting closer now.  Now let’s modify the Actions we are dispatching from the AnimalMiddleware when there’s an error.

This is how it looks:

#### AnimalMiddleware.swift
``` swift
enum AnimalMiddlewareError: Error {
    case unknown
    case networkError
}

func animalMiddleware(service: AnimalService) -> Middleware<AppState, AppAction> {

    return { state, action in
        switch action {

        case .animal(.fetch):
                return service.generateAnimalInTheFuture()
                    .subscribe(on: DispatchQueue.main)
                    .map { AppAction.animal(action: .fetchComplete(animal: $0 )) }
                    .catch { (error: AnimalServiceError) -> Just<AppAction> in
                        switch(error) {
                        case .unknown:
                            return Just(AppAction.animal(action: .fetchError(error: AnimalMiddlewareError.unknown)))
                        case .networkError:
                            return Just(AppAction.animal(action: .fetchError(error: AnimalMiddlewareError.networkError)))
                        }
                    }
                    .eraseToAnyPublisher()

            default:
                break
            }

        return Empty().eraseToAnyPublisher()
    }
}
```

I have created a new AnimalMiddlewareError type to define additional error types we may need later on and then simply modified our catch statement to return them accordingly.

Now let's update the Reducer to mutate the state based on each error.

#### AnimalReducer.swift
``` swift
import Foundation

func animalReducer(state: inout AnimalState, action: AnimalAction) -> Void {

    switch(action) {
        case .fetch:
            state.fetchError = nil
            state.fetchInProgress = true

        case .fetchComplete(let animal):
            state.fetchInProgress = false
            state.current = animal

        case .fetchError(let error):
            state.fetchInProgress = false
            switch error {
                case AnimalMiddlewareError.networkError:
                    state.fetchError = "Oops!.  It seems someone made a mistake!"
            default:
                state.fetchError = "I'm sorry, but the server went away"
            }
    }
}
```
<img src="/assets/posts/2020-08-07-redux-like-architecture-with-swiftui-error-handling/result3.gif" width="400">

Pretty cool, huh?  😎



## Conclusion and Next Steps
There is always room to continue improving error handling, such as retrying fetching the data automatically, adding localizations, and more.  Taking time to correctly handle errors in your app, especially in the UI, will always be welcomed by your users.

This tutorial completes the initial “Redux-like architecture” series, but stay tuned for new stuff coming up related to better managing your SwiftUI views, and other improvements to this approach.

As usual, you can get a fresh copy of the code from the [Repo](https://github.com/afterxleep/Redux-Architecture-SwiftUI-ErrorHandling).    I hope you enjoyed this tutorial.   If you have any questions or comments, feel free to ping me on [Twitter](https://twitter.com/afterxleep).

---
Posts in this series
1. [The Basics]({{ site.url }}/redux-like-architecture-with-swiftui-basics/)
2. [Side Effects]({{ site.url }}/redux-like-architecture-with-swiftui-middleware/)
3. Error Handling (This Post)


---

Sources & Refs:
- [Error Handling in Combine](https://www.avanderlee.com/swift/combine-error-handling/)
- [mcichecki Combine error handling playground](https://github.com/mcichecki/combine-error-handling)
- [Combine in practice - WWDC19](https://developer.apple.com/wwdc19/721)











