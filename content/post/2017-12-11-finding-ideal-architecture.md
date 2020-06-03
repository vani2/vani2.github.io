---
tags: [architecture, ios]
date: "2017-12-11T12:12:00Z"
title: Finding ideal architecture

---

![Header image](/assets/2017-12-11/header.png)

For 9 years since Redmadrobot was founded, year by year, day by day there were more features in our applications. That’s become easier to get confused with. When engineers became more than a dozen, there was another problem — how quickly swap people between projects. Outsource development has strict deadlines, and engineers have no months or weeks to dive into specifics of a new project, whilst it’s unnecessary to let people work on several projects through their career because they don’t get bored and develop new skills.

<!--more-->

The main problem in long live applications — scalability. The solution is a transition to a new architecture, in other words, refactoring of the code base with adding new entities, that make a massive object more lightweight.

# Disclaimer

There is no term like “Ideal architecture”. A silver bullet doesn’t exist. Everyone chooses in favor of one, to work comfortably with during the whole project. What we effectively use can be overhead in other teams.

# Beginning


We’ve begun with popular MVC and his buddy Network Manager. Main problems of MVC: making requests to network and database, business logic, and navigation are inside view controller. Because of that objects have a strong coupling, reusing of code, and testing is almost impossible.

Network Manager becomes [God Object](https://en.wikipedia.org/wiki/God_object) and it’s impossible to maintain it because of his size.

Let’s assume we have a store application. It has deals, a profile, and settings screens. Deals screen displays current deals in the store, profile screen — bonuses of the user, his name, and phone, in settings he can change push-notifications settings about new deals. To use the app you need to authorize by login and password.

So, all network requests — authorization, deals list, profile information, changing settings are in Network Manager. The logic for creating model objects from JSON is there too.

## Goodbye, Network Manager

We decided to use the SOA approach — separation service layer for many services due to a model type.

![Services UML](/assets/2017-12-11/services.png)

Concrete Service in our example app — AuthService, UserService, DealsService, and SettingsService. Authorization Service works with authorization flow, user service — with a user, and so on. It’s a good practice to define different paths on a server-side: `/auth`, `/user`, `/deals`, `/settings`.

## Serialization/deserialization of JSON/XML and others.

For serialization and deserialization of objects, we use `Parser` and `Serializer` classes. Operations are the reverse: `Parser` map data from the server to model object, `Serializer` — from model object to data type for transfer to the server. Inside these classes, we check mandatory fields and log errors.

```swift
class UserParser: JSONParser<User> { 
  func parseObject(_ data: JSON) -> User? 
} 

class UserSerializer: JSONSerializer<User> { 
  func serializeObject(_ object: User) -> Data? 
}
```

![Parser UML](/assets/2017-12-11/parser.png)

For each entity, we have parsers: AuthParser, UserParser, DealParser, and SettingsParser. The same is with Serializers, except situations we don’t need it.


# Layered architecture

We’ve decided to separate our objects into different layers, the upper layer knows only about the bottom layer.

![Application layers](/assets/2017-12-11/layers.png)

## Persistence layer

We implement this layer by DAO pattern most of the time. It let us abstract from database specifics on another layer. We have implemented solutions for Realm and CoreData. The Realm is our favorite now. For more information check [our repo](https://github.com/RedMadRobot/DAO).

![DAO UML](/assets/2017-12-11/dao.png)

Let’s assume we need to cache deals in our app. From this moment we’ll create classes:

* DBDeal — Realm entity (entry) for Deal class
* DealTranslator — translator for entities.
* DealDAO — DAO for Deals (based on Realm).

# What about the UI layer?

We’ve had several attempts. We’ve analyzed popular approaches: MVVM and VIPER, but that was difficult to decide without practical use. VIPER for our needs has seemed overhead: a lot of classes for one module (in many cases they are just proxy in a call chain), a complex implementation for storyboard routing, fighting with UIKit. For sure, the testability of VIPER is a big advantage.

In our opinion, MVVM was easier to catch with MVC knowledge. Bindings help us with data updates, code has become testable. We had no problems with reactive programming because we’ve tried it with MVC before.

## Transition to MVVM

There are a lot of sources about MVVM (like [this](https://www.objc.io/issues/13-architecture/mvvm/)), so there is no need to describe it one more time. What is the main advantage for us in MVVM? In most cases, information from the server should be changed somehow and displayed to the user. That’s why logic is encapsulated inside view models or inside a factory of view models, in case of depending on each other.

![View Model](/assets/2017-12-11/viewmodel.png)

## Transition to Presentation Model and Router

From some moment we’ve understood that MVVM is too little for us. View Controller became bigger and bigger, especially if there are several network requests.

The next step was to extract a service call to a new entity — Presentation Model, and View Controller has lost knowledge about services.

Navigation (with segue or without) from one view controller to many others leads to the Massive View Controller also. Note that the calling method is just 2–3 lines of code, but a configuration of data for a new view controller can be for instance 10 lines. That is the reason we’ve encapsulate navigation to Router (I guess many of you think that’s VIPER almost 😅). Using routers becomes convenient also when we’ve started using xibs instead of storyboards. Reading router class is more difficult to storyboard screens with segues, but it’s a mess when your navigation code everywhere in view controllers.

![Router UML](/assets/2017-12-11/router.png)

`Router` — property view controller, we create it in viewDidLoad() method. In some projects, we’ve created it just before navigation.

Please note we don’t strictly separate every view controller for a presentation model and router. For a simple screen, it’s useless, for example, with a view controller with 200 lines.

Our router has the following methods:
* `showProfile()` — show user profile.
* `showDeal(_ deal: Deal)` — show full information about the deal.
* `showSettings()` — show settings.

Settings and profile are the only single instance, there is no need to pass parameters to a router for configuration. Opposite, with the list of deals we need to specify what information for the deal to show, configuring presentation model for full information view controller with the deal model (or view model if it’s good enough).

## What about table and collection?

The first step was to create a data source and delegate as a separate class and store it on a view controller. This class took view models from a presentation model.

![Router UML](/assets/2017-12-11/collection.png)

Cell Mapper — closure, that define cell class for specific view model class. That’s to avoid register cells for table or collection. A lot of code moved to ListDataSource. After some time we understood it was uncomfortable to use delegate outside view controller. And one class for data source had no big deal.

Later we’ve simplified our scheme: TablePresentationModel became data source, and view controller became a delegate. That’s simpler, means better.

![Router UML](/assets/2017-12-11/presentationmodel.png)

`DataSource` and `CellMapper` are depricated now.

## Changes in routing

All methods should be written inside a view controller in the current implementation. For loose coupling between navigation and internals of view controller we’ve made the following:

1. For the concrete presentation model, we have optional closure — hander (or several, if we need some different navigation cases).
2. After creating of presentation model we set this handler. Let’s say, navigate to the full information screen.
3. Inside view controller, we call handler.

As a result, the view controller has lost knowledge of routing.

# Code generation

One more thing. Developers in Redmadrobot have their code generation features. Based on model objects with console utility we have generated [parsers](https://github.com/RedMadRobot/core-parser-generator), [translators](https://github.com/RedMadRobot/DAO-generator) for DAO.
This is our Deal entity, that we’ve written.

This is our `Deal` entity, that we’ve written.

```swift
/* 
    Deal
    @model
 */
class Deal: Entity {

    /* 
        Title
        @json 
     */    
    let title: String

    /* 
        Subtitle
        @json 
     */    
    let subtitle: String?

    /* 
        End date of deal
        @json end_date
     */    
    let endDateString: String 

    init(title: String, subtitle: String?, endDateString: String) {
        self.title = title
        self.subtitle = subtitle
        self.endDateString = endDateString
        super.init()
    }

}
```

Using annotations (`@model`, `@json`) utility generates class `DealParser`, class `DBDeal` for Realm, and `DealTranslator`.

```swift
class DealParser: JSONParser<Deal> {

    override func parseObject(_ data: JSON) -> Deal? {
        guard
            let title: String = data["title"]?.string,
            let endDateString: String = data["end_date"]?.string
        else { return nil }
        
        let subtitle: String? = data["subtitle"]?.string

        let object = Deal(
            title: title,
            subtitle: subtitle,
            endDateString: endDateString
        )
        return object
    }

}
```

```swift
class DBDeal: RLMEntry {
    
    @objc dynamic var title = ""

    @objc dynamic var subtitle: String? = nil

    @objc dynamic var endDateString = ""

}
```

```swift
class DealTranslator: RealmTranslator<Deal, DBDeal> {    
    
    override func fill(_ entity: Deal, fromEntry: DBDeal) {
        entity.entityId = fromEntry.entryId
        entity.title = fromEntry.title
        entity.subtitle = fromEntry.subtitle
        entity.endDateString = fromEntry.endDateString
    }
    
    
    override func fill(_ entry: DBDeal, fromEntity: Deal) {
        if entry.entryId != fromEntity.entityId {
            entry.entryId = fromEntity.entityId
        }
        entry.title = fromEntity.title
        entry.subtitle = fromEntity.subtitle
        entry.endDateString = fromEntity.endDateString
    }
    
}
```

Recently we started to generate services based on a protocol of its methods (that is a good thing for another article). Before we started to use Zeplin, we generated color and font styles based on a text file. For code generation utilities we use our [Model Compiler](https://github.com/RedMadRobot/model-compiler), but if you want something like you can check very popular [Sourcery](https://github.com/krzysztofzablocki/Sourcery).

# Conclusion

Developing our architecture, we, first of all, thought about the scalability of our projects, SOLID principles and easy way to understand for new developers. Of course, we also face complex scenarios where some parts of our architecture are not very good and we come up with how to get out of this situation, how to put the responsibility on auxiliary entities and make the code more understandable. Obviously, no architecture solves absolutely all problems. On several projects that we are developing for several years, our approaches have proved to be convenient and rarely there are any problems with this.
We do not preach MVC, MVVM, VIPER, Riblets, and other architectures. We are constantly trying something new without compromising efficiency. At the same time, we try not to reinvent the bicycle. Then we check how convenient it is to work with this or that approach, how quickly new developers can catch these changes.
