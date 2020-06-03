---
tags: [architecture, module, ios]
date: "2020-03-26T10:11:00Z"
title: Modular iOS architecture. Our way.

---

![Header image](/assets/2020-03-26/header.jpeg)

Here at Redmadrobot, we try to deal with serious businesses and long-term projects. So many of our clients work with us for years while we as developers improve functionality by new features. We need to refactor our code base from time to time, support new iOS versions to be on the edge.

<!--more-->

In this article, I tell you about our own experience to scale the developers’ team from 4 engineers up to 10 (6 teams now). The last architectural approaches scale badly for a big number of developers and different product units at one application. I prefer to call it a monolith. And our goal was to make it modular and let people work independently in their feature teams (units).

# Start point

![Code structure](/assets/2020-03-26/code.png)

4 iOS engineers, monolithic architecture, bank project, that is 5 years old, almost 300K lines of code, 86% Swift.

# Reasons to do it

1. Changes in project management model. From one team: manager, product owner, 4 iOS engineers to different units with their own teams and KPI. In the end of this year, the number of teams should be 20 (25+ iOS engineers).
2. Cold build (with empty cache) time is over 3 minutes on the latest MacBook Pro hardware. Not so long, but we always want it faster. Hot build (from cache) is around 1m 30 seconds.
3. At some point, we get a compilation error ***The operation couldn’t be completed. Argument list too long.*** That error occurs when the length of all source code file paths exceeds the maximum size. That path passed as an argument of xcodebuild in a terminal. The compile error can be shown at any team member because some of us use a longer path to the project folder. The workaround to put the project folder as higher as possible to root, but this is not the silver bullet. If you add new files intensively, that can be dead-end very soon. Look more at [radar](http://www.openradar.me/35879960).
4. iMessage, Siri, Notification Extension targets — each of them uses some sources, they marked with membership checkmark at the right panel. That looks like a mess when adding one involves some dependent files.

# High-level plan

We use Cocoapods as a dependency manager, so the main target was dependent on pods. As a result, we wanted to get a thin main target (Application) with dependent modules. Monolith can be divided by vertical layers — Model, Service Layer, Common UI as well as horizontal layers — features. The great thing about pods is with modules we can connect specific dependency directly to some module, not the whole app.

![Monolith scheme](/assets/2020-03-26/monolith-scheme.png)

# Just do something, everything will be fine

If you start without any plan, the situation can be something like this.

![Monolith first step](/assets/2020-03-26/monolith-services.jpeg)

Firstly, you extract the services module. Services depend on Alamofire and also on models. But models now are a part of the monolithic main app, so you get a dependency cycle!

My recommendation to start from simple things like styles (fonts, colors), other libraries that you have like a source code folder in your app (loaders, loggers, analytics). After that go to the model layer and extract models. We have different modules for models (structs) and DTOs (Decodable objects).

Next, we extract code that deals with a database, common UI elements, and lastly feature modules.

# Who is a module?

If we speak in terms of modules, that can be different things: cocoapods dependency, dynamic framework target, a dependent project with a dynamic framework as the main target (subproject). Let’s dive into each of them.

## Cocoapod

Our dummy podfile will look like this.

```ruby
platform :iOS, ’13.0’
use_frameworks!

target ‘ModuleExample’ do
  pod ‘Model’, :path => ‘Model’
end
```

Podspec for Model module.
```ruby
Pod::Spec.new do |s|
  s.name             = ‘Model’
  s.version          = ‘1.0’
  s.summary          = ‘Model Module’
  s.homepage         = ‘a link’
  s.author           = { ‘vani2’ => ‘vani2@me.com’ }
  s.source           = { :git => ‘a link’, :tag => s.version, :submodules => true }
  s.platform         = :iOS, ’13.0’
  s.swift_version    = ‘5.1’
  s.source_files     = ‘Classes/*’
end
```

In the navigator it look like this.

![Navigator with cocoapod](/assets/2020-03-26/navigator.png)

User.swift

```swift
public struct User {
  public let name: String

  public init(name: String) {
    self.name = name
  }
}
```

Using in the app.

```swift
import UIKit
import Model

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

  let user = User(name: “Johny”)

  func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    return true
  }
}
```

Advantages of using Cocoapods:

1. Reusable in other projects. Cocoapods let you make a standalone framework and publish it into a private repository. Reusing is simple, you just need to properly set the name of your pod and private spec repository or private git URL. If reusing is not the case, you can attach pod locally.
2. Cocoapods make all “magic” for you (signing, importing). Cocoapods properly integrate your framework into a workspace, don’t worry about anything.

The disadvantage is making a proper podspec.

## Dynamic framework target

Just add a new target and choose the right template.

![Add new framework](/assets/2020-03-26/new-framework.png)
![Xcode new project](/assets/2020-03-26/project.png)
![Add new file to framework](/assets/2020-03-26/new-file.png)

Don’t forget to add the framework to the main target. Also, check file membership while adding a new file to the module.


Our pod file now looks like this.

```ruby
platform :iOS, ’13.0’
use_frameworks!

target ‘ModuleExample’ do
end

target ‘Service’ do
  pod ‘Model’, :path => ‘Model’
end
```

Service.swift

```swift
import Foundation
import Model

public class Service {
  public init() {}
    
  public func getUsers() -> [User] {
    return [User(name: “Johny”)]
  }   
}
```

```swift
import UIKit
import Service

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

  let user = Service().getUsers()
    
  func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {      
    return true
  }
}
```

This approach looks the easiest for us and we’ve chosen it. No need to think about signing dependencies, you just need to import your framework to targets you want. Also, it’s convenient to see all the targets in one list.

Later on, we figure out the main disadvantage is you need to manage file location manually because there is no checking of a new file location inside the target folder. Also, you have one *pbxproj*-file so it’s hard to merge it.

## Subproject

Create a new project with a dynamic framework template and add it to our workspace.

![Xcode new subproject](/assets/2020-03-26/new-project.png)

Create a new project with a dynamic framework template and add it to our workspace. The result should be the following.

![Xcode new subproject](/assets/2020-03-26/new-subproject.png)

Don’t forget to add *Database.framework* as a dependency to the main target as Service module earlier.

In this way, you create a project with a dynamic framework target and add it as a dependency. The advantages are:

* Membership managed automatically — the file can be the target of one project only.
* *pbxproj* is not huge, because now you have many of them.

We using [XcodeGen](https://github.com/yonaskolb/XcodeGen), so the *pbxproj* merge is not very painful for our project.

The main disadvantage was to disable signing by custom script.

Now Application delegate looks like this.

```swift
import UIKit
import Service
import Database

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

  let user = Service().getUsers()
    
  func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
    Database.clean()
        
    return true
  }
  
}
```

## Other

With Swift Package Manager it’s not possible to have resources in the module now, Carthage is not widely used in our project before. Git submodule management is a pain too.

# Final architecture

The dependency graph has become more clean and obvious. During the extraction of modules, you’ll find some circular dependency and if you do properly, you can achieve loose coupling between modules. Siri Extension uses Services and Models. Now we are in the process of split application into features, that seems we need to use protocols for communication between modules.

![Final architecture scheme](/assets/2020-03-26/final.jpeg)

# That’s not so simple

* It’s not easy to add new functionality to the app and refactor for modules. At least you need to be a git merge ninja.
* Every class, property, function visible outside should be open or public so long boring changes. Also if you have structs, be ready to write an initializer for every public stricture (swift generates internal initializer).
* Dances with static libraries, like Google maps. Classes will be duplicated if you added it twice or more, which makes the app start slower. ***Class <…> is implemented in both <…>. One of the two will be used. Which one is undefined.***
* If your storyboards and nib-files contain custom classes for controls, check the module name after extracting a new module with UI classes.

# Other optimizations

While refactoring we tried to tune in some build settings to improve building time. They are

* ***-Xfrontend -warn-long-function-bodies=300 and -Xfrontend -warn-long-expression-type-checking=300.*** With these settings, you get a warning to functions and expressions with long compile-time (300 ms).
* Build Active Architecture Only ***ONLY_ACTIVE_ARCH = YES*** — that tells the compiler to build for the selected device, that is much quicker.
* Optimization Level — No Optimization for Debug ***SWIFT_OPTIMIZATION_LEVEL = “-Onone”.*** Optimization takes some time, so no optimization is faster.

# Results

* After we have added 10 modules application start increased by 50 ms with a total 800 ms. The disadvantage of the static library is duplicated code and bigger application size as a result.
* In Xcode 11(beta 3) ***USE_SWIFT_RESPONSE_FILE*** setting was added, so paths to swift sources now passed by text file.
* By different build setting changes, refactor some complex code structures and modules — after small change compiler build only changed file or module, we reduce our hot build time from 1 minute 30 seconds to 23 seconds. But a cold build time increased by 60% up to almost 6 minutes.
