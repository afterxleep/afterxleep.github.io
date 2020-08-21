---
layout: post
comments: true
title: Redux-like architecture with SwiftUI&#58; Real World App
image: /posts/2020-08-21-redux-like-architecture-with-swiftui-real-world-app/header.png
tags:
  - development ios swift
---

After looking at the Redux approach, figuring things out and writing the previous set of posts ([The Basics]({{ site.url }}/redux-like-architecture-with-swiftui-basics/), [Side Effects]({{ site.url }}/redux-like-architecture-with-swiftui-middleware/) and [Error Handling]({{ site.url }}/redux-like-architecture-with-swiftui-error-handling/)), I wanted to put it all in practice with a real app, and decided to rewrite Bouncer (again).<!--more-->

Bouncer is a little open-source app I wrote, that takes advantage of the *IdentityLookup* framework to [filter unwanted messages](https://developer.apple.com/videos/play/wwdc2017/249/).  It is as simple as a plain text word-list, and an extension that matches message contents against it.  So simple, it’s the perfect candidate to try this out.

[The original version](https://github.com/afterxleep/Bouncer/releases/tag/v1.2.0) was written using an MVP+Coordinators pattern with UIKit, so the process required a full rewrite of the UI too, but the good thing is that (not considering the time I spent on design and tweaking things in SwiftUI to make it look nice) the full rewrite took me approximately 8 hours of coding work, including refactoring the Models to use Combine and writing tests for them.

If you followed my tutorials on Redux, you’d find the code to be very clear and easy to follow, and the only thing that might look a bit different is the “ContainerViews.”

To de-couple the data flow, I have used the “[Container views in SwiftUI](https://swiftwithmajid.com/2019/07/31/introducing-container-views-in-swiftui/)” approach Majid wrote about last year.  In general, a Container View is responsible for the data flow (Dispatch actions and access our state), but does not have any User interface and passes the data to a dumb *Rendering View* that presents it.

[The new version](https://github.com/afterxleep/Bouncer) was written using Xcode 12 beta and the latest available beta version of iOS14, which includes new features on message filtering, its open-source too and will be available in the App Store as soon iOS 14 is released.  

If you have any ideas, comments, improvements, or want to help with localization, feel free to open a Pull Request to [the repo](https://github.com/afterxleep/Bouncer) or drop me a line on [Twitter](https://twitter.com/afterxleep).

---
Posts in this series
1. [The Basics]({{ site.url }}/redux-like-architecture-with-swiftui-basics/)
2. [Side Effects]({{ site.url }}/redux-like-architecture-with-swiftui-middleware/)
3. [Error Handling]({{ site.url }}/redux-like-architecture-with-swiftui-error-handling/)
4. Real World App (This Post)


---

Sources & Links:
* [‎Bouncer on the App Store](https://apps.apple.com/us/app/bouncer-sms-block-list/id1457476313) (Version 1.x while iOS 14 is released)
* [Bouncer GitHub](https://github.com/afterxleep/Bouncer) (Version 2.x)
* [Bouncker Github UIKit version](https://github.com/afterxleep/Bouncer/releases/tag/v1.2.0) (Version 1.x last release)
* [Introducing Container views in SwiftUI - Swift with Majid](https://swiftwithmajid.com/2019/07/31/introducing-container-views-in-swiftui/)
* [SMS and message filtering -  Apple](https://developer.apple.com/documentation/sms_and_call_reporting/sms_and_mms_message_filtering)
* [Filtering Unwanted Messages with Identity Lookup - WWDC 2017](https://developer.apple.com/videos/play/wwdc2017/249/)











