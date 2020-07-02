---
layout: post
comments: true
title: Combine and Protocols in Swift - Defining Protocols with a @Published property wrapper
image: /posts/2020-07-01-combine-and-protocols-in-swift/header.jpg
tags:
  - development ios swift
---

The [@ObservableObject and @Published](https://www.hackingwithswift.com/quick-start/swiftui/what-is-the-published-property-wrapper) property wrappers are the cornerstone of Combine powered apps. With Combine and SwiftUI, it's easy to use the @Published wrapper in our ViewModels/Presenters and have the Views automatically update as changes to these properties happen.

Everything works great until you want to use Protocols to facilitate dependency injection and testing in your Models <!--more--> and ViewModel classes, as we've been doing in our regular MVVM apps for the past few years,

You will soon discover that Swift (As of now, version 5.3) does not support property wrappers in Protocol declarations, and marking a property as @Published in a protocol will throw an error.

## The Problem
To explain the issue, we will use a Playground to write a quick demo app using Combine and SwiftUI and following the MVVM pattern.  It will consist of three components

### AnimalGenerator
This is our Model.  It holds a list of animal names and has has a method to publish one random animal name via a Publisher.  *(@Published private(set) var name)*

### TheViewModel
Our ViewModel, responsible for managing the View.  It owns our Model, subscribes to it and then re-publishes *name* via its *textToShow* publisher. *(@Published var textToShow).*

### TheView
This is a simple SwiftUI View, with a text and a button.  It owns our ViewModel and subscribes to it.  It displays the ViewModel's *textToShow* value as a Text whenever the data changes.

When you tap the button, it calls the generate() method in the ViewModel, that does the same in the Model, and then Combine does the rest.


#### Initial Code
```swift
class AnimalGenerator: ObservableObject {

     @Published private(set) var name: String = ""
     let animals = ["Cat", "Dog", "Crow", "Horse", "Iguana", "Cow", "Racoon"]

    init() {
        generate()
    }

    func generate() {
        self.name = animals.randomElement() ?? ""
    }
}

class TheViewModel: ObservableObject {

    @Published var displayData: String = "No data"
    var generator: AnimalGenerator = AnimalGenerator()
    var cancellable: AnyCancellable?

    init() {
        cancellable =
            generator
                .$name
                .receive(on: RunLoop.main)
                .sink { [weak self] data in
                    self?.displayData = data
            }
    }

    func generate() {
        generator.generate()
    }
}

struct TheView: View {

    @ObservedObject var viewModel: TheViewModel = TheViewModel()

    var body: some View {
        VStack {
            Text(viewModel.displayData).font(.system(.largeTitle)).padding()
            Button("Tap me", action: { self.viewModel.generate() })
        }
    }
}

PlaygroundPage.current.setLiveView(TheView())
```

As you can see, everything works, but now let's say that instead of instantiating AnimalGenerator in our ViewModel, we want to use a protocol, and use different models, depending on what type of content we want to generate and display in the View.

Let's write a simple protocol called * "Generator"* and update our * "AnimalGenerator"* class to conform.

#### Our Protocol
```swift
protocol Generator {
    @Published var name: String { get }
    func generate()
}
```

#### Our AnimalGenerator Model conforming to it
```swift
class AnimalGenerator: Generator, ObservableObject {

     @Published private(set) var name: String = ""
     let animals = ["Cat", "Dog", "Crow", "Horse", "Iguana", "Cow", "Racoon"]

    init() {
        generate()
    }

    func generate() {
        self.name = animals.randomElement() ?? ""
    }
}
```

#### And the expected error (Oops!)

![Property declared inside a protocol cannot have a wrapper](/assets/posts/2020-07-01-combine-and-protocols-in-swift/wrapper-error.png)
`Property declared inside a protocol cannot have a wrapper.`

The error is reminding us what I was mentioning at the beginning: Wrappers and Stored properties are not allowed in Swift protocols and extensions (at least for now).


## The Workaround
Before doing the fix, let's talk about the @Published wrapper.

In essence, a property wrapper is a type that wraps a value and attaches some logic to it.   When a property of an ObservableObject has a @Published wrapper, specific logic is attached to it, so its value is published anytime it changes.  More explicitly in the *willSet* block.

In simple terms, @Published creates a publisher that can be accessed with the *'$'* operator as we are doing in our ViewModel, allowing us access to its value whenever it changes in the future.

Since we cannot use a @Published wrapper as part of our protocol declaration, we need to describe its synthesized methods explicitly. Let's change our Generator protocol and AnimalGenerator Model to the following.

#### Our Updated Protocol
```swift
protocol Generator {
    var name: String { get }
    var namePublished: Published<String> { get }
    var namePublisher: Published<String>.Publisher { get }
    func generate()
}
```

#### Our Updated Model conforming to it
```swift
class AnimalGenerator: Generator, ObservableObject {

    var namePublished: Published<String> { _name }
    var namePublisher: Published<String>.Publisher { $name }

     @Published private(set) var name: String = ""
     let animals = ["Cat", "Dog", "Crow", "Horse", "Iguana", "Cow", "Racoon"]

    init() {
        generate()
    }

    func generate() {
        self.name = animals.randomElement() ?? ""
    }
}
```

We have now manually defined our Publisher in the Protocol declaration, and exposed the *namePublisher* publisher in the Model, thus creating a polymorphic interface that allows us to decouple things and now use our protocol in the ViewModel.

#### Our Updated ViewModel
```swift
class TheViewModel: ObservableObject {

    @Published var textToShow: String = "No data"
    var generator: Generator
    var cancellable: AnyCancellable?

    init(generator: Generator) {
        self.generator = generator
        cancellable =
            generator
                .namePublisher
                .receive(on: RunLoop.main)
                .sink { [weak self] data in
                    self?.textToShow = data
            }
    }

    func generate() {
        generator.generate()
    }
}
```

Note that we are now using the *generator.namePublisher* we manually defined instead of *generator.$name*.

## And now, let's fix the View
In the original example, we are instantiating the ViewModel from the View, and we can continue to do the same, but just passing the Generator we want to use, like this:

#### Our Updated View
```swift
struct TheView: View {

    @ObservedObject var viewModel: TheViewModel = TheViewModel(generator: AnimalGenerator())

    var body: some View {
        VStack {
            Text(viewModel.textToShow).font(.system(.largeTitle)).padding()
            Button("Tap me", action: { self.viewModel.generate() })
        }
    }
}
```

It works, but it does not look good, as we are now coupling the View, ViewModel and Model together there, so let's instantiate our ViewModel and Model separately and inject them into our View.

First, let's update our View and remove the initialization for the ViewModel.

#### And the final version
```swift
struct TheView: View {

    @ObservedObject var viewModel: TheViewModel

    var body: some View {
        VStack {
            Text(viewModel.textToShow).font(.system(.largeTitle)).padding()
            Button("Tap me", action: { self.viewModel.generate() })
        }
    }
}
```

And then, we can simply instantiate our Model and ViewModel and pass them to the View like this.

#### Instantiating our Model and Viewmodel together
```swift
let viewModel: TheViewModel = TheViewModel(generator: AnimalGenerator())

PlaygroundPage.current.setLiveView(
    TheView(viewModel: viewModel)
)
```

#### Let's try it out with another Model

```swift
class PersonGenerator: Generator, ObservableObject {

    var namePublished: Published<String> { _name }
    var namePublisher: Published<String>.Publisher { $name }

     @Published private(set) var name: String = ""
     let persons = ["John", "Jane", "Carlos", "Daniel", "Helen", "David", "Bill"]

    init() {
        generate()
    }

    func generate() {
        self.name = persons.randomElement() ?? ""
    }
}

let viewModel = TheViewModel(generator: PersonGenerator())

PlaygroundPage.current.setLiveView(
    TheView(viewModel: viewModel)
)

```

## Note on iOS 14
iOS 14 introduced a new @StateObject wrapper, and it might be a good idea to use it instead of @ObservedObject.

An @ObservedObject instance is replaced every time SwiftUI decides to discard and redraw the View, and thus you might experience weird crashes in different scenarios.  By using @StateObject, you ensure the instance we create is retained even when the View is redrawn or discarded.

Using @StateObject is then particularly useful when you are instantiating your ViewModel from the View itself, as we did in the first example

## Wrapping up
As you can see, it is reasonably straightforward to use Combine and maintain a polymorphic approach via Protocol Oriented Programming in your classes.

Some would say that the next step would be decoupling the View itself to become completely ignorant of it's ViewModel and use a Protocol instead, especially if you are still stuck with UIKit and UIViewControllers. Nevertheless, in this case, I find it overkill.   Maybe we can cover that in another post.

The final example is available [here](https://gist.github.com/afterxleep/2246e5a38ea3704b9f9b866a106b8d8f), so go ahead, paste it on a Playground and send over your comments and suggestions.


