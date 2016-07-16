# What makes programming languages really powerful?
> 09/05/2012

It is quite common to hear discussions in programmers' circles about how wonderful programming language *X* is because it has so cool features *Y1, Y2, ..., Yn* and how unfortunate the programmers in *Z* (*Z != X*) must be because their language doesn't have them. I always do my best not to get involved because I don't find such discussions productive. Simply everyone should use tools which suit the needs best. However, I would like to have a think about what actually makes programming languages powerful and what only gives an impression of power.

While looking for a reference documentation of some string-manipulation methods in Python, I found out on a [certain website](http://www.tutorialspoint.com/python/python_strings.htm) that: "One of Python's coolest features is the string format operator %". I stopped for while and thought about how can something like that be one of the best features of a programming language. I perfectly understand what the author meant; it cannot be taken literally, he just meant it is very convenient to format a string using this syntax (and it certainly is). It kind of reflects though, how people often think about languages and their features; what the languages can and what cannot do.

In an ideal world, the programming language should not provide native language support for formatting a string, but rather provide a support for writing abstractions which resemble native language support. For example in Scala, there is no built-in string-formatting operator, only the `format()` method defined for strings that is doing more or less the same thing as `%` in python. It's very easy, however, to enrich the `String` class with `%` method which, when written in infix operator notation, acts exactly like its Python cousin:

```scala
class PythonizedString(val str: String) {
    def %(args: Any*) = str.format(args: _*)
}

implicit def stringToPythonizedString(s: String) = new PythonizedString(s)

// And from now on, we can use Python-like syntax for string formatting
val greeting = "%s, %s!" % ("Hello", "World")
println(greeting) // => "Hello, World!"
```

This solution doesn't mess the language syntax because it's not a native construct, but rather an ordinary method defined (through an implicit conversion to a wrapper class) for all string objects and can be easily dropped by simply not importing the implicit conversion. Moreover, it is much more flexible because in the wrapper class `PythonizedString`, we can do whatever we can think of (call the existing formatting function `format()`, implement arbitrary string formatting logic, call a web service, etc.). In Python, however, as far as I know, we are stuck with the default implementation of the `%` operator.

Another example might be automatic resource management introduced in Java 7. Such feature needs a native language support only because Java does not (yet) support closures. The following snippet shows a possible approach to the automatic resource closing in Scala using a high-order function:

```scala
def use[A <: Closeable, B](stream: A)(closure: A => B): B = {
    try {
    val result = closure(stream)
    result
    } finally {
    if (stream != null) stream.close()
    }
}

val firstLine = use(new BufferedReader(new FileReader(("/home/semberal/scala/Sample.scala")))) {
    _.readLine()
}

println(firstLine)
```

I believe that new features should be added into the language very carefully, with strong emphasis on its impact on the flexibility of the language. Introducing a new special syntax only because of a stream-closing feature is, in my opinion, unfortunate. On the other hand, adding closures, which would allow creating library abstractions (including those, which would automatically close streams), would be a huge step forward. The problem is that now, when the automatic resource management feature is already part of the language, it will never go away; even though it won't be necessary anymore once closures are added to the language (as currently planned for Java 8).

No additional flexibility for a high cost of a polluting the source code with constructs which would be never possible to drop from the language is also a reason why I think that adding XML literals to Scala was not a good design decision; however I might understand the intentions of Scala authors to provide a native language support for such a popular format. Not only it makes source code look ugly with embedded snippets of xml, while not adding any additional flexibility, but also raises questions like: "Should we add a native language support for JSON when various JavaScript frameworks or NoSQL databases using it are now so popular? Or what about any other data formats?" Language designers cannot predict which formats will arise in the future and, therefore, languages should ideally provide features to create abstractions allowing to work with any format in a reasonable way.

So, to sum up my ideal world, I don't want a language which has a powerful feature Y; I want a language which gives me powerful built-in language constructs to create Y', the syntax of which would be difficult to distinguish from the syntax of a built-in language feature.

**Note:** I'm aware that this blog post might resemble the unproductive discussion described in the first paragraph, but my point is more about the idea of extensibility of programming languages in general, not the particular languages themselves.

**Update:** There is a follow-up discussion at this article's [reddit page](http://www.reddit.com/r/scala/comments/thwmt/what_makes_programming_languages_really_powerful/).