scopt
=====

scopt is a little command line options parsing library.

Sonatype
--------

```scala
libraryDependencies += "com.github.scopt" %% "scopt" % "3.1.0"

resolvers += "sonatype-public" at "https://oss.sonatype.org/content/groups/public"
```

Usage
-----

scopt provides two styles of parsing: immutable and mutable.
Either case, first you need a case class that represents the configuration:

```scala
import java.io.File
case class Config(foo: Int = -1, out: File = new File("."), xyz: Boolean = false,
  libName: String = "", maxCount: Int = -1, verbose: Boolean = false, debug: Boolean = false,
  mode: String = "", files: Seq[File] = Seq())
```

In immutable parsing style, a config object is passed around as an argument into `action` callbacks.
On the other hand, in mutable parsing style you are expected to modify the config object in place.

### Immutable parsing

Here's how you create a `scopt.OptionParser[Config]`. See [Scaladoc API](http://scopt.github.io/scopt/3.1.0/api/#scopt.OptionParser) for the details on various builder methods.

```scala
val parser = new scopt.OptionParser[Config]("scopt") {
  head("scopt", "3.x")
  opt[Int]('f', "foo") action { (x, c) =>
    c.copy(foo = x) } text("foo is an integer property")
  opt[File]('o', "out") required() valueName("<file>") action { (x, c) =>
    c.copy(out = x) } text("out is a required file property")
  opt[(String, Int)]("max") action { case ((k, v), c) =>
    c.copy(libName = k, maxCount = v) } validate { x =>
    if (x._2 > 0) success else failure("Value <max> must be >0") 
  } keyValueName("<libname>", "<max>") text("maximum count for <libname>")
  opt[Unit]("verbose") action { (_, c) =>
    c.copy(verbose = true) } text("verbose is a flag")
  opt[Unit]("debug") hidden() action { (_, c) =>
    c.copy(debug = true) } text("this option is hidden in any usage text")
  note("some notes.\n")
  help("help") text("prints this usage text")
  arg[File]("<file>...") unbounded() optional() action { (x, c) =>
    c.copy(files = c.files :+ x) } text("optional unbounded args")
  cmd("update") action { (_, c) =>
    c.copy(mode = "update") } text("update is a command.") children(
    opt[Unit]("not-keepalive") abbr("nk") action { (_, c) =>
      c.copy(keepalive = false) } text("disable keepalive"),
    opt[Boolean]("xyz") action { (x, c) =>
      c.copy(xyz = x) } text("xyz is a boolean property")
  )
}
// parser.parse returns Option[C]
parser.parse(args, Config()) map { config =>
  // do stuff
} getOrElse {
  // arguments are bad, usage message will have been displayed
}
```

The above generates the following usage text:

```
scopt 3.x
Usage: scopt [update] [options] [<file>...]

  -f <value> | --foo <value>
        foo is an integer property
  -o <file> | --out <file>
        out is a required file property
  --max:<libname>=<max>
        maximum count for <libname>
  --verbose
        verbose is a flag
some notes.

  --help
        prints this usage text
  <file>...
        optional unbounded args

Command: update
update is a command.

  -nk | --not-keepalive
        disable keepalive
  --xyz <value>
        xyz is a boolean property
```

#### Options

Command line options are defined using `opt[A]('f', "foo")` or `opt[A]("foo")` where `A` is any type that is an instance of `Read` typeclass.

- `Unit` works as a plain flag `--foo` or `-f`
- `Int`, `Long`, `Double`, `String`, `BigInt`, `BigDecimal`, `java.io.File`, and `java.net.URI` accept a value like `--foo 80` or `--foo:80`
- `Boolean` accepts a value like `--foo true` or `--foo:1`
- `java.util.Calendar` accepts a value like `--foo 2000-12-01`
- A pair of types like `(String, Int)` accept a key-value like `--foo:k=1` or `-f k=1`

This could be extended by defining `Read` instances in the scope.

By default these options are optional. 

#### Short options

For plain flags (`opt[Unit]`) short options can be grouped as `-fb` to mean `--foo --bar`.

`opt` accepts only a single characeter, but using `abbr("ab")` a string can be used too:

```scala
opt[Unit]("no-keepalive") abbr("nk") action { (x, c) => c.copy(keepalive = false) }
```

#### Help, Version, and Notes

There are special options with predefined action called `help("help")` and `version("version")`, which prints usage text and header text respectively.

`note("...")` is used add given string to the usage text.

`override def completeOnError(config: C)` to override the default action on parse error.
You may show brief usage on error with this.

```scala
// options and args...
override def completeOnError(config: C) = {
  Console.err.println("try `scopt --help' for more information")
  Some(c) // whole parse result becomes `c'
}

// otherwise
override def completeOnError(config: C) = {
  Console.err.println("try `scopt --help' for more information")
  None // whole parse result becomes fail
}
```

#### Arguments

Command line arguments are defined using `arg[A]("<file>")`. It works similar to options, but instead it accepts values without `--` or `-`. By default, arguments accept a single value and are required.

#### Occurrence

Each opt/arg carries occurrence information `minOccurs` and `maxOccurs`.
`minOccurs` specify at least how many times an opt/arg must appear, and
`maxOccurs` specify at most how many times an opt/arg may appear.

Occurrence can be set using the methods on the opt/arg:

```scala
opt[String]('o', "out") required()
opt[String]('o', "out") minOccurs(1) // same as above
arg[String]("<mode>") optional()
arg[String]("<mode>") minOccurs(0) // same as above
arg[String]("<file>...") optional() unbounded()
arg[String]("<file>...") minOccurs(0) maxOccurs(1024) // same as above
```

#### Validation

Each opt/arg can carry multiple validation functions.

```scala
opt[Int]('f', "foo") action { (x, c) => c.copy(intValue = x) } validate { x =>
  if (x > 0) success else failure("Option --foo must be >0") } validate { x =>
  failure("Just because") }
```

The first function validates if the values are positive, and
the second function always fails.

#### Commands

Commands may be defined using `cmd("update")`. Commands could be used to express `git branch` kind of argument, whose name means something. Using `children` method, a command may define child opts/args that get inserted in the presence of the command. To distinguish commands from arguments, they must appear in the first position within the level. It is generally recommended to avoid mixing args both in parent level and commands to avoid confusion.

```scala
arg[String]("<file>...") unbounded() optional()
cmd("update") action { (_, c) =>
  c.copy(mode = "update") } text("update is a command.") children(
  opt[Unit]("not-keepalive") abbr("nk") action { (_, c) =>
    c.copy(keepalive = false) } text("disable keepalive"),
  opt[Boolean]("xyz") action { (x, c) =>
    c.copy(xyz = x) } text("xyz is a boolean property")
)
```

In the above, `update test.txt` would trigger the update command, but `test.txt update` won't.

Commands could be nested into another command as follows:

```scala
cmd("backend") text("commands to manipulate backends:\n") action { (x, c) =>
  c.copy(flag = true) } children(
  cmd("update") children(
    arg[String]("<a>") action { (x, c) => c.copy(a = x) } 
  )     
)
```

### Mutable parsing

Create a `scopt.OptionParser[Unit]` and customize it with the options you need, passing in functions to process each option or argument. Use `foreach` instead of `action`.

```scala
val parser = new scopt.OptionParser[Unit]("scopt") {
  head("scopt", "3.x")
  opt[Int]('f', "foo") foreach { x =>
    c = c.copy(foo = x) } text("foo is an integer property")
  opt[java.io.File]('o', "out") required() valueName("<file>") foreach { x =>
    c = c.copy(out = x) } text("out is a required file property")
  opt[(String, Int)]("max") foreach { case (k, v) =>
    c = c.copy(libName = k, maxCount = v) } validate { x =>
    if (x._2 > 0) success else failure("Value <max> must be >0") 
  } keyValueName("<libname>", "<max>") text("maximum count for <libname>")
  opt[Unit]("verbose") foreach { _ =>
    c = c.copy(verbose = true) } text("verbose is a flag")
  opt[Unit]("debug") hidden() foreach { _ =>
    c = c.copy(debug = true) } text("this option is hidden in any usage text")
  note("some notes.\n")
  help("help") text("prints this usage text")
  arg[java.io.File]("<file>...") unbounded() optional() foreach { x =>
    c = c.copy(files = c.files :+ x) } text("optional unbounded args")
  cmd("update") foreach { _ =>
    c.copy(mode = "update") } text("update is a command.") children(
    opt[Boolean]("xyz") foreach { x =>
      c = c.copy(xyz = x) } text("xyz is a boolean property")
  )
}
if (parser.parse(args)) {
  // do stuff
}
else {
  // arguments are bad, usage message will have been displayed
}
```

Building
--------

sbt to build scopt.

License
-------

MIT License.

Credits
-------

- January 13, 2008: Aaron Harnly creates [aaronharnly/scala-options](https://github.com/aaronharnly/scala-options).
- December 1, 2009: Tim Perrett introduces it [as a gist](http://gist.github.com/246481) on [Parsing command lines argument in a "scalaesque" way](http://www.scala-lang.org/node/4380).
- January 10, 2010: James Strachan takes the code, adds usage text, sbt build, etc and creates [jstrachan/scopt](https://github.com/jstrachan/scopt), which is also mentioned in [Scala CLI Library?](http://www.scala-lang.org/node/4959).
- March 4th, 2010: Eugene Yokota joins scopt project, improves usage text, and adds support for key=value option and argument list.
- May 27, 2011: scopt 1.0.0 is released to scala-tools.org.
- March 18, 2012: Eugene adds immutable parser, forks the project to [scopt/scopt](https://github.com/scopt/scopt), and releases scopt 2.0.0.
- June 7, 2013: Eugene rewrites scopt from scratch for polymorphic options, and releases scopt 3.0.0.

Changes
-------

See [notes](https://github.com/scopt/scopt/tree/master/notes).
