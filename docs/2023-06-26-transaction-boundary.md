---
title: "Understanding database transaction boundary"
date: 06-26-2023
tags: "go,database,pattern"
---

# Comparing alternatives for working with repository pattern

Last weekend I did live streaming to review [@lynxluna](https://twitter.com/lynxluna)'s take on module driven architecture. Understanding others perspective on how they approach problem is always interesting stuff that I really like to do. On this writeup, I might share few notes on his take on using pure function when defining repository. There are other interesting stuff being discussed there. You might want to check directly on [the repository](https://github.com/kadcom/mda) and on my [live stream](https://youtube.com/live/PxlHx6Whcic).

I still remember back then relatively junior engineer at work complaining about a modification done by a senior engineer. The junior came from this kind of practice when writing repository code:

```go
type Repository struct {
  DB *gorm.DB //db connection stored here
}

func (r *Repository) FindAll(ctx context.Context) ([]Object, error) {
  // somewhere use r.DB to run query the database
}
```

Here is sample of what senior engineer did. Instead of keeping the connection on struct, he used pure function and make the function accepts the db connection.

```go
package something

func FindAll(ctx context.Context, tx *gorm.DB) ([]Object, error) {
  // instead, this uses tx passed on the method arguments to make query to database
}
```

## Testability

Both approaches above, has no issue with testability. No matter whether you want to test it by creating mock with [go-sqlmock](https://github.com/DATA-DOG/go-sqlmock) or by running real db instance with [testcontainers](https://golang.testcontainers.org/), both approaches can be tested easily.

## Domain boundaries

This is where it gets more interesting. The second approach accepts `tx` as method arguments. This makes the caller has responsibility to supply the db connection or db transaction from the upper layer (e.g. use case layer or service layer). 

```go
package something

type Service struct {
  DB *gorm.DB  
}

func (s Service) ListSomething(ctx context.Context) ([]Object, error) {
  return FindAll(ctx, s.DB)
}
```

Boom! In this case we possibly leak the database implementation to the upper layer. Now the FindAll implementation is tightly coupled with the Gorm implementation. What if we want to use different sql library other than gorm? or even in more extreme condition what if we want to change the database implementation because we cant simply swap the implementation on the service layer anymore? Is it correct to take this?

That was confusing for the junior engineer at that time.

How often in your project that you have to change the database from e.g. MySQL to PostgreSQL or to NoSQL database? I bet mostly never in normal condition. Do we really need to prepare for future where we are going to support several types of database? Nah. I don't think so.

Are we leaking the database implementation to the upper layer (e.g. service layer)? May be. But to some extends it might be necessary for the service layer to have control over the transaction boundaries. Thus, it is better for the service layer to create the boundary itself. Let me give you example. Here is things that your service layer probably need to do transferring money from one user's account to another. Assume that you have a function named `Transfer()` whose following things to do:

1. Get user's balance. --> `repository.GetUserBalance(user)`
1. Check whether the amount is enough for transfer
1. Deduct the user's balance. --> `repository.UpdateBalance(user, new_balance)`
1. Increment destination's balance. --> `repository.UpdateBalance(user, new_balance)`

Imagine what happened if each query is using different transaction and being called from the service layer? If deduction is success, but updating destination's balance fails, it is hard to rollback the transaction entirely. This is where having service controls the database transaction boundary might be handy.

Instead of using different transactions when calling different repository functions, service layer can create a database transaction used within the `Transfer()` method. Rough example on how it looks:

```go
func (s Service) Transfer(to, from User, total int) error {
  return s.DB.Transaction(func (tx *gorm.DB) error {
      source := GetUserBalance(from, tx)
      //check amount
      destination := GetUserBalance(to, tx)
      //update balance

      err := UpdateBalance(from, amount-total, tx)
      if err != nil {
        return err
      }
      
      err = UpdateBalance(to, amount+total, tx)
      if err != nil {
        return err
      }
      return nil
  })
}
```

On the sample above, if the closure returns error, the transaction will be rolled back and committed otherwise. 

## Summary

Everything has tradeoff. Choose wisely. :p 

