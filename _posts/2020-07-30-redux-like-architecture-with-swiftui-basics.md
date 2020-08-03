---
layout: post
comments: true
title: Redux-like architecture with SwiftUI&#58; Basics
image: /posts/2020-07-30-redux-like-architecture-with-swiftui-basics/header.png
tags:
  - development ios swift
---

Since the beta release of iOS 14 and now that SwiftUI is a little more mature, I have been exploring different architectures that better fit its patterns. After working with React for a while, I got curious about using something similar to Redux on iOS.<!--more-->

The basic idea is to describe the whole App State using a single or set of structs, to have a single source of truth for the entire application.  This means, having all the information required by our User Interface readily available and up to date.

## Intro to Redux
Let's start with a basic intro to the pattern.  If you are already familiar with it and just want to get to coding, feel free to skip to the next section

Redux architecture is about strict, unidirectional data flow.  All data in the application follows the same lifecycle, making the logic easier to understand and debug and eliminating the need to maintain multiple copies of data across the application.

With Redux (and Flux), all the data in your app follows a single direction, and it's kept at the same place (the App State), which is the single point of truth.  Check out the data flow below.

#### Data Flow
![](/assets/posts/2020-07-30-redux-like-architecture-with-swiftui-basics/diagram1.png)


### Action
Actions contain the data that we have to process to mutate the App State, and a reference to a function (Reducer) that will be used to perform those modifications.

### Dispatcher
It is the central hub that manages all the data flow in your app but has no real intelligence or function.  It receives an action and sends it to the Store for processing.

### Store
It contains the application state and its logic.  Similar to a Model in classic MVC, the Store manages the State of every single object in your application.

