---
title: Notes on How I Work with Error in Go
date: 05-07-2023
tags: go
---

# Notes on How I Work with Error in Go

This writing is just a note that I want to use to record my lesson learned in writing and handling error in Go. I hope many of you will find this notes useful.

## Propagating errors

As newbie writing Go, it feels very weird to me every time I return error as it is up to the call stack. But i did it anyway because it is a no brainer. lol

```go
var errUnexpected = errors.New("unexpected error")

func f2() error {
  return errUnexpected
}

func f1() error {
  if err := f2(); err != nil {
    return err
  }
  return nil
}
```

We learned from the mistake. And I think now doing it that way prevents us from knowing where the error is actually originating from. Thus, to prevent that, we can add context to the error path so that it is easier for use to search from the code when we get error.

```go
func f1() error {
  if err := f2(); err != nil {
    return fmt.Errorf("f2 call failed: %w", err)
  }
  return nil
}
```

Adding something like `f2 call failed:` to the error message above will probably help us in finding the source of later when our logging is not enough in detecting those.

Also, because go has introduced error wrapping since 1.13, we should always utilize that to wrap the error from the source so that we can check the sentinel error type later on if necessary. 

```go
func main() {
  ok := error.Is(err, errUnexpected)
  // ...
}
```

## Abusing sentinel error

I used to use common sentinel error to ensure that I can return proper error code for every http request received by the server. In doing so, I created a common package containing all of the sentinel errors which will be used by internal packages.

```go
package common

var ErrNotFound = errors.New("not found")
```

Later, I use this `common.ErrNotFound` almost everywhere in other package to tell this type of error is not found error which should be handled in a specific way by the client (e.g. to return 404 HTTP status code, etc.).

```go
package vehicle

func FindCarByID(id string) (Car, error) {
  var car Car
  err := d.db.Where("id = ?", id).Find(&car).Error;
  if err != nil {
    if errors.Is(err, gorm.ErrNotFound) {
      return fmt.Errorf("car %s: %w", id, common.ErrNotFound)
    }
    return err
  }
  // ... 
}
```

