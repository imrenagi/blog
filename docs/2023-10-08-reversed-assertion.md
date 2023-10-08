---
title: "Reversed Assertion"
date: 2023-10-08
tags: "golang,testing"
---

# Reversed assertion

> Disclaimer: This `Reversed assertion` is not a common term used in testing. I just made it up to describe the technique I will explain below.

If you have been writing unit tests for a while, you might have seen this pattern:

```go
func TestIsDriving(t *testing.T) {
    mock := &MockSomething{}
    mock.On("StartAutopilot", mock.Anything).Return(true)

    car := NewCar(mock)
    err := car.Drive()

    assert.True(t, car.IsDriving())
    mock.AssertExpectations(t)
}
```

Test above is testing `Drive` function of `Car` struct. The `Drive` function will call `StartAutopilot` function of an interface. The `StartAutopilot` function will return `true` if the autopilot is started successfully. Later on we assert that `IsDriving` function of `Car` struct returns `true` and ensure that `StartAutopilot` from the mock function is called.

That is good. But, what if we want to test the opposite? What if we want to test when `StartAutopilot` never gets called when optional argument named `WithManual` to `Drive` function is given? We can do it like this:

```go
func TestIsDriving(t *testing.T) {
    mock := &MockSomething{}
  
    car := NewCar(mock)
    err := car.Drive(WithManual())

    assert.True(t, car.IsDriving())
    mock.AssertExpectations(t)
}
```

In this case `WithManual()` argument will cause `StartAutopilot` function to never get called. And how do we know that? If you are using mock, you probably know that `mock.AssertExpectations(t)` is your saviour here. That function will ensure that mock function will validate that all expected functions are not called because the `On()` function is not defined. If it get called otherwise, it will fail the test expecting the mock never get called.

Mock indeed simplify the test. But, what if you don't use mock? How do we test the opposite? 

Lets take different use case. The car now is equipped with capability to report car statistics to the server for analytics purpose. However, the owner can opt-out for this feature by providing a setting `WithDisabledAnalytics()`. When this option is given, the car should never make a call to the server. 
To test this we will need to create a fake or [hermetic server](https://testing.googleblog.com/2012/10/hermetic-servers.html) acting as the analytics server.

This is the normal case where the owner does not opt-out from analytics. Hence when the `Drive` is called, the car will report the statistics to the server by using the api client to make the call to the server.

```go
fakeServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte(`{"message": "analytics data is received"}`))
}))
defer fakeServer.Close()

client := api.NewClient(fakeServer.URL)

car := NewCar(client)
ok := car.Drive()
assert.True(t, ok)
```

Now, lets test the opposite. The owner opt-out from analytics. The car should never make a call to the server. How do we test this? 

Instead of checking that the car never make a call to the server, we should check the other way. If the server is called then we should immediately failed the test on the server handler. So that we know that the code should never reach out to the server at all. This is how we know that the hermetic server never gets called if the test pass.

```go
fakeServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    t.Log("server should never get called")
    t.Fail()
}))
defer fakeServer.Close()

client := api.NewClient(fakeServer.URL)

car := NewCar(client)
ok := car.Drive(WithDisabledAnalytics()) // opt-out from analytics
assert.True(t, ok)
```

What I meant by failing immediately in this case is not returning non-200 status code error to the client. Rather, we use `testing.T`'s utility to fail the test when the test reaches the server handler. This is how we know that the server should never get called.

You might be wondering what is the benefit of this approach compared to mock?

To me, it is more on the code clarity and readability. When you are using mock and reading the test, you can't see clearly when the test will fail since the failure is hidden behind `mock.AssertExpectations(t)`. To discover this behavior, you will have to look at the EUT (entity under test) code to see how it uses the mock. 

The other way I show later, it **explicitly** tells you that the test will fail when the server handler is called. You can see the failure immediately when the server handler is called. This is how I call it `Reversed assertion`. 

Thanks for reading!
