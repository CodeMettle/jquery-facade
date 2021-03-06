# jquery-facade
A strongly-typed Scala.js facade for jQuery

### Installing the Library

To use jquery-facade, add this line to your libraryDependencies:
```
"org.querki" %%% "jquery-facade" % "0.2"
```
The jquery-facade library will import the underlying jQuery code; you do not need to (and shouldn't) repeat it in
your own build.sbt. This will be exposed as "jquery.js", and you can depend on that in jsDependencies.

### Using jquery-facade

**The Important bit:** mostly, this facade should just work as expected. We discuss the details a lot
below, but mostly those details are there so that it *does* work as expected. The objective is a facade
that matches idiomatic JavaScript examples reasonably well, but is strongly-typed at the Scala level.

In general, to use jQuery, I recommend simply putting

```
import org.querki.jquery._
```

at the top of your file. While individual pieces *can* be imported, this facade uses a fair number of implicits
in order to do its thing, so it is usually easier to just import the entire thing.

The most important function in this facade is `$()`, which works the same way it does in JavaScript: it
"selects" the given Element, or the Elements indicated by the String Selector. 
(Or creates a node from a given HTML String, although I recommend using Scalatags for that instead -- 
it's more strongly-typed and Scala-ish.) `$` is an alias for the underlying
JQueryStatic object, which is the facade for the global JQuery object in JavaScript. I recommend using
the `$` alias, since that matches JavaScript idiom, and means that much JS documentation matches the
resulting Scala code.

By and large, this is a straightforward facade, so you can use jQuery as you would expect, chaining calls
together to do what you want.

Note that some methods take odd types, most often Selector. There are pseudo-type-unions, which allow you
to pass any of several types in to this parameter. The type unions currently in use are:

* Selector -- this defines a way to get to an Element. It can be a String (using jQuery's [enhanced version of CSS Selectors](http://api.jquery.com/category/selectors/), an actual Element, or a js.Array[Element].
* ElementDesc -- this is a sort of enhanced version of Selector that some methods use. It takes all of Selector, or another JQuery.
* AttrVal -- this is any of the types that can be in an Attribute: String or Int or Boolean.

For ease of use, Selector and ElementDesc are set up so that you can pass a Seq[Element] instead of a 
js.Array[Element], and it will be auto-converted.

As discussed below, the JQuery facade contains a number of `XXXInternal` methods, which are loosely-typed. Do
not use these; use their strongly-typed counterparts in JQueryTyped instead. (JQueryTyped is an implicit
conversion from JQuery, so you usually don't need to worry about this.)

#### Extensions

There is also a JQueryExtensions trait, as another implicit conversion from JQuery. This contains methods
that are *not* straightforward facades, but which I have found useful in working with jQuery from Scala.js.
While I'm not going hog-wild with this, I'm slowly adding to it. Pull Requests are welcomed. (Also, consider
these a bit experimental at the moment -- most of them predate SJS 0.6 and this being pulled out into a library,
and some might be redundant at this point.)

Note that, as of this writing, I am pondering the notion of fleshing out a proper JQuery Monad, so that you
could use JQuery inside for comprehensions. I don't think the idea is crazy; opinions are welcomed.

### Structure of the facade

Note that the facade is actually broken up into several different traits. This isn't terribly obvious from
a code point of view, but important for reading the documentation and finding the signatures. On the one hand,
there is the **JQuery** trait -- the "real" facade, which represents an actual JS jQuery object. Then there is
**JQueryTyped** -- a supplemental trait, which adds many more overloads on top of "internal" entry points in JQuery.

The reason for this division is that jQuery uses overloading *heavily*, in a way that doesn't play well with
Scala. Essentially, a number of jQuery entry points define parameters as type unions: you can use any of several
types in the parameter, and jQuery will do the right thing depending on the type actually passed in. The problem
is, this leads to a combinatoric explosion of overloads in Scala, which is difficult to maintain.

jquery-facade deals with this problem by using a sort of imitation type union, built on top of Scala's standard
Either type. At the moment, there are several of these type unions defined in package.scala. The most important
is Selector, which is a common and explicit parameter type in jQuery, but ElementDesc and AttrVal have also
proved quite useful. But this type union requires a little massaging before the actual call to JavaScript, so
calls using it need to be implemented in actual Scala code.

The result of all this is that the entry points are divided between JQuery and JQueryTyped. Methods with relatively
simple signatures live in JQuery. Methods that use the type-union trick live in JQueryTyped, and call "Internal" methods
in JQuery.

Most of the time, you can ignore all of this -- I mention it mainly so you can find signatures or documentation
when you're looking for it, and to understand the context for contributing PRs.

### Why this library?

JQuery is one of the most central components in the JavaScript ecosystem: not only is it used directly by many
web applications, a large fraction of the other major libraries are built on top of it. So while it isn't
strictly necessary for Scala.js development, complex projects often find that they need it.

The original facade for jQuery -- scala-js-jquery -- was one of the first facades promulgated as Scala.js began
to be ready for real use. It is more or less complete, but it was written fairly quickly, and the result is that
it is pretty loosely typed. That is, it functions, but it doesn't provide a lot of type support for the compiler
and IDE. This is mostly because the type-union problem was mostly dealt with by simply defining these parameters
as js.Any. It is also slightly inaccurate in a few details -- in particular, some facade signatures return T where
they should return UndefOr[T], which would cause surprising crashes in Scala.js code when the function returned
undefined.

Over the course of several months of using it very heavily, I got somewhat frustrated by this, and gradually decided
to do a rewrite. I originally thought about trying to update scala-js-jquery in place, but the desired changes
were *so* dramatic, sometimes breaking, that the authors of scala-js-jquery and I agreed that I should create a new
library instead, which folks could opt into.

### Caveats

As of this writing, this facade is quite incomplete.  jQuery is enormous and complex, and the philosophy behind this
facade has been that doing it right is more important than shoving it all out the door quickly. There are a bunch
of functions entirely missing, and a substantial number of overloaded signatures that need to be filled in. If you
need a few specific functions, drop me a line and I'll see about adding them. (And Pull Requests are welcomed.)

The type unions such as Selector are a bit inefficient -- they wind up creating and then unwinding some small objects
in the course of invocation. I haven't found this to be a problem in practice, and it's likely a drop in the bucket
compared to the function-indirection weight inside jQuery itself, but it's not a trivial consideration. If you care
about squeezing every cycle out of your Scala.js code, you may want to pay attention to this.

### Converting from scala-js-jquery

By and large, I tried to avoid *gratuitous* incompatibilities between the libraries (after all, I had a large code
base of my own to convert), but some code change is necessary:

* At the least, you need to change the libraryDependency, and change the imports in your .scala files.
* If you weren't already aliasing `jQuery` to `$`, you'll want to change your calls.
* Some methods are not implemented here yet, so you may want to check the JQuery and JQueryTyped traits for what you need before converting.
* A few methods have subtly different signatures. Most importantly, attr() returns UndefOr[String] here, where it returns String in scala-js-jquery. You will need to add a `.get` to such a call, or (better) treat it like an Option and use something like `.map`.

### Contributing to jquery-facade

Pull Requests are welcome, but please observe the style guidelines of this library:

* When the underlying jQuery call takes a parameter matching one of the type unions (especially Selector), please use that union type. However, please note that the jQuery API documentation is very inconsistent: when they say a parameter takes "Selector", they sometimes really mean a Selector -- String or Element or js.Array[Element] -- but sometimes just mean String. And sometimes they *don't* say that it takes Selector, but instead spell out the possible types and leave you to infer that they mean Selector. Read the API docs carefully, and try to interpret them correctly.
* All entry points involving type unions should go in JQueryTyped, and call an underlying loosely-typed XXXInternal call in JQuery.
* Entry points that do not require type unions should go directly into JQuery.
* JQuery and JQueryTyped are in alphabetical order, to make it easier to find things; please keep it that way.
* Please include a brief comment with each entry point, to remind folks of what it does. I usually use the summary line from the actual jQuery API.
* You do *not* have to include every possible overload for a function, but thoroughness is appreciated.
* Accuracy is paramount: all overloads should be strictly typed to match the jQuery documentation as best possible.
* If a parameter takes "anything", try to figure out whether jQuery is processing that parameter in any way. If it is, the parameter should be js.Any; if not (if it is completely opaque), then it should be scala.Any.
* If a callback parameter ignores its return value, the return type should be scala.Any.
* Be careful about the return value from a method. Most JQuery methods return JQuery, but not all.
* When a facade function takes a property bag, if it is understood to be name/value pairs in JS, declare it as js.Dictionary[T]. Often, we can constrain T; if not, just put js.Any, and it is at least explicit that it is name/value pairs.

### License

Copyright (c) 2015 Querki Inc. (justin at querki dot net)

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