When the Store receives an action, it passes the data to a [reducer function](https://redux.js.org/basics/reducers)  that receives a "live" copy of the State, alongside some data, and then mutates the State accordingly.

Reducers are pure functions and should be limited to computing the next version of the App State.  They should be 100% predictable, which means that calling them with the same inputs, should always produce the same results.  

They should be the only ones responsible for mutating your app's State.

### Views
Views have read-only access to the Application State.  This is a huge advantage when it comes to debugging, updating the UI, and dealing with complex data sets.  

In Redux, the only way to update the State is through a dispatcher, which you can call from the views, based on user interaction like the tap of a button or typing on a text field.  (As you can see in the diagram below)

#### Data Flow (Dispatching from a View)
![](/assets/posts/2020-07-30-redux-like-architecture-with-swiftui-basics/diagram2.png)

---

## Let's get to iOS
For his example, we will be re-building the same animal name generator from the [last Combine and protocols post]({{ site.url }}/combine-and-protocols-in-swift/) using an interpretation of the Redux architecture.


#### The final result
<br>
<img src="/assets/posts/2020-07-01-combine-and-protocols-in-swift/demo.gif" width="400">
<br>


#### Note on Xcode 12
I will be using Xcode 12, (currently in Beta), and creating a new app with the new SwiftUI lifecycle, which helps us get rid of the AppDelegate and storyboards completely, but you can use Xcode 11 as everything works in iOS 13.

To create the app, select these options.

![](/assets/posts/2020-07-30-redux-like-architecture-with-swiftui-basics/xcode.png)

---

Now let's create the different parts of our app.  I have separated them into different files for clarity, but do as you see fit.

### App State
Our app state is a single Struct.

#### AppState.swift
```swift
import Foundation

struct AppState {
    var currentAnimal: String = ""
}
```

The idea here is to inject a read-only version of the AppState to all our views using the SwitUI environment feature.  We will put a single object in our root view, and pass it along with the @EnvironmentObject property to every child that needs access to it.


### App Actions
As we mentioned before, we will modify our State by dispatching actions to our reducers.  Every Action gets an identifier, and if required, some data for the Reducer to use.   For iOS, we will use a simple enum to define our actions.

#### AppActions.swift
```swift
import Foundation

enum AppAction {
    case getAnimal     
}
```

### App Reducers
Remember about reducers?   These guys are responsible for receiving the current State and performing an action to mutate it.

For convenience, we will create a typealias for our Reducers.  They will receive the State as an inout parameter, and a corresponding Action.  By using an inout parameter, we guarantee that we are mutating the only source of truth, every single time.

Then we will have a simple appReducer function that will switch between the available actions in our app and mutate the State accordingly.

#### Reducers.swift
```swift
typealias Reducer<State, Action> = (inout State, Action) -> Void

func appReducer(state: inout AppState, action: AppAction) -> Void {

    switch(action) {

        case .getAnimal:
            state.currentAnimal = ["Cat",
                                   "Dog",
                                   "Crow",
                                   "Horse",
                                   "Iguana",
                                   "Cow",
                                   "Racoon"]
                .randomElement() ?? ""

    }

}
```


### App Store
We now need to create our Application Store, which will hold the State of the app and enable read-only access.  

#### AppStore.swift
```swift
import Foundation

typealias AppStore = Store<AppState, AppAction>
final class Store<State, Action>: ObservableObject {

    // Read-only access to app state
    @Published private(set) var state: State

    private let reducer: Reducer<State, Action>

    init(initialState: State, reducer: @escaping Reducer<State, Action>) {
        self.state = initialState
        self.reducer = reducer
    }

    // The dispatch function.
    func dispatch(_ action: Action) {
        reducer(&state, action)
    }
}
```

First, we will define an AppStore typealias for convenience, and then our final class, to hold the application Store.

Since we are using SwiftUI, we can make it an ObservableObject, and set the App State to be a @Published property.   That way, our views can subscribe to changes and update themselves automatically.  To make it read-only, we set the property access control to private(set).

Our initializers allow us to set an initial State, in case we want to load some data into it at initialization, and lets us define the Reducer we will be using to mutate it.

As you can see, we are also defining our Dispatch function, which takes an Action as a parameter and passes it to a reducer.

That's it.  With about 35 lines of code, we are driving all our data flow of our app.

### View
A simple view, with a label and a button.   The current animal is displayed in the label, and the button will dispatch an action that selects another random animal.

#### AnimalView.swift
```swift
import SwiftUI

struct AnimalView: View {
    @EnvironmentObject var store: AppStore

    func loadAnimal() {
        store.dispatch(.getAnimal)
    }

    var body: some View {
        VStack {
            Text(store.state.currentAnimal).font(.system(.largeTitle)).padding()
            Button("Tap me", action: { self.loadAnimal() })
        }
    }
}
```

As you can see, the view binds itself to the AppStore via the @Environment property wrapper and gets its data from the State.

When you tap the button, we dispatch the "getAnimal" Action.  It arrives at our Reducer, which generates a random name and modifies the State.  SwiftUI automatic binding re-renders the view as soon as the state changes.  Pure magic!.


### Main View
The only thing we have to do is to initialize the State at launch, include our AnimalView, and pass the Store as an environment object.

Let's modify our default project View.

#### ContentView.swift
```swift
struct ContentView: View {
    let store = AppStore(initialState: .init(currentAnimal: "Dog"),
                         reducer: appReducer)

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

As you can see, we set the "currentAnimal" value by force when initializing the Store so that we can have something rendered in the view at launch.  This is useful if, for example, we were loading the data from disk or a database.

However, I don't recommend modifying the State directly, even on initialization, as it breaks the Data flow of the app. Use this only when there is no other choice.

To fix this, let's create an initializer that simply dispatches the getAnimal Action, as follows:

#### ContentView.swift
```swift
import SwiftUI

struct ContentView: View {
    let store = AppStore(initialState: .init(),
                         reducer: appReducer)

    init() {
        store.dispatch(.getAnimal)
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

---

That's it!.   Pretty neat, huh?.     There is plenty of room for optimization, and we will follow up with this in our next post.   

The resulting app for this post is available in [this repo](https://github.com/afterxleep/Redux-Architecture-SwiftUI-Basics).

On [the next post]({{ site.url }}/redux-like-architecture-with-swiftui-middleware/), we will be improving the app to support Side Effects and perform async actions.

I hope you enjoyed this tutorial.   If you have any questions or comments, feel free to ping me on [Twitter](https://twitter.com/afterxleep).


---
Posts in this series
1. The Basics (This Post)
2. [Side Effects]({{ site.url }}/redux-like-architecture-with-swiftui-middleware/)

---
Sources & Refs:
- Special thanks to [Majid](https://twitter.com/mecid) for his [Redux-like containers post](https://swiftwithmajid.com/2019/09/18/redux-like-state-container-in-swiftui/) in which this example is based.
- [Swift UI with Redux](https://github.com/kitasuke/SwiftUI-Redux)
- [Swift Hub Tutorials](https://swiftuihub.com/swiftui-tutorials/redux/)
- [How to implement Redux in Swift](https://medium.com/swlh/how-to-implement-redux-in-swift-a2a0e7b7f10e)
- [Redux.js](https://redux.js.org)
- [ELM architecture](https://www.google.com/search?client=safari&rls=en&q=elm+architecture&ie=UTF-8&oe=UTF-8)
- [Flux Architecture](https://facebook.github.io/flux/docs/in-depth-overview/)












