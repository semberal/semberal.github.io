# Parser combinators (and what if regular expressions are not enough)
> 11/06/2012

Regular expressions are the most widely used technique of describing text. All languages somehow support regular expressions, they are relatively easy to use and all over the internet there are predefined ones for many common scenarios (such as for a correct e-mail, domain name, etc.). They might look, therefore, as an universal solution for all text manipulation needs. In this article, I would like to show there is an other, more powerful, way of processing text which is as easy to use as regexps - parser combinators.

I first encountered parser combinators during Haskell courses on the university. Unfortunately, my attitude that time was something like "Don't bother me with monads, I wanna do real-world. Give me Spring and Hibernate!" However, now, when doing real-world, I see how invaluable these theoretical foundations are and how often I can benefit from it in practice.

The idea behind parser combinators is quite simple, two fundamental elements are **parsers** and **parser combinators**. Parsers recognize and process some input. They can either succeed or fail. Parser combinators, on the other hand, combine existing parsers into more complex ones. Therefore, complicated parsers can be built from bottom up, using an appropriate composition of simpler parsers, which are easier to understand, implement and test. Let's take a look at several scenarios where parser combinators are used as a convenient alternative to regular expressions (or rather their extension - as you will see later). Code examples are written in Scala, which provides very nice DSL for this purpose. However, parser combinator libraries exist for many languages and once you understand the concept, it shouldn't be difficult to port it across different languages.

### Simple example: parsing IPv4 addresses

Let's take a look at the following code snippet:

```scala
def parseUsingRegexp(input: String): Option[(Int, Int, Int, Int)] = {
  val IP = """b(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?).(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?).(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?).(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)b""".r
  input match {
    case IP(a, b, c, d) => Some(a.toInt, b.toInt, c.toInt, d.toInt)
    case _ => None
  }
}
```