I use [gorm](https://gorm.io/) on the example above to query the database and detect whether data is available or not by checking `gorm.ErrNotFound`. If it is `gorm.ErrNotFound`, I changed the error with type that I have in common package so that I don't expose `gorm` dependency up to the call stack.

Now whenever I returned this error up to the call stack, the caller could utilize this to do specific error handling on their own. This is how I used to do it on the http handler.

```go
package server

func handleError(w http.ResponseWriter, err error) {
  switch {
    case errors.Is(err, common.ErrNotFound):
      return http.Error(w, err, http.StatusNotFound)
    // .. handle other common sentinel types
  }
}

func handler() http.HandlerFunc {
  return func(w http.ResponseWriter, r *http.Request) {
    err := callFunction(); err != nil {
      handleError(w, err)
    }
    // ... other things to do
  }
}
```

There are several issues worth to be discussed in this case.

Even when I'm not propagating `gorm.ErrNotFound` to the caller anymore, there is still issue where I add coupling or dependency between my domain and server package with my common error package.

On the handler side, as you have seen, I also had to check all sentinel errors extensively in switch block to ensure that sentinel error is mapped into correct http status code. 

And the worst case was I usually ended up writing many sentinel errors to and mapped them to propoer HTTP status code on the server package.

```go
var ErrNotFound = errors.New("not found")
var ErrUnprocessableEntity = errors.New("can't be processed")
var ErrInvalidArgument = errors.New("invalid argument")
var ErrBadRequest = errors.New("bad request")
// ... so on and so forth
```

Now, image the situation where I have to change my server from HTTP to gRPC. The error message wrapped on the domain package might looks weird  in gRPC response now because gRPC doesn't have something named UnprocessableEntity. Even though there is alternative that is almost similar `FailedPrecondition`, but that is mostly used for status code instead of error message.

### Use typed error for determining behaviour

So instead of checking the type of sentinel error with `errors.Is` or `errors.As`, it is better in my opinion by checking for specific behavior. In this case, we probably will have to create a type for the error and define a method determining the HTTP status code for the error.

```go
package common

type NotFoundError struct {
  Message string
}

func (e NotFoundEror) Error() string {
  return e.Message
}

func (e NotFoundError) HTTPStatusCode() int {
  return http.StatusNotFound
}
```

Now let's update the implementation of `FindCarByID()` above into:

```go
package vehicle

func FindCarByID(id string) (Car, error) {
  // ....
  if errors.Is(err, gorm.ErrNotFound) {
    return &common.NotFoundError{
      Message: fmt.Sprintf("car %s not found", id),
    }
  }
  // ....
}
```

More lines compared to sentinel? Exactly. But it gives you flexibility to define behavior for your error type. Imagine where you need to know whether an error can be retried or not. To achieve this, you can add method for that behavior.

```go
func (e NotFoundError) CanBeRetried() bool {
  return false
}
```

Now, instead of checking for the error type, we should check for the behavior of the error itself. This can be done by defining a local interface to get the status code from the error and cast the error to the interface.

```go
package server

type httpStatusCodeProvider interface {
  HTTPStatusCode() int
}

func errorCode(err error) int {
  code := http.StatusInternalServerError
  pr, ok := err.(httpStatusCodeProvider)
  if ok {
    code = pr.HTTPStatusCode()
  }
  return code
}

func handleError(w http.ResponseWriter, err error) {
  return http.Error(w, err, errorCode(err))
}
```

As you might see, from the caller point of view, there is no extensive switch case required anymore. I know some of you might say polymorphism or dynamic dispatch is slow, but that's out of scope for this writing. lol.

Now, whenever there is a case where you need to use this error for different representation layer such as gRPC, you know what you need to do. Yes, add another method to define the gRPC status code that has to be returned. Probably something like `GRPCStatusCode() codes.Code`.

However, there is still issue that I don't like from this implementation tho. Since this kind of generic error receives `Message` that gives flexibility to the client to supply the error message, I found this could lead us to different issue such as duplication and various error message. So how can we get better?

### Don't obscure the domain language

Writing several common typed errors used everywhere in your code might look simple and convenience because it is only written once and can be used anywhere. However, IMO it reduces the readability, possibly add more duplication and might hide important domain knowledge from your code or make it more obscure.

Refer to the `FindCarByID()` method above, we now return the `NotFoundError` type in order to ensure that the HTTP server can return proper status code (hopefully 404) to the HTTP client. Even though we can easily notice that this is error related to not found car by reading the error message, it is hard for the code itself to know which entity this error is related with.

Thus, instead of using common package as mentioned above, I prefer to use domain specific error type to give information about the error.

```go
type CarNotFoundError struct {
  ID string
}

func (e CarNotFoundError) Error() string {
  return fmt.Sprintf("car %s is not found", e.ID)
}

func (e CarNotFoundError) HTTPStatusCode() int {
  return http.StatusNotFound
}
```

You might have realized that there is still possibility of duplicate code if other entities also has their specific not found error type. Yes. Software engineering is always about trade off. However, in my opinion, now it is clear now what this error is about and what behavior each error can have differently.

This approach gives you flexibility to determine what the behavior of your domain typed error. For instance, even though the `CarNotFoundError` might be translated to 404 HTTP status code, type like `PoliceNotFoundError` might be translated to different status code like 200. (I know it is considered normal in some countries if police is missing when they are needed the most lol /s). So this approach help you to bring more domain ubiquitous language to the code and make it more readable and maintainable.

Another benefit is that you can tailored the arguments required to construct the error message specifically instead of accepting only generic argument like string `Message`. On the above example, since we can set `ID` property, we are know able to change the error message easily without a need to update the source of the error. Imagine where you can pass other objects to the error and use their internal property to construct the error message. This will removes the duplication from where the errors are originating from.

## References:
* [Don’t just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully) by Dave Cheney. 
