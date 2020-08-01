---

title:  Dependency Injection - What, Why and How?
date:   2017-03-20
categories: Design-Patterns
tags: [iOS, Swift]

---

We are going to explore dependency injection with emphasis on swift iOS development. But the concept
applies to most object oriented languages. We will also see some practical considerations on 
applying DI in iOS environment. This article is result of my deep dive into implementing DI, and learning about 
various practical, theoretical aspects of it.


# What is Dependency Injection?


>"Dependency Injection" is a 25-dollar term for a 5-cent concept".

is a often repeated maxim regarding DI.

In its essence DI means wherever possible, **replace object creation inside a piece of code 
by providing it from outside that piece of code**. Hence the term.

Constructor, Property, Method are where we usually do object creation, and that can be replaced from outside. So we end up with 3 types of injection.

1. Constructor Injection
2. Property Injection
3. Method Injection

We will explore they 3 types in "How ..?" section.


# Why Dependency Injection?

Let's dwelve into this with help of a scenario.

## Scenario - Koala Koder!


Say you have an app with a user login. A user model struct/class  encapsulates the logged in user data.

Suppose you are a Koala Koder( a programmer who is as lazy as a koala bear). You perhaps made a solution which is quick and dirty. Store the user model inside NSUserDefaults, and fetch it via properties. And as we all know how apple loves its singleton classes, we follow them making UserModel a singleton.

```swift
class UserModel {

    static let sharedInstance = UserModel()

    var name: String {
        get {
            return NSUserDefaults.standard.string(forKey: "userName")  ?? ""
        }
        set {
            NSUserDefaults.standard.set(newValue, forKey:"userName")
        }
    }

    func greet() -> String {
        return "\(name), Vannakam :)"
    }
}

```

We write our app with this usermodel in mind, and `UserModel.sharedInstance` is everywhere in our app.