Unless you are Perl hacker, it took you a couple of seconds to find out what is actually going on in here. It is a method that receives a string on the input and returns either `None` if the string is not correct IPv4 address or a tuple of 4 integers representing the four parts of the IPv4 address. The regular expression on the second line is copied from a [website](http://www.regular-expressions.info/examples.html) collecting useful regexps. As we can see, such expression is very complicated (163 characters long) and thus error-prone (just imagine you accidentally delete one character - nobody might ever find out). Moreover, it is also repetitive. Since IPv4 contains four 1-byte numbers separated with dots, it would be great if we could somehow define the pattern for one group and repeat it several times. Indeed, we might have used quantifiers to specify exact repetition, but then we wouldn't have had access to captured groups.


Now let's take a look, how the same functionality can be achieved using parser combinators:

```scala
def parseUsingParserCombinator(input: String): Option[(Int, Int, Int, Int)] = {
  def dot: Parser[Any] = "."
  def ipGroup: Parser[Int] = "25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?".r ^^ (_.toInt)
  def ip: Parser[(Int, Int, Int, Int)] = (ipGroup <~ dot) ~ (ipGroup <~ dot) ~ (ipGroup <~ dot) ~ ipGroup ^^ {
    case (a ~ b ~ c ~ d) => (a, b, c, d)
  }
  parseAll(ip, input) match {
    case Success(r, _) => Some(r)
    case _ => None
  }
}
```

Here we have 2 simple parser combinators: `dot`, simply denoting a dot, and `ipGroup` for numbers 0-255. The third parser, `ip` is created as a sequential composition (parser combinator `~`) of those two simpler ones. Sequential composition means that the parsers chained with `~` will be executed consecutively, one after another, and the computation succeeds if all the parsers in the chain succeed. There is also parser combinator `<~`, which is doing the same as `~`, it just forgets the result of its right argument, returns only the result of the left one and thus passes it to a further processing (here we are not interested in any further processing of dots).

All parsers have a type. The type of a parser denotes what does it produce when it successfully reads the input. For example, the type of the parser `ipGroup` is `Parser[Int]`, which means that after the parser reads its portion of the input, it will result in an `Int`. In order to convert parser results, there is the parser combinator `^^`, which takes a transformation function as an argument.

We can now ask what can actually be a parser. In case of the `dot` parser, we assign an ordinary string, while `ipGroup` seems to be a regular expression. In reality, all the parsers are obliged to extend `(Input => ParseResult[T])`, but we can use strings and regular expressions as parsers because they are implicitly converted to `Parser[String]`. However, this is a Scala-specific issue, not applicable in other languages. For details consult its [documentation](http://www.scala-lang.org/api/current/index.html#scala.util.parsing.combinator.RegexParsers).

### School example: Infix to prefix conversion

Let's go to another example, now something what cannot be done using regular expressions. We have a following grammar producing boolean expressions:

```scala
Expr ::= Elem and Elem | Elem or Elem | Elem
Elem ::= not Token | Token
Token ::= true | false | (Expr)
```

Examples of strings generated by this grammar are `true`, `false`, `not true`, `not (true and false)`, `true or (not false)`, etc. We can see that this language is not regular anymore, yet  context-free. The idea of the proof is very simple - just imagine an n-state finite automaton and an expression starting with n+1 opening parentheses. How could such machine verify that all opening parentheses are properly closed? Therefore, the language generated by such grammar cannot be described by regular expressions.

Let's say we want to write a method which would check if an expression passed as an argument is correct and convert it into the Scheme-like prefix notation (incl. converting values `true` and `false` to `#t` and `#f`, respectively). So, for example `true or false` becomes `(or #t #f)`, `true and not (false or true)` becomes `(and #t (not (or #f #t)))`, etc. I remember doing something similar few years back as a home assignment in C. If I had known parser combinators that time, the number of lines necessary would have reduced to one fifth or something like that.

Our strategy will be following: instead of directly manipulate the input string, we will first convert the expression into our internal representation and then create a new string representing the very same expression, just in the prefix notation. Our object model will look like this:

```scala
sealed trait Expr
case class Atom(value: Boolean) extends Expr
case class Not(expr: Expr) extends Expr
case class And(left: Expr, right: Expr) extends Expr
case class Or(left: Expr, right: Expr) extends Expr
case class Paren(value: Expr) extends Expr
```

We will need to write three parsers in total. Each one will correspond to a specific rule of the grammar:

```scala
def expr: Parser[Expr] =
    (elem <~ whiteSpace ~ "and" ~ whiteSpace) ~ elem ^^ {case x ~ y => And(x, y)} |
    (elem <~ whiteSpace ~ "or" ~ whiteSpace) ~ elem ^^ {case x ~ y => Or(x, y)} |
    elem

def elem: Parser[Expr] = "not" ~> whiteSpace ~> atom ^^ {Not(_)} | atom

def atom: Parser[Expr] = ("true" | "false") ^^ {x => Atom(value = x.toBoolean)} |
    "(" ~> expr <~ ")"
```
Now, when our parsing infrastructure is ready, we only need to write a method which triggers parsing and then converts our internal representation of the expression into the prefix notation:

```scala
def convert(input: String) = {
  def doConvertToInfix(expr: Expr): String = expr match {
    case x: And =>
      "(and " + doConvertToInfix(x.left) + " " + doConvertToInfix(x.right) + ")"
    case x: Or =>
      "(or " + doConvertToInfix(x.left) + " " + doConvertToInfix(x.right) + ")"
    case x: Not =>
      "(not " + doConvertToInfix(x.expr) + ")"
    case x: Atom => if (x.value) "#t" else "#f"
    case x: Paren => doConvertToInfix(x.value)
  }

  def parseExpression(s: String): Expr = parseAll(expr, s) match {
    case Success(res, _) => res
    case Failure(msg, _) => throw new Exception(msg)
  }

  val ast = parseExpression(input)
  doConvertToInfix(ast)
}
```

### Real-world example: parsing chess board state

In chess programming, there are six elements necessary to remember in order to reconstruct the game at the very same state: the position of pieces on the board, the player whose turn it currently is, available castlings, en passant square, the number of halfmoves since the last capture and the current move number. There is a de-facto standard notation for this purpose, called [Forsythâ€“Edwards Notation](http://en.wikipedia.org/wiki/Forsyth%E2%80%93Edwards_Notation) (FEN). For details of the structure of games stored in this format, please consult the [wiki](http://en.wikipedia.org/wiki/Forsyth%E2%80%93Edwards_Notation), its [documentation](http://kirill-kryukov.com/chess/doc/fen.html) or [F.A.Q](http://www.chessgames.com/fenhelp.html). FEN string of a chess game in the default position is following: `rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1`.

Parsing a FEN string and converting it to some internal structure representing a game state is not that difficult. Without the use of some more advanced techniques, however, it would require complicated nested loops and long switches. I will present here, how parsing such string using parser combinators can be elegant, concise and fun.

Let's have a following object model:

```scala
// Piece representation
sealed abstract class Piece {
  val white: Boolean
}

object Piece {
  def apply(str: String): Piece = str match {
    case "p" => new Pawn(false)
    case "P" => new Pawn
    case "r" => new Rook(false)
    case "R" => new Rook
    ...
    case _ =>
      throw new IllegalArgumentException("Passed incorrect piece representation")
  }
}

case class Pawn(white: Boolean = true) extends Piece
case class Rook(white: Boolean = true) extends Piece
case class Knight(white: Boolean = true) extends Piece
case class Bishop(white: Boolean = true) extends Piece
case class Queen(white: Boolean = true) extends Piece
case class King(white: Boolean = true) extends Piece

// Castling representation
sealed abstract class Castling {
  val white: Boolean
}

object Castling {
  def apply(str: String): Castling = str match {
    case "k" => new KingsideCastling(false)
    case "K" => new KingsideCastling
    case "q" => new QueensideCastling(false)
    case "Q" => new QueensideCastling
    case _ =>
      throw new IllegalArgumentException("Passed incorrect castling representation")
  }
}

case class KingsideCastling(white: Boolean = true) extends Castling

case class QueensideCastling(white: Boolean = true) extends Castling


// Game representation
class GameState private(val board: List[Option[Piece]], val whiteOnTurn: Boolean,
                        val castlingAvailability: Set[Castling],
                        val enPassantSquare: Option[Int], val halfMoveClock: Int,
                        val fullmoveNumber: Int)

object GameState {
  def apply(fen: String): Option[GameState] = ... // parsing will be here
}
```

If we implement `GameState` this way, we will be able to use the following very concise code to convert a string, containing a game in FEN, into our internal representation:

```scala
val fen = "rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1"
val gameState = GameState(fen)
```

Let's now take a look what actually do we need to parse. The FEN string consist of six elements separated with whitespace. Therefore, our parser will be a sequential composition of eleven parsers, six ones parsing individual elements of the FEN string and five ones parsing whitespace between elements. Let's start with the most simple ones. Last two elements are simple integers. Our parser, therefore, will be simple regular expression matching integers:

```scala
  val intParser: Parser[Int] = """d+""".r ^^ (_.toInt)
```

Fourth element denotes en passant square. It is written as a square in algebraic notation (e.g. a1, c6, h8, ...). However, in the `GameState` class, we store all fields as integers, starting with 0 up to 63, hence our parser will be of type `Parser[Option[Int]]` and not `Parser[String]`. `Option` is used because the en passant square might be empty (and thus written as `-` in FEN).

```scala
val enpassantSquareParser: Parser[Option[Int]] = "-" ^^ (_ => None) |
    ("a" | "b" | "c" | "d" | "e" | "f" | "g" | "h") ~
    ("1" | "2" | "3" | "4" | "5" | "6" | "7" | "8") ^^
    {case x ~ y => Some(8 * (y.head - '1') + (x.head - 'a'))}
```

Third element in a FEN string represents available castlings. Since it does not make sense to hold duplicate values, available castlings are represented as a set in our object model. The parser, therefore, is of type `Parser[Set[Castling]]`:

```scala
val castlingParser: Parser[Set[Castling]] =
    "-" ^^ {_ => Set[Castling]()} |
    rep("k" | "K" | "q" | "Q") ^^ {_.map {x => Castling(x)}.toSet}
```

The current player's color is denoted by the second element in FEN. It is either "w" or "b". In the object model, the color is denoted by a boolean value passed to the constructor of all pieces. Hence the type of the parser is `Parser[Boolean]`:

```scala
val activeColorParser: Parser[Boolean] = ("w" | "b") ^^ {_ == "w"}
```

Not let's get to the most complicated parser which is responsible for parsing the first element of the FEN string - the position of pieces on the board. In our object model, pieces on the board are represented by a list of exactly 64 elements of type `Option[Piece]`. It means that if there is a piece `p` on a square `i` of a board `board`, then `board(i) = Some(p)`; or `board(i) = None` otherwise. Therefore, the type of our parser will be `Parser[List[Option[Piece]]]`.

When we look at the format of the first FEN element, we can see some repetition there. The string consists of 8 strings describing position of pieces on a particular rank (= row on the chessboard), separated with a slash. We can thus create a parser which parses one rank and parsing of the whole board will be a composition of rank parsers. We can proceed even further - the rank parser parses an input which consists either of characters representing the pieces on the board, or of digits denoting how many consequent empty squares there are. Let's create separate parsers for them, too:

```scala
val occupiedSquaresParser: Parser[List[Option[Piece]]] =
    ("p" | "P" | "r" | "R" | "n" | "N" | "b" | "B" | "q" | "Q" | "k" | "K") ^^ {
      x => List(Some(Piece(x)))
}

val emptySquaresParser: Parser[List[Option[Piece]]] =
    ("1" | "2" | "3" | "4" | "5" | "6" | "7" | "8") ^^ {
      n => List.fill(n.toInt)(None)
}
```

Now, when we are able to parse individual characters, let's create the rank parser:

```scala
val rankParser: Parser[List[Option[Piece]]] =
    rep(occupiedSquaresParser | emptySquaresParser) ^^ {_.flatMap {x => x}}
```

You can see that the parser combinator `rep` is used here. It applies specified parser as many times as possible and its result is of type `List[A]`, where `A` is the type of the parser passed as an argument. In this case, both `occupiedSquaresParser` and `emptySquaresParser` are of type `Parser[List[Option[Piece]]]` and thus the result of the parser `rep` would be `List[List[Option[Piece]]]]`. Therefore, it is necessary to merge all the nested lists into a single one using `flatMap()`.

Now, we can use our `rankParser` to create the parser of the whole board:

```scala
val boardParser: Parser[List[Option[Piece]]] = repsep(rankParser, "/") ^^
        (_.reverse.flatMap(x => x))
```

You might wonder, why the list is reversed in the `boardParser`. The FEN string describes the board starting from the black's side (back rank), but in our representation, we start from white's perspective.

All necessary helper parsers are ready, let's now create a complete parser, converting a string in FEN notation into our object model:

```scala
val fenParser: Parser[GameState] =
    (boardParser <~ whiteSpace) ~ (activeColorParser <~ whiteSpace) ~
    (castlingParser <~ whiteSpace) ~ (enpassantSquareParser <~ whiteSpace) ~
    (intParser <~ whiteSpace) ~ intParser ^^
    {case a ~ b ~ c ~ d ~ e ~ f => new GameState(a, b, c, d, e, f)}
```

Do you see the beauty here? Such complicated parsing logic written as a combination of simple, easily understandable tasks.

The last necessary step is to write a method which triggers parsing itself:

```scala
def doParse(fen: String): Option[GameState] = parseAll(fenParser, fen) match {
  case Success(result, _) => Some(result)
  case Failure(msg, _) => throw new Exception(msg)
}
```

### Conclusion

In this article I've shown  how parser combinators work, how can we use them as a complement to regular expressions and an example where parser combinators are superior to regexps in parsing a language, which is not regular. You can find complete sources to all the examples from this article, as well as various ScalaTest tests, [on GitHub](https://github.com/semberal/blog-examples/tree/master/parser-combinators-and-what-if-regular-expressions-are-not-enough).