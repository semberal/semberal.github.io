# JSON validation and error handling in Play 2.1
> 21/04/2013

Play framework 2.1 comes with a slightly new approach to reading JSON values. In version 2.0, there was a `Reads[T]` trait with a single method `reads(json: JsValue): T`, which simply took the JSON object and converted it to the domain object. With such a simple approach, it was very difficult to handle various conversion errors such as missing field, incorrect data types, etc. Since it was not even possible to return something like `Either[ErrorDescription, T]` from the `reads` method, it was necessary to throw and handle various parsing exceptions. With the new Play framework 2.1, thankfully, this has changed.

Play 2.1 `reads` method doesn't return `T` directly anymore, but rather a `JsResult[T]`, which can be either a `JsSuccess[T]` if the read succeeded or `JsError[T]` otherwise. Examples and deeper explanation can be found in the [documentation](http://www.playframework.com/documentation/2.1.1/ScalaJsonCombinators). I would like to present here, how you can utilize the `JsError` object to *automatically* notify the clients about conversion/validation errors.

Consider an example scenario, when we are writing a RESTful web service, which accepts JSON objects representing user registrations. We can represent the user by the following case class:

```scala
case class User(id: Option[Long], email: String, password: String, fullname: Option[String])
```

Let's create an implementation of the `Format[User]` trait, which converts between the domain object and its JSON representation:

```scala
implicit object UserJsonConverter extends Format[User] {
  def reads(json: JsValue): JsResult[User] = (
    (__ \ "id").readNullable[Long] and
    (__ \ "email").read(email) and
    (__ \ "password").read(minLength[String](8)) and
    (__ \ "fullname").readNullable[String])(User.apply _).reads(json)

  def writes(o: User): JsValue = Json.obj(
    "id" -> o.id,
    "email" -> o.email,
    "password" -> o.password,
    "fullname" -> o.fullname
  )
}
```

Notice how we handle not just the JSON conversion here, but also some validation logic, such as minimal password length or correct e-mail format. You might see some resemblance with parser combinators I wrote about in the [last post](https://github.com/semberal/semberal.github.io/blob/master/parser-combinators-and-what-if-regular-expressions-are-not-enough.md); you might also write the `and` combinator as `~` to emphasise they are both monadic combinators.

In the action handler, we can now parse the JSON contained in the body of the POST request:

```scala
def post = Action {
  request =>
    request.body.asJson.map(_.validate[User] match {
      case JsSuccess(user, _) => // store the user in the database
      case err@JsError(_) => BadRequest(Json.toJson(err))
    }).getOrElse(BadRequest)
}
```

Here, we validate the JSON as the `User` instance and pattern match whether or not it succeeded. If so, we can obtain the `User` instance from the `JsSuccess` and pass it to a further processing. If the result is `JsError`, however, we need to somehow inform the user that the conversion/validation failed. If we are writing a RESTful web service, for example, it might be handy to return a JSON containing some information about the validation errors. We can write an implicit `Writes[JsError]`, which would serialize the `JsError` object into JSON. Here is one of the possible implementations:

```scala
implicit object JsErrorJsonWriter extends Writes[JsError] {
  def writes(o: JsError): JsValue = Json.obj(
    "errors" -> JsArray(
      o.errors.map {
        case (path, validationErrors) => Json.obj(
          "path" -> Json.toJson(path.toString()),
          "validationErrors" -> JsArray(validationErrors.map(validationError => Json.obj(
            "message" -> JsString(validationError.message),
            "args" -> JsArray(validationError.args.map(_ match {
              case x: Int => JsNumber(x)
              case x => JsString(x.toString)
            }))
          )))
        )
      }
    )
  )
}
```

So, if we post an empty JSON - `{}` - to the `post` action, for example, we would receive the following JSON with the description of validation errors:

```javascript
{
  "errors": [
    {
      "validationErrors": [
        {
          "args": [],
          "message": "validate.error.missing-path"
        }
      ],
      "path": "/email"
    },
    {
      "validationErrors": [
        {
          "args": [],
          "message": "validate.error.missing-path"
        }
      ],
      "path": "/password"
    }
  ]
}
```

What is beautiful about this approach is that you get conversion/validation problem notification almost for free. You only have to write a `Reads[JsError]` object (or use the one I presented), make it implicit in the scope and you are all set. Then, you can run validate on the JSON object you receive and in case of an error, return the serialized `JsError` instance to the client. This can be particularly handy when you are writing a web service and communicate with the client with JSON objects only. If the action handler, on the other hand, generates the view directly, the [standard Play form validation approach](http://www.playframework.com/documentation/2.1.1/ScalaForms) is a probably a better option.

### EDIT: 18/01/2014

I just discovered `JsError.toFlatJson(e)` method in the Play framework API, which is doing essentially the [same thing](https://github.com/playframework/playframework/blob/2.2.1/framework/src/play-json/src/main/scala/play/api/libs/json/JsResult.scala#L44).
