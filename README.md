scopt
=====

scopt is a little command line options parsing library.

Sonatype
--------

```scala
libraryDependencies += "com.github.scopt" %% "scopt" % "3.0.0-SNAPSHOT"

resolvers += "sonatype-public" at "https://oss.sonatype.org/content/groups/public"
```

Usage
-----

scopt provides two styles of parsering immutable and mutable.
Either case, first you need a case class that represents the configuration:

```scala
case class Config(foo: Int = -1, out: String = "", xyz: Boolean = false,
  libName: String = "", maxCount: Int = -1, verbose: Boolean = false,
  mode: String = "", files: Seq[String] = Seq())
```

In immutable parsing style, a config object is passed around as an argument into `action` callbacks.
On the other hand, in mutable parsing style you are expected to modify the config object in place.

### Immutable parsing

Here's how you create a `scopt.OptionParser[Config]`. See [Scaladoc API](http://scopt.github.com/scopt/latest/api/) for the details on various builder methods.

```scala
val parser = new scopt.OptionParser[Config]("scopt") {
  head("scopt", "3.x")
  opt[Int]('f', "foo") action { (x, c) =>
    c.copy(foo = x) } text("foo is an integer property")
  opt[String]('o', "out") required() valueName("<file>") action { (x, c) =>
    c.copy(out = x) } text("out is a required string property")
  opt[(String, Int)]("max") action { case ((k, v), c) =>
    c.copy(libName = k, maxCount = v) } validate { x =>
    if (x._2 > 0) success else failure("Value <max> must be >0") 
  } keyValueName("<libname>", "<max>") text("maximum count for <libname>")
  opt[Unit]("verbose") action { (_, c) =>
    c.copy(verbose = true) } text("verbose is a flag")
  note("some notes.\n")
  help("help") text("prints this usage text")
  arg[String]("<file>...") unbounded() optional() action { (x, c) =>
    c.copy(files = c.files :+ x) } text("optional unbounded args")
  cmd("update") action { (_, c) =>
    c.copy(mode = "update") } text("update is a command.") children {
    opt[Boolean]("xyz") action { (x, c) =>
      c.copy(xyz = x) } text("xyz is a boolean property")
  }
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
        out is a required string property
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

  --xyz <value>
        xyz is a boolean property
```

#### Options

Command line options are defined using `opt[A]('f', "foo")` or `opt[A]("foo")` where `A` is any type that is an instance of `Read` typeclass.

- `Unit` works as a plain flag `--foo` or `-f`
- `Int`, `Double`, and `String` accept a value like `--foo 80` or `--foo:80`
- `Boolean` also accepts a value like `--foo true` or `--foo:1`
- A pair of types like `(String, Int)` accept a key-value like `--foo:k=1` or `-f k=1`

By default these options are optional.

#### Arguments

Command line arguments are defined using `arg[A]("<file>")`. It works similar to options, but instead it accepts values without `--` or `-`. By default, arguments accept a single value and are required.

#### Commands

Commands may be defined using `cmd("update")`. Commands could be used to express `git branch` kind of argument, whose name means something. Using `children` method, a command may define child opts/args that get enabled in the presence of the command. To distinguish commands from arguments, they must appear in the first position within the level.

```scala
cmd("update") action { (_, c) =>
  c.copy(mode = "update") } text("update is a command.") children {
  opt[Boolean]("xyz") action { (x, c) =>
    c.copy(xyz = x) } text("xyz is a boolean property")
}
arg[String]("<file>...") unbounded() optional()
```

In the above, `update test.txt` would trigger the update command, but `test.txt update` won't.

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

### Mutable parsing

Create a `scopt.OptionParser[Unit]` and customize it with the options you need, passing in functions to process each option or argument. Use `foreach` instead of `action`.

```scala
val parser = new scopt.OptionParser[Unit]("scopt") {
  head("scopt", "3.x")
  opt[Int]('f', "foo") foreach { x =>
    c = c.copy(foo = x) } text("foo is an integer property")
  opt[String]('o', "out") required() valueName("<file>") foreach { x =>
    c = c.copy(out = x) } text("out is a required string property")
  opt[(String, Int)]("max") foreach { case (k, v) =>
    c = c.copy(libName = k, maxCount = v) } validate { x =>
    if (x._2 > 0) success else failure("Value <max> must be >0") 
  } keyValueName("<libname>", "<max>") text("maximum count for <libname>")
  opt[Unit]("verbose") foreach { _ =>
    c = c.copy(verbose = true) } text("verbose is a flag")
  note("some notes.\n")
  help("help") text("prints this usage text")
  arg[String]("<file>...") unbounded() optional() foreach { x =>
    c = c.copy(files = c.files :+ x) } text("optional unbounded args")
  cmd("update") foreach { _ =>
    c.copy(mode = "update") } text("update is a command.") children {
    opt[Boolean]("xyz") foreach { x =>
      c = c.copy(xyz = x) } text("xyz is a boolean property")
  }
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

scopt 1.x was created by James Strachan based on the code from Tim Perrett,
which is based on Aaron Harnly's code mentioned [in this thread](http://old.nabble.com/-scala--CLI-library--ts19391923.html#a19391923) and [this thread](http://old.nabble.com/Parsing-command-lines-argument-in-a-%22scalaesque%22-way-tp26592006p26595257.html), which is available [as a gist](http://gist.github.com/246481) or [here](http://harnly.net/tmp/OptionsParser.scala).
Eugene Yokota joined the project, and forked to scopt/scopt for 2.x.

2.x added immutable parser, and 3.x was rewritten from scratch to support better polymophism and validation.

Changes
-------

See [notes](https://github.com/scopt/scopt/tree/master/notes).
