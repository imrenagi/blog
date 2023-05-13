---
title: "Refactoring: Thinking About Problem Domain"
date: 05-13-2023
tags: "go,refactor"
---

# Refactoring: Thinking About Problem Domain

I really like reading people's code. It helps me understand how people approach the problem and increases my critical thinking ability. How they can come up with this approach and why they did it this way in the first place? Since I'm one of people defining the requirements for internal product we are developing internally, understanding the way people solving the problems helps me to know what could be missing or unclear from the requirements or technical direction I defined as the tech lead.

As a context, I'm currently working on automation platform where we want to provide functionality for the developers to perform automatic upgrade for their database. **One of the requirements is that user should be able to initialize upgrade for a single instance of postgresql. The new version given must be greater than the current version of the postgresql node.**

To give you idea about the entities, we have something similar (but not exactly the same) to this:

```go
type Version int

// Instance is representation of one postgresql node. It could be either primary or replica instance.
type Instance struct {
  // ... other attributes
  Version Version
}

// Upgrade keeps all information related to upgrade
type Upgrade struct {
  // ... other attributes
  PrevVersion Version
  NewVersion  Version
  Instance    *Instance
}
```

Iterating the requirement once again:

> User should be able to initialize upgrade for a single instance of postgresql. The new version given must be greater than the current version of the postgresql node.

Since we know that user should be able to create new upgrade, it is pretty easy to decide that we should create a new constructor to construct the `Upgrade` instance and returned error when the given version is not valid. Hence, this is what I found on the implementation later on:

```go
func NewUpgrade(instance *Instance, oldVersion, newVersion Version) (*Upgrade, error) {
  if oldVersion == 0 || newVersion == 0 {
    return nil, fmt.Errorf("version invalid, version must be 9 - 15")
  }
  if newVersion <= oldVersion {
    return nil, fmt.Errorf("new version must be greater than %d", oldVersion)
  }

  pgUpgrade := &Upgrade{
    Instance: instance,
    PrevVersion: oldVersion,
    NewVersion: newVersion,
  }
  return pgUpgrade, nil
}
```

The code above looks correct, right? Yes it is. It passed all of the test cases defined when the pull requests was submitted. But, how can we get better? (fyi. I like to ask this kind of question over and over again when I write code. lol I never get satisfied easily. :p)

What I found most of the time during my career as software engineer is that people tend to put logic too far away from where it is supposed to be. The sample code above IMO is just a normal constructor. But what is hidden from the implementation about is about the domain problem. Let's reiterate the requirement once again. 

> User should be able to initialize upgrade for a single node of postgresql. The new version given must be greater than the current version of the postgresql node.

If we take time to think carefully about the requirement, we can spot domain language here: **initialize upgrade for a single instance of postgresql**. This means an instance of postgresql should be able to initialize the upgrade on its own. How this can be translated to the implementation? Remember! We have `Instance` entity defined earlier. This entity is not supposed to be used to only keep data. It may has logic and keep business rule as well and **it must!**.

So initially what we can do to make the code better is to move the `NewUpgrade` function as method of `Instance` entity. After I moved the previous test cases to test this method instead, I should be able to get the same result and ensure all test are passed.

```go
// InitializeUpgrade create new instance of upgrade if it is valid
func (i *Instance) InitializeUpgrade(toVersion Version) (*Upgrade, error) {
  if i.Version == 0 || toVersion == 0 {
    return nil, fmt.Errorf("version invalid, version must be 9 - 15")
  }
  if toVersion <= i.Version {
    return nil, fmt.Errorf("new version must be greater than %d", i.Version)
  }

  pgUpgrade := &Upgrade{
    Instance: i
    PrevVersion: i.Version,
    NewVersion: toVersion,
  }
  return pgUpgrade, nil
}
```

The implementation is actually the same. But what are the benefits?

* You are now able to see that the method name now is clearly describe what the requirement is. Instead of using `NewUpgrade`, now we know that `Instance` has capability to initialize an new instance of `Upgrade`. 
* You are now able to see that we are now accepting less function arguments as before. It was 3 before and now we only have 1: `toVersion Version`. This benefit is something that people most of the time are not aware of. While an entity probably has all information it needs to perform and operation, why does it has to pass it to other function while it actually can do it itself? Remember something about message passing in object oriented paradigm class?
* This help us in doing more refactoring. How? Let's see.

Let's iterate the requirement once agan for the last time only. :p

> User should be able to initialize upgrade for a single node of postgresql. The new version given must be greater than the current version of the postgresql node.

Again, if you pay attention. There is another domain rules mentioned here. **The new version given must be greater than the current version of the postgresql node.** . I hear somebody shouting for far "Yes we have `Version` entity!". You are correct! 

Instead of doing the validation on the `Instance.InitializeUpgrade()`, why don't we delegate this to this to the entity which must own this logic?

One way we could approach this is by adding a method called `CanBeUpgraded`. Why not just using `IsValid` as the name? Yeah you may. But how the `IsValid` name can describe about your domain problem? The problem now is that we need to know whether a `Version` can be upgraded to a certain `Version` or not. You may have other reasoning. The key is as long as it easily describes your domain, that is fine.

```go
// CanBeUpgraded checks whether versions are valid and the target version is greater than the current version
// than this version
func (v Version) CanBeUpgraded(to Version) error {
  if v == 0 || to == 0 {
    return nil, fmt.Errorf("version invalid, version must be 9 - 15")
  }
  if to <= v {
    return nil, fmt.Errorf("new   version must be greater than %d", v)
  }
  return nil
}
```

Thus, we can also do modification to the `InitializeUpgrade`.
```go
func (i *Instance) InitializeUpgrade(toVersion Version) (*Upgrade, error) {
  err := i.Version.CanBeUpgraded(toVersion)
  if err != nil {
    return nil, err
  }
  // ... removed for brevity
}
```

I hope it is clear now why doing this can increase the readability and help the future you understand how the implementation relates with the requirement or domain problem.

Another benefit I would like to mention here is that by having this implementation, you are now able to tackle new requirement coming easily and do it in better function.

Let's imagine a imaginary requirement such as:

> As a API user, I should be able to receive 500 error when existing instance version is invalid, and 400 when user input version is invalid.

What can we do? We can easily modify the `Version.CanBeUpgraded` method to return the respective error (possibly with domain error too). All we need is only adding unit tests to this function. Since other function is untouched, we don't have to do anything.

Let me give you brief sample:

```go
func (v Version) CanBeUpgraded(to Version) error {
  if v == 0 {
    return &InvalidVersionError{
      Message: fmt.Sprintf("postgresql version %d is not recognized", v),
      Internal: true
    }
  }
  if to == 0 {
    return &InvalidVersionError{
      Message: fmt.Sprintf("%d is not recognized", to),
    }
  }
  if to <= v {
    return &InvalidVersionError{
		Message: fmt.Sprintf("target version must be higher than %d", v),
    }
  }
  return nil
}
```

As you can see. The `InvalidVersionError` in this case has `Internal` attributes that we can use possibly later to decide which http status code we should define. Since HTTP is not the domain language, but more like the adapter we use to interact with outside world, I will not discuss it on this write up. But if you are interested, you can get idea on how we can do it from my [previous article about error handling](./05-07-2023-error-handling.md).
