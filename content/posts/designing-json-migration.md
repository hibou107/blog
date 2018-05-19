---
title: Designing a json migration tool
date: 2017-12-20
tags:
    - scala
    - json
---

# The problems

Json is probably the most popular data format today. It's used both for data exchange (between front-end and backend for example). It's also used to store inside database like `MongoDB`, even [`postgresql`](https://www.postgresql.org/docs/9.3/static/functions-json.html) supports json data type.

Working with `json` means defining 2 functions:

* A Writer that converts an object into `json` format
* A Reader that reads a `json` and produces an object

An example of this use case is:

```scala
case class Model(x: String, y: String)
```

In [`play-json`](https://github.com/playframework/play-json), we can define a `Format` and uses the macro helper to automatically define a `Reader` and a `Writer` of `Model`

```scala
implicit val modelFormat: Format[Model] = Json.format[Model]
```

This is an example value:

```json
{ “x”: “toto”, “y”: “tata” }
```

Everything works perfectly and now the json data is persisted in database and test folder.

One day the client want to add one more field in the model, the new model should be:

```scala
case class Model(x: String, y: String, z: Int)
```

We don't need to modify the `Format` because the macro takes care of it automatically.

But what about the old json format? The current format cannot read the old json data because it lacks `z`

Now we should find a solution in order to be able to read the old json format by setting the `z` to a default value

# First solution

We've found a quick solution to the problem. We can for example define a `Reader` that can read both current and old version.

The first thing we should do is to split the `Format` into `Reader` and `Writer`:

```scala
implicit val modelReads: Reads[Model] = (
  (JsPath \ "x").read[String] and
  (JsPath \ "y").read[String]
)(Model.apply _)

implicit val modelWrites: Writes[Model] = (
  (JsPath \ "x").write[Double] and
  (JsPath \ "y").write[Double]
)(unlift(Model.unapply))

```

Then we define two versions of the `Reader`:

```scala
val modelReadsV0: Reads[Model] = (
  (JsPath \ "x").read[String] and
  (JsPath \ "y").read[String] and
  Reads.pure(0)
)(Model.apply _)

implicit val modelReadsV1: Reads[Model] = (
  (JsPath \ "x").read[String] and
  (JsPath \ "y").read[String] and
  (JsPath \ "z").read[Int]
)(Model.apply _) orElse modelReadsV0

```

The system works like that: the implicit readers is always in the last version. If the `ReaderV1` can not read the `json`, its used the `ReaderV0` which defines a default value for the field `z`

But the problems of this design are:

* If we add or remove a field, we need to update all the readers
* Do not work in more complicated cases: move a field to a nother place, rename a field, change value of a field
* Splitting `Format` to `Reader` and `Writer` can cause a Reader/Writer mismatch. We need to add a lot of tests 

# Second solution

## Imperative vs Functional

The solution is to clone the SQL migration design: Each migration is defined by a script.

The question is how we define this migration script. The `JsValue` is immutable and creating new value from the old one is tedious. I am a fan of functional programming but this is where the imperative way is much more easy. For example 

Imperative way | Functional way
---|---
`x.y.z += 1`   |`x.copy(y = x.y.copy(z = x.y.z + 1))`
`value[“key25”] = {“key251”: value[“key2”][“key21”]}` | `(__ \ 'key25 \ 'key251).json.copyFrom( (__ \ 'key2 \ 'key21).json.pick )`

I write the library for a team of both functional and less functional programer in the team. It should be easy to write migration script and the functional way is very hard for this task. Moreover, for the advanced task it's near impossible to do. For example, let's say we want to modify all the `JsObject` that has the field `toto` and change the field value to 0. An example of this json value is:

```json
{
“X”:  { “toto”: 0 },
“Y”:  [{“toto”: 1}, {“tata”: 2} ],
“Z” :  { “zz”: {“toto”: 2}}
}

```

The value that we want to modify can be inside a field, nested in 2 levels fields or even inside an array. Updating these values in a functional way is hard and the only way I've found is using advanced concept like [the zipper](https://en.wikipedia.org/wiki/Zipper_(data_structure))

All the difficulties lead to the obvious solution: convert temporary immutable value into mutable version, applying changes and convert back to the original value

## Wrapper for mutable version

Let's define a `trait` for this new data struture:

```scala
sealed trait JsValueWrapper
case class JsObjectWrapper(value: collection.mutable.Map[String, JsValueWrapper]) extends JsValueWrapper
case class JsStringWrapper(value: String) extends JsValueWrapper
case class JsArrayWrapper(value: ArrayBuffer[JsValueWrapper]) extends JsValueWrapper
case class JsBooleanWrapper(value: Boolean) extends JsValueWrapper
case class JsNumberWrapper(value: BigDecimal) extends JsValueWrapper
case object JsNUllWrapper extends JsValueWrapper
```

This build an equivalent of all the possbile values of `JsValue`. The only difference is the `JsObjectWrapper` and `JsArrayWrapper` use mutable collection internally

## Conversion functions

Let's define 2 methods to convert between `JsValue` and `JsValueWrapper`. Of course there are some recursivities when dealing with `JsObject` and `JsArray`

```scala
implicit def create(input: JsValue): JsValueWrapper =
   input match {
     case x: JsObject    => JsObjectWrapper(collection.mutable.Map(x.value.mapValues(create).toSeq: _*))
     case x: JsArray     => JsArrayWrapper(ArrayBuffer(x.value: _*).map(create))
     case x: JsString    => JsStringWrapper(x.value)
     case x: JsBoolean   => JsBooleanWrapper(x.value)
     case x: JsNumber    => JsNumberWrapper(x.value)
     case JsNull         => JsNUllWrapper
   }



implicit def toJson(input: JsValueWrapper): JsValue = {
input match {
    case x: JsObjectWrapper    => JsObject(x.value.map { case (name, value) => (name, toJson(value)) }.toSeq)
    case x: JsArrayWrapper     => JsArray(x.value.map(toJson))
    case x: JsStringWrapper    => JsString(x.value)
    case x: JsBooleanWrapper   => JsBoolean(x.value)
    case x: JsNumberWrapper    => JsNumber(x.value)
    case JsNUllWrapper         => JsNull
}
}

```

## Script

Let's define a `trait` that defines a method `migrate` that make in place modification

```scala
trait JsonMigrator {
 def migrate(input: JsValueWrapper): Unit
 def transform(input: JsValueWrapper): JsValueWrapper = {
   migrate(input)
   input
 }
}
```

We should define a way to combine multiple scripts to form a global script. This operation is called `append` and if we can define a neutral script (a script that does nothing), then we can define a `Monoid` instance

```scala
implicit val monoid: Monoid[JsonMigrator] = new Monoid[JsonMigrator] {
   def zero: JsonMigrator = (_: JsValueWrapper) => ()
   def append(f1: JsonMigrator, f2: => JsonMigrator): JsonMigrator = {
     (input: JsValueWrapper) => {
       f1.migrate(input)
       f2.migrate(input)
     }
   }
 }

```

## Some helpers

We should create some conversion implicit to help our users write more concise migration code. But it's not a safe operation because the `asInstanceOf` can throw exceptions

```scala
implicit class JsObjectWrapperConverter(input: JsValueWrapper) {
   def apply(field: String): JsValueWrapper = input.asInstanceOf[JsObjectWrapper].value(field)
   def number: BigDecimal = input.asInstanceOf[JsNumberWrapper].value
   def map: mutable.Map[String, JsValueWrapper] = input.asInstanceOf[JsObjectWrapper].value
   def setDefault(field: String, value: JsValueWrapper): Unit = {
     if (!has(field))
       input.asInstanceOf[JsObjectWrapper].map.update(field, value)
   }
   def remove(field: String): Option[JsValueWrapper] = {
     input.asInstanceOf[JsObjectWrapper].map.remove(field)
   }

```

## Examples

Let's say we have 3 migrations:

```scala
private val migrator1 = new JsonMigrator() {
    def migrate(x: JsValueWrapper): Unit = {
      x("field1").map.remove("field11")
      ()
    }
  }
private val migrator2 = new JsonMigrator { // add new field field1/field12
def migrate(input: JsValueWrapper): Unit =
  input("field1").map.update("field12", "myNewField")

private val migrator3 = new JsonMigrator { //  Change all sFields to "hahaha"
def migrate(input: JsValueWrapper): Unit =
  PathResolver.migrate(input, List(RecurFieldCond(HasField("sField")))) { w =>
    w.map.update("sField","hahaha")
  }
}

```

We can create a global migrator because we've already defined the `Monoid` for it

```scala
import scalaz.syntax.foldable._
import scalaz.std.list._
val globalMigrator = List(migrator1, migrator2, migrator3).suml
```

Now with the global script, we convert the immutable json value into the mutable version, transform it and convert back to the immutable version

```scala
val x = JsValueWrapper.create(json) // first we need to create a mutable version of the original json
allMigrator.migrate(x) // then mutate it by applying the global migration
val result = JsValueWrapper.toJson(x)
```

## Integration into deployment system

This is just an example how we can automatically migrate a database with a lot of json: The database store the current version somewhere. When executing a migration, the proram fetches the current version and find all the migration scripts, combine them to make a global script, iterate all the json value in database, convert to mutable version, transform it with the global script and finally convert it back to the immutable version

The users have to maintain a file that defines the mapping between version and script:

```scala
val l = List(
  (0, migration1),
  (1, migration2),
  (2, migration3)
)
```

If 2 users modify the file at the same time, they have to resolve conflict during the merge

# Conclusion

The library is successful because my team uses it every day. There is some very complex script that may take more than 100 lines of code. One of the big avantage is that we can use the `Format` type class instead of `Reader/Writer` thus there is no mismatch problem between the reader and the writer