# Domain-Oriented Scala Style Guide

What you will find below is not a typical style guide. If you're looking for a general resource of this kind, consider
the official [Scala Style Guide](https://docs.scala-lang.org/style/) or the
great [Scala Best Practices](https://github.com/alexandru/scala-best-practices) collection.

What you will find here is a collection of hints and advises on how to write your code in a way that brings the focus to
the business domain and problems being solved rather than technical aspects of it. Most of the points are opinionated
and hence highly questionable. **Use your own experience and judgment to decide which advice to follow.**

> [!NOTE]
> This document is new and minimal. Please help to make it better, contributions are much welcomed!

**Table of Contents**
<!-- TOC -->

* [Domain-oriented Scala Style Guide](#domain-oriented-scala-style-guide)
* [Rules](#rules)
    * [Organise code based on domains, not technical layers](#organise-code-based-on-domains-not-technical-layers)
    * [Model your domain](#model-your-domain)
    * [Separate domain problem from technical solution](#separate-domain-problem-from-technical-solution)
        * [Don't use FP primitives in the business interfaces](#dont-use-fp-primitives-in-the-business-interfaces)
        * [Don't use monad transformers in business interfaces](#dont-use-monad-transformers-in-business-interfaces)
    * [Define interfaces explicitly](#define-interfaces-explicitly)
    * [Express business logic through well-formatted for-comprehensions](#express-business-logic-through-well-formatted-for-comprehensions)
    * [Use suffix operators to highlight important bits over technical type adjustments](#use-suffix-operators-to-highlight-important-bits-over-technical-type-adjustments)

<!-- TOC -->

# Rules

## Organise code based on domains, not technical layers

Both ways are fine from the general standpoint, but one of them is better if we want to prioritize thinking about the
business domain.

```
// Wrong!
example/
├── repository/
│   ├── UserRepository.scala
│   ├── OrderRepository.scala
├── service/
│   ├── UserService.scala
│   ├── OrderService.scala

// Right!
example/
├── user/
│   ├── UserRepository.scala
│   ├── UserService.scala
├── order/
│   ├── OrderRepository.scala
│   ├── OrderService.scala
```

## Model your domain

`String` is not a domain concept. Avoid using simple types in your domain model. Instead, try to name them, especially
if they are used in a standalone way. Select a typesafety level and modeling approach that you're comfortable with: type
aliases, [opaque types](https://docs.scala-lang.org/scala3/book/types-opaque-types.html), case class wrappers,
[new-types](https://github.com/estatico/scala-newtype).

```scala 3
// Wrong!
case class Rectangle(width: Int, height: Int)

// Right!
case class Rectangle(width: Width, height: Height)
```

## Separate domain problem from technical solution

Business logic should be expressed in terms of concepts belonging to the domain and not in technical terms used to solve
a particular problem.

```scala 3
// Problem
arbitrageFinder.findArbitrage(rates)
// Solution
ratesGraph.findNegativeCycle

// Problem
sendUserNotification()
// Solution
sendEmail()

// Problem
processPayment()
// Solution
repository.updateAccountBalance()
```

### Don't use FP primitives in the business interfaces

Monoid is typically not a business-domain concept. While domain objects might form certain algebraic structures, this
should be an implementation detail, not a first-class property.

```scala 3
// Wrong
trait TaxCalculator {
  def calculateTax[T: Monoid](transactionAmounts: Seq[T]): Money
}

// Right!
trait TaxCalculator {
  def calculateTax[T <: Taxable](transactionAmounts: Seq[Taxable]): Money

  // Or
  def calculateTax[T: Taxable](transactionAmounts: Seq[Taxable]): Money
}

// Or 
trait TaxCalculator[T] {
  def calculateTax(transactionAmounts: Seq[T]): Money
}

class ProfitTaxCalculator extends TaxCalculator[Profit]
```

### What about `IO`?

It's also an FP concept that often leaks into the domain interfaces. Should we try to eliminate it as well?

We could try, but this becomes rather difficult or costly in practice. One of the ways to do it to parametrize
interfaces by `F[_]` but it comes with its own tradeoffs. So instead, let's stay pragmatic and allow IO as a special
case.

### Don't use monad transformers in business interfaces

Monad transformers are isomorphic to the underlying structure and bring only practical benefits but no conceptual
difference. Keep the signatures as simple as possible.

```scala 3
// Wrong!
trait NotificationService {
  def notifyUser(): EitherT[IO, UserDisabled, NotificationId]
}

// Right!
trait NotificationService {
  def notifyUser(): IO[Either[UserDisabled, NotificationId]]
}
```

## Define interfaces explicitly

All core business logic should be exposed through interfaces, even if there is exactly one implementation. The goal of
this is to make the interface explicit and bring focus to it. Such an interface can then be understood on its own and
analyzed for coherence or distance from the business domain. This applies also to static methods with no
dependencies (`object`s).

```scala 3
// Wrong!
class NotificationService() {
  def notifyUser()
}

// Right!
trait NotificationService {
  def notifyUser()
}

class NotificationServiceImpl extends NotificationService

// Wrong!
object ReferenceParser {
  def parse()
}

// Right!
trait ReferenceParser {
  def parse()
}

object ReferenceParser extends ReferenceParser
```

## Express business logic through well-formatted for-comprehensions

Sequential logic should read as a series of steps, with one line per expression and vertical alignment (if you can stand
it).

```scala 3
// Wrong!
for {
  _ <- if (amount > 0) IO.unit else IO.raiseError(InvalidAmount)
  transactionId <- transactionType match {
    case CryptoTransfer => sendThroughBlockchain()
    case FiatTransfer => sendThroughBank()
  }
  _ <- if (isUserTransaction) notifyUser(transactionId) else IO.unit
} yield ()

// Right!
for {
  _             <- validate()
  transactionId <- execute()
  _             <- notify(transactionId)
} yield ()


```

## Use suffix operators to highlight important bits over technical type adjustments

Code is read from left to right, and line should begin with the most important information. If the types need to be
adjusted in particular context use suffix operators (e.g., from cats syntax) or `pipe` method from the standard library.

```scala 3
// Wrong!
IO.pure(amount)
// Right!
amount.pure[IO]

// Wrong!
EitherT.fromEither[IO](validationResult)
// Right!
validationResult.pipe(EitherT.fromEither[IO])
```