![Koala lazing off](https://ih0.redbubble.net/image.5150955.4141/flat,1000x1000,075,f.jpg)


## Problems, problems everywhere...

### 1. Unit Tester Vader strikes!

A senior developer suddenly turns to the dark side, and starts ranting about *unit testing*.
He/She will not let apps which are not unit tested pass the code review. 

![Meme: Darth Vader says "I find your lack of unit testing disturbing"](http://s2.quickmeme.com/img/03/0347c3efdc17cc1959d089f60b8b2fc267d9093caa8e8cb483bf476b58e63e45.jpg)

### 2. A wild New Use case appears!

And if that isn't enough a *new use case*  should be supported. Our app should now support *multiple users*.

![Meme: Back to future - Dr Brown says "New Usecase in no time? I've an extra flux capacitor"](https://cdn.meme.am/instances/500x/64312241.jpg)

### 3. "Lets move to \<insert any serialization library here\>"

Now we also reached a point where userdefaults didn't scale and  wish to migrate to new data serialization method. Now even our getters and setters inside the UserProfile is not safe.


## Why are there problems?


So we find ourselves in deep trouble. Lets analyze why so.

### 1. Singletons are hard to test

Now if you want to unit test a viewcontroller that uses this singleton.

```swift 

class UserProfile: UIViewController {

    func viewDidLoad() {
        greetingLabel.text = UserModel.sharedInstance.greet()
        //...
    }
}

```

In unit testing you create a UserProfile object, call viewDidLoad or other methods manually. Now you have to verify 
whether `UserModel.sharedInstance.greet()` was called.

Since UserModel.sharedInstance is immutable, either we can't replace it with our mock class extending UserModel, which overrides greetUser, and sets a flag which can be checked. So our testing coverage comes down.


### 2. Instantiation inside our code constraints creates strong coupling, creating harmful constraints

In our scenario, by using a singleton, we tied our codebase to a single UserModel but now our app needs multiple user models. So in general, using singletons will make it hard to adapt to new use cases. *What you thought as a singleton suddenly is not so single anymore*.

 But consider that we didn't use singleton, but instantiated UserModel, by calling `UserModel()` wherever we needed it.

```swift 

class UserProfile: UIViewController {

    let userModel = UserModel()

    func viewDidLoad() {
        userNameLabel.text = userModel
        //...
    }
}

```

We can't support our new usecase of multiple users, without userprofile class knowing about multi-usermodel, or some other global allowing UserModel() to return the correct usermodel, these solutions are ugly hacks, which add complexity by either giving too much knowledge to classes or using globals and forgoing object oriented encapsulation.


### 3. Concrete Type usage creates strong coupling  

Consider the UserProfile, it  uses NSUserDefaults, now suppose we move to coredata to save our data, we are again in trouble because of using singleton inside. **Our UserProfile rather needed only just a way to serialize some data, it didn't need to know about WHAT we use for serialization**. This is the key insight to keep in mind  when thinking about dependency injection.


## DI to the rescue 

DI helps to solve the variety of issues that we face above, with regard to  Unit Testablity, Singletons and strong coupling we face above.


# How to do Dependency Injection?

## Constructor Injection

So we will be injecting a serializer into UserModel via constructor.

**Move dependency to constructor, and if possible make it a interface/protocol type instead of concrete class/struct**

So we remove singleton access to userdefaults. And rather pass a serializer protocol which has methods we require for serialization to constructor.

```swift

protocol Serializing {
    func string(forKey defaultName: String) -> String?
    func set(_ value: Any?, forKey defaultName: String)
}

class UserModel {
    static let serializer: Serializing

    init(_ serializer: Serializing) {
        self.serializer = serializer
    }

    var name: String {
        get {
            return serializer.string(forKey: "userName")  ?? ""
         }
        set {
            serializer.setObject(newValue, forKey:"userName")
        }
    }

}

```

Since we are Koala Koder, we just make the Serializing protocol methods to same ones in NSUserdefaults, so we can make NSUserDefaults conform easily by 

```swift
extension NSUserDefaults: Serializing {}
```
.

Now to use user defaults as our serializer, we do the following.

```swift
UserModel(serializer: NSUserDefaults.standard)
```

For our fancy multiple user use case we can also do this. (Note: did this in a hurry, it may have edge cases, just providing it as a illustration)

```swift

struct MultiUserSerializer: Serializing {

    let serializer: Serializing
    init(_ serializer: Serializing) {
        self.serializer = serializer
    }

    // So we fetch the current currentUserId and use that to prefix stored data
    private func fetchCurrentUserId() -> String {
        return serializer.string(forKey: "CURRENT_USER_ID") ?? "0"
    }

    func string(forKey defaultName: String) -> String? {
        return serializer.string(forKey: fetchCurrentUserId() + ":" + defaultName)
    }

    func set(_ value: Any?, forKey defaultName: String) {
        return serializer.set(value, forKey defaultName: fetchCurrentUserId() + ":" + defaultName)
    }

}

//Instantiate using userdefaults, assume we implemented contextFetcher somewhere else

let multiUserSerializer = MultiUserSerializer(serializer: NSUserDefaults.standard)

let userModel = UserModel(serializer: multiUserSerializer)

```

So now userModel fetches data, and sets data to the current userID without even knowing about it. We could also use something other than `NSUserDefaults.standard` to serialize the data in top level.

The point is by removing replacing the concrete  dependency `NSUserDefaults.standard` out of `UserModel` and swapping it to `Serializing` protocol we can now satisfy the new usecase of multi user modelling easily. This is the core idea behind of **loose coupling**.

Also we can now unit test User Model by just passing a Mock implementation of `Serializing`


### Dependency injection works even with only concrete types.

Sometimes you don't have the time, or are  sure that concrete type used will not have to be changed. You can still DI the concrete type without bothering with protocol creation, just for the sake of it.

#### For example:

```swift 

class UserProfile: UIViewController {

    let userModel: UserModel

    // This works only if you don't use storyboards
    init(userModel: UserModel) {
        self.userModel = userModel
    }

    func viewDidLoad() {
        userNameLabel.text = userModel.name
    }
}

```

We  move UserModel creation out of the controller, without creating any protocol for UserModel properties.

We still gain advantages of unit testability using ordinary mock objects, and also we are free to use our multi user serialized UserModel , hence making UserProfile support multi user model without changing any logic in user profile.

(But if you have to do something to notify changes in data, that is seperate topic)

```swift
let multiUserSerializer = MultiUserSerializer(serializer: NSUserDefaults.standard)
let multiUserModel = UserModel(serializer: multiUserSerializer)
let userProfile = UserProfile(userModel: multiUserModel)
```

### My constructor is a monster now :(

A common problem which you will come across is, your UIViewController (if you don't use storyboards) or any class where you do DI becomes a to big.

```swift
class UserProfile: UIViewController {

init(userModel: UserModel, apiService: APIService, x: XService, y:YService, z: ZService, ....) 
```

This is a good thing, it points out clearly that your class is violating **Single Responsibility rule** . Single responsibility rule states that a  class should have singe responsibility. 

We can solve this by moving some of the current dependencies to a new class, and pass the new class as dependency to UserProfile.

Guess what if we do this we properly we would be re-inventing design patterns l
MVVM(Nodel View ViewModel) or MVP. Whew! DI just solved Huge ViewController problem!!.

> DI can easily help refactoring code to better design, in a gradual manner.


## Property Injection

We seemed to have solved all our above problems, why bother with more types of injections? 

> If you don't control the creation of a object, your best bet is property injection. But prefer constructor injection if that's not the case.

Sometimes you don't create objects of classes, some messy framework does it. For example, if you use Storyboards, you can't do stuff like `let userProfileViewController = UserProfile(multiUserModel)`. You would have to refactor to something like


```swift 

class UserProfile: UIViewController {

    var userModel: UserModel!

    func viewDidLoad() {
        userNameLabel.text = userModel.name
    }
}

let multiUserSerializer = MultiUserSerializer(serializer: NSUserDefaults.standard)
let multiUserModel = UserModel(serializer: multiUserSerializer)

let storyboard = UIStoryboard(name: "main", bundle: nil)
let userProfile = storyboard.instantiateViewController(withIdentifier: "UserProfile") as! UserProfile
userProfile.userModel = multiUserModel

```

By using implicitly unwrapped Optional property, and setting it from outside, we achive the same effects of constructor injection. But is not perfect, as `userModel` property can be mutated from outside. Butthis is as good as it can get.


## Method Injection

Method injection is just replacing instantiating inside method by one of its parameters.

```swift
// Before injection

func someMethod() {
    let x = X()
    let y = x.something() + 10
    return y
}

// After injection

func someMethod(_ x: XService) {
    let y = x.something() + 10
    return y
}

```

## Runtime Injection - Factory pattern

Sometimes you want to create a object of a particular class in runtime. But you want to use only protocol type(or a super type) instead of actual implementation (or subclass). Let's say you need a networking service, which you set based on a user action, but as you are going to need it in runtime, you may think that you can't inject it. 

But what you can do is inject a factory Networking object or closure, that constructs it in runtime. This factory can fill the dependency of the concrete Networking class, so your class can be unaware of this.


``` swift

protocol Networking {
 // Some methods ..
}

class ProxyNetworking: Networking  {
    init(a: A) {

    }
}

class NormalNetworking: Networking {
    init(a: A) {

    }
}


// Injection Via Factory Class

class NetworkingFactory {

    let a: A
    init(a: A) {
        self.a = A
    }

    func create(_ withProxy: Bool) -> Networking {
         if (withProxy) {
            return ProxyNetworking(a: a)
         } else {
            return NormalNetworking(a: a)
         }
    }
}


class Some :UIViewController {

  // This has to be injected via property injection
    var networkingFactory: NetworkingFactory! 

    var networking: Networking?

    @IBOutlet weak var proxySwitch: UISwitch!

    @IBAction func userTappedSubmit(sender: UIButton) {
        networking = networkingFactory.create(withProxy: proxySwitch.isOn)
    }
}


// Injection using closure

class Some :UIViewController {

    // This has to be injected via property injection
    var networkingFactory: ((Bool) -> Networking)!

    var networking: Networking?

    @IBOutlet weak var proxySwitch: UISwitch!

    @IBAction func userTappedSubmit(sender: UIButton) {
        networking = networkingFactory(proxySwitch.isOn)
    }
}


```

# Dependency injection - Containers & Frameworks

There are Dependency Injection frameworks that make the job of dependency injection easier. You may say, "whoa Sai! wait,Do we really need a dependency injection framework as a dependency? Can't it be done ".

When we examine what we are doing with DI, we are building a graph with our concrete types as nodes, and their dependencies linking them. If we do this completely, all dependencies will originate from a root object. 

Without a framework, we will be doing a lot of copy paste coding. If our app uses networking protocol type in multiple areas, we have to type out the same concrete implementation everywhere, and fill out every dependency of networking class everywhere.


For example:

```swift
 class A { 
    var propertyInjectionVar :V!

    init(networking: Networking, ...) 
  }

 // To create A we have to create networking and also fill out any property injection vars it needs

 class NetworkService: Networking { init(x: X, y: Y, z: Z) {} }

 // Now networking will inturn have its dependencies , which inturn have more ..
 // So to create A, You have to create NetworkService, X, Y, Z 

A(networking NetworkService(x: X(), y: Y(), z: Z()))

```

To make this process easier we can build something to store list of dependency type, and their concrete implementation. We will have a table of mappings.


| Dependency Type  |  Implementation type   |
| -------------    | ------------- 			|
| Networking       | NetworkService 		|
| X	    		   | XService			    |
| Y	    		   | YService			    |
| Z	    		   | ZSubClass			    |
| A                | A                      |

We can register a protocol Dependency Type to concrete implementation, like Networking, X, Y to NetworkService, XService, YService respectively. Or map concrete type to its concrete implementation which can be exactly same type, like A, or subclass like mapping Z to ZSubClass. Now what the container does is when A has to be created, it auto resolves each dependency of A from the table.

You can either implement a container, or use a DI Framework which does that for you.

Containers also allow you to autofill property injection, handle lifecycle of dependencies like marking them as singleton, so that your whole container has only one object of that type created and other fancy features to make your life easy.

> ### A key point to remember is that your classes should not depend on DI framework container ie its not good idea to pass the container to your class as a dependency.


## [Dip Framework - Swift](https://github.com/AliSoftware/Dip)

Among the DI frameworks exiting now for swift, I recommend Dip. Dip has some nifty features to make your DI pain free.

### Features of Dip

- **Scopes**. Dip supports 5 different scopes (or life cycle strategies): _Unique_, _Shared_, _Singleton_, _EagerSingleton_, _WeakSingleton_;
- **Auto-wiring** & **Auto-injection**. Dip can infer your components' dependencies injected in constructor and automatically resolve them as well as dependencies injected with properties.
- **Resolving optionals**. Dip is able to resolve constructor or property dependencies defined as optionals.
- **Type forwarding**. You can register the same factory to resolve different types implemeted by a single class.
- **Circular dependencies**. Dip will be able to resolve circular dependencies if you will follow some simple rules;
- **Storyboards integration**. You can easily use Dip along with storyboards and Xibs without ever referencing container in your view controller's code;
- **Named definitions**. You can register different factories for the same protocol or type by registering them with [tags]();
- **Runtime arguments**. You can register factories that accept up to 6 runtime arguments (and extend it if you need);
- **Easy configuration** & **Code generation**. No complex containers hierarchy, no unneeded functionality. Tired of writing all registrations by hand? There is a [cool code generator](https://github.com/ilyapuchka/dipgen) that will create them for you. The only thing you need is to annotate your code with some comments.
- **Weakly typed components**. Dip can resolve "weak" types when they are unknown at compile time.
- **Thread safety**. Registering and resolving components is thread safe;
- **Helpful error messages and configuration validation**. You can validate your container configuration. If something can not be resolved at runtime Dip throws an error that completely describes the issue;

### Annotations in Dip

Java has good frameworks like [Dagger](https://square.github.io/dagger/), and [Guice](https://github.com/google/guice) that use annotations to make the job even simpler compared to swift. Dip allows you to leave annotations of dependencies in comments and also generate code for DI from it. How cool is that?


## Poor man's DI in Swift

Though I recommend the framework approach, supposing you don't want to use framework initially and still want the to do DI, Fear not. You can use Swift's default parameters to do constructor/method injection, and just use variable properties for property injection. You can maintain a Seperate DI singleton and fill dependencies using that.


```swift


class UserModel {
    static let serializer: Serializing

    init(_ serializer: Serializing = PoormanDIContainer.instance.getSerializer()) { // Poor man DI
        self.serializer = serializer
    }

    var name: String {
        get {
            return serializer.string(forKey: "userName")  ?? ""
         }
        set {
            serializer.setObject(newValue, forKey:"userName")
        }
    }
}


class PoorManDIContainer {
    let instance = PoorManDIContainer() 
    func getSerializer() -> Serializing {
        // If Serializer has some dependencies, it will again use default constructor to obtain it from PoorManDIContainer
        return Serializer() 
    }
}


```

But if you use the above method, beware of circular dependencies.

# Conclusion

So in conclusion DI is great. It allows you to progressively make your code better remove  singletons, make your code modular, testable and also allow you to evolve good design patterns. Try it out in your existing code base, it will be one of the easiest way to refactor legacy OOP code, without modifying internal logic initially.If you have any doubts, suggestions, constructive criticisms, comment below.
