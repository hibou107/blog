---
title: Create a Validation DSL from scratch with scala
date: 2019-04-12T15:07:12+02:00
showDate: false
draft: false
tags: ["scala","fp", "validation"]
---
# Introduction

If I have to pick any subject to talk about functional programming, I will choose the validation. Any developers who do not need to validate some input data and provide all the errors associated with them?

It's a common mistake by junior developers to ignore the error handlings process. They take the correct input, test the program and are very happy when they see it works, then push to production.

The better way to handle the errors is to catch all possible errors in the main program. It's just not ideal because if you have a big configuration file with hundreds of parameters, then you have to correct one error after another. We want to see all the errors at once. Let's explore how functional programming techniques can solve this problem nicely.

# Simple implementation

In functional programming, we have to think about the type that describes best our problem. The `Try` or `Either` data type is not the best candidate because it can only carry one error value. So let's write our `Validation` data type.

```scala
sealed trait Validated[E, A]
case class Invalid(errors: List[E]) extends Validated[E, A]
case class valid(result: A) extends Validated[E, A]
```
We defined the data type `Validated` with two type parameters `E` and `A`. If it can be an invalid result that carries a list of errors of type `E`, and if it's a valid result, it carries the result of type A.

Wait a minute; there could be a problem here. We know that a `List` could be `Empty` or `having something`. In this case, there is a possibility that this value exists: `Invalid(Nil)` and it's very wrong. How on earth an invalid value that could not provide any error values? If there are no errors, then it must be a `Valid` value.

Let's quote from [Yaron Minsky](https://www.google.com)

> Making illegal states unrepresentable

Let's make a data type called `NonEmptyList` that can never be empty.

```scala
case class NonEmptyList[A](first: A, rest: List[A])
```

Another improvement to be considered is that we have to make the data type covariant by both `E` and `A` to accept any subtype of `E` and `A`

Here's the complete definition of our data type

```scala
case class NonEmptyList[+A](first: A, rest: List[A])

sealed trait Validated[+E, +A]
case class Invalid(errors: NonEmptyList[E]) extends Validated[E, A]
case class Valid(result: A) extends Validated[E, A]
```

Let's make an example to be easier to understand. Imagine we have an input in the form of `Map[String, String]` and we want to validate and convert it into a `Person`type defined as follow:

```scala
case class Person(name: String, age: Int)
```
We have two fields to parse from the `Map`. The first field is the `name` of type `String`. We have to define our exception data type. For the sake of example, we can specify only one error data type. 

```scala
case class ErrorParsing(message: String)

type Validation[A] = Validated[ErrorParsing, A]

```

Let's write our code to parse the `fieldName` String value from the `input`

```scala
def validateString(input: Map[String, String], fieldName: String): Validation[String] = {
    input.get(fieldName) match {
      case Some(v) => Valid(v)
      case None => Invalid(NonEmptyList(ErrorParsing(s"Field $fieldName does not exist"), Nil))
    }
  }
```

And another function to parse the `Int`value from the `input`

```scala
 def validateInt(input: Map[String, String], fieldName: String): Validation[Int] = {
    input.get(fieldName) match {
      case Some(v) => Try(v.toInt) match {
        case Success(int) => Valid(int)
        case Failure(exception) => Invalid(NonEmptyList(ErrorParsing(s"Cannot convert $v to String"), Nil))
      }
      case None => Invalid(NonEmptyList(ErrorParsing(s"field $fieldName does not exist"), Nil))
    }
  }
```

Now we have a value of `Validation[String]` and a value of `Validation[Int]`

```scala
def validatePerson(input: Map[String, String]): Validation[Person] = {
    val validatedName = validateString(input, "name")
    val validatedAge = validateInt(input, "age")

    (validatedName, validatedAge) match {
      case (Valid(name), Valid(age)) => Valid(Person(name, age))
      case (Invalid(nameError), Valid(_)) => Invalid(nameError)
      case (Valid(_), Invalid(ageError)) => Invalid(ageError)
      case (Invalid(errorName), Invalid(errorAge)) => Invalid(
        NonEmptyList(errorName.first, errorAge.first :: errorName.rest)
      )
    }
  }
```

This code does the job. But somehow we can see that this code is not reusable. If we have another case class called `Position(x: Int, y: Int)`, we have to rewrite the same code again and again.

This leads us to write a more generic function called validate2 that takes 2 validations with different types and a constructor for a third value.

```scala
def validate2[A, B, C](first: Validated[ValidationError, A], second: Validated[ValidationError, B])(f: (A, B) => C): Validated[ValidationError, C] = {
    (first, second) match {
      case (Valid(a), Valid(b)) => Valid(f(a, b))
      case (Invalid(firstError), Valid(_)) => Invalid(firstError)
      case (Valid(_), Invalid(secondError)) => Invalid(secondError)
      case (Invalid(firstError), Invalid(secondError)) => Invalid(
        NonEmptyList(firstError.first, secondError.first :: firstError.rest)
      )
    }
  }
```

# Introduction to cats

The function `validate2` is very similar to the function `product`defined in an applicative functor.

```scala
trait Applicative[F[_]] extends Functor[F] {
    def product[A, B](fa: F[A], fb: F[B]): F[(A, B)]
}
```
And this is way more generic than our code that only works with the type `Validated[ValidationError, A]`. This code can work with a lot of higher kind type that we already know (`Option`, `Either`, `Future`, `List`). Using a library like `cats` that defined all these functions is very convenient. Using the `cats` library, we can have automatically `map2`, `map3` and even `mapN`

Take a look at [validation in cats](https://typelevel.org/cats/datatypes/validated.html) and see the very convenient syntax to define a validation:

```scala
def validateForm(username: String, password: String, firstName: String, lastName: String, age: Int): ValidationResult[RegistrationData] = {
    (validateUserName(username),
    validatePassword(password),
    validateFirstName(firstName),
    validateLastName(lastName),
    validateAge(age)).mapN(RegistrationData)
  }
```

# Conclusion
In this article, I've introduced the needs and a simple implementation of the validation. We quickly arrive at the point that we need a more powerful abstraction to reuse code. This is how a generic library like cats or scalaz can shine.


