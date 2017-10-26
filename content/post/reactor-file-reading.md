+++
title = "File Reading in Reactor"
date = 2017-10-26T12:50:00+02:00
#notonhomepage = true

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["reactive programming", "Java 8"]
categories = ["Reactor"]

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
[header]
image = "headers/reactor-file-reading.png"
caption = "[Code Vignette](# 'CC-By-SA Simon BASLE')"

# Useful shortcodes:
# {{% toc%}}
# {{% alert note %}} ... {{% /alert %}}
# {{% alert warning %}} ... {{% /alert %}}
# {{< figure src="/img/foo.png" title="Foo!" >}}
# {{< tweet 666616452582129664 >}}
# {{< gist USERNAME GIST-ID  >}}
# {{< speakerdeck 4e8126e72d853c0060001f97 >}}

# Footnotes: [^1] and [^1]: Footnote example.

+++

File reading, line by line in Java... It's never been a breeze, and until Java 8
the only high level option you had was to read the lines into a `List<String>`
:sweat:

Then Java 8 came along, with its `Stream` interface, and a `Files.lines(Path)`
method that returns a `Stream<String>`. Turns out, this stream will **lazily**
read the lines from the file, without ever having to hold the whole content of
the file in memory :heart_eyes:

Let's see how we can use that with Reactor!

<!--more-->

{{% toc%}}

# How To Correctly Convert The `Stream` To A `Flux`?

Let's not kid ourselves, the `Stream` is doing all the heavy lifting here. But
the `Stream` API is not as rich as the `Flux` API, and maybe the rest of your
app is using `Flux` anyway?

Fortunately, the conversion is pretty straightforward since there's a `Flux.fromStream`
factory method.

Ah! But this particular `Stream` of lines is doing I/O and should be **closed**
when we're done with it, so let's add a little resource management with `using`:

```java
private static Flux<String> fromPath(Path path) {
	return Flux.using(() -> Files.lines(path),
			Flux::fromStream,
			BaseStream::close
	);
}
```

# The Setup

The example we'll take is one of reading a larger than usual text file to find
specific lines in it. Namely, the file is a concatenation from several books
from [Project Gutenberg](https://www.gutenberg.org/), further separated into
"bookshelves" by the use of the special line "`##BOOKSHELF##`".

The books are:

 - `Alice’s Adventures in Wonderland` by Lewis Carroll
 - `Beowulf` (unlisted author)
 - `Dracula` by Bram Stoker
 - `The Works of Edgar Allan Poe` by Edgar Allan Poe (duh)
 - `Grimms’ Fairy Tales` by (you guessed it) The Brothers Grimm
 - `Pride and Prejudice` by Jane Austen
 - `The Adventures of Sherlock Holmes` by Arthur Conan Doyle
 - `The Republic` by Plato

I downloaded them into their own `.txt` files, all prefixed by `book-`, and did
the concatenation 7 times to simulate 7 bookshelves, using the following shell
command:

```bash
cat book-* >> bookshelf.txt && echo "##BOOKSHELF##" >> bookshelf.txt
```

This gives me a nice **34MB** file:
```
book-alice.txt 170K
book-beowulf.txt 295K
book-dracula.txt 863K
book-edgardAllanPoe.txt 570K
book-grimmsFairyTales.txt 548K
book-pridePrejudice.txt 710K
book-sherlockHolmes.txt 581K
book-theRepublic.txt 1.2M
bookshelf.txt 34M
```

# The Code

## Imperative Version
For reference, here is an imperative approach, from the pre-Java 8 days:

```java
private static void listVersion(Path path) throws IOException {
  final Runtime runtime = Runtime.getRuntime(); // <1>

  List<String> wholeData = Files.readAllLines(path);
  List<String> books = new ArrayList<>();
  Iterator<String> iter = wholeData.iterator();
  String title = null;
  while(iter.hasNext()) {
    String line = iter.next();

    if (line.startsWith("Title: ")) {
      title = line.replaceFirst("Title: ", "");
    }
    else if (line.startsWith("Author: ")) {
      String author = line.replaceFirst("Author: ", " by ");
      books.add(title + author);
      title = null;
    }
    else if (line.equalsIgnoreCase("##BOOKSHELF##")) {
      System.gc(); // <2>
      System.out.println("\n\nFound new bookshelf of " + books.size() + " books:");
      System.out.println(books);
      System.out.printf("Memory in use while reading: %dMB\n",  // <3>
        (runtime.totalMemory() - runtime.freeMemory()) / (1024 * 1024));
      books.clear();
    }
  }
}
```

Notice I sprinkled in some system calls to get a rough idea of the program's
memory consumption (:one: :three:) and to try and keep it minimal by triggering
GCs (:two:). This is all very naive, but will give us a comparison point.

So what does it do?

 1. Load the lines into a `List`
 2. Use an `Iterator<String>` to walk the `List`
 3. Detect lines of interest: in Gutenberg files, there's a Front Matter with,
 notably, the `Title: xxx` and `Author: xxx` lines. We also look for bookshelf
 boundaries with `##BOOKSHELF##`
 4. If we find the title, we temporarily store it.
 5. Then we find the author, combine that with the title and put it in a `List`
 for the current bookshelf.
 6. When finding a bookshelf boundary, we print the content of the current bookshelf
 and `clear()` the collection. This is also the point where we regularly GC and
 report about memory usage.

### Running The Imperative Version
Running this part of the program prints the following output 7 times:

```console
Found new bookshelf of 8 books:
[Alice’s Adventures in Wonderland by Lewis Carroll, Beowulf by  , Dracula by Bram Stoker, The Works of Edgar Allan Poe by Edgar Allan Poe, Grimms’ Fairy Tales by The Brothers Grimm, Pride and Prejudice by Jane Austen, The Adventures of Sherlock Holmes by Arthur Conan Doyle, The Republic by Plato]
Memory in use while reading: 97MB
```

Notice the memory consumption: 97MB (welp!)

## The `Stream` And `Flux` Version

We already saw how to load the file lines in a `Stream<String>` and convert it
to a `Flux<String>` properly.

Let's look at implementing the bookshelf algorithm in a reactive way:

```java
private static Flux<String> fluxVersion(Path path) {
  final Runtime runtime = Runtime.getRuntime();

  return fromPath(path)
      .filter(s -> s.startsWith("Title: ") || s.startsWith("Author: ")
          || s.equalsIgnoreCase("##BOOKSHELF##"))
      .map(s -> s.replaceFirst("Title: ", ""))
      .map(s -> s.replaceFirst("Author: ", " by "))
      .windowWhile(s -> !s.contains("##"))
      .flatMap(bookshelf -> bookshelf
          .window(2)
          .flatMap(bookInfo -> bookInfo.reduce(String::concat))
          .collectList()
          .doOnNext(s -> System.gc())
          .flatMapMany(bookList -> Flux.just(
              "\n\nFound new Bookshelf of " + bookList.size() + " books:",
              bookList.toString(),
              String.format("Memory in use while reading: %dMB\n", (runtime.totalMemory() - runtime.freeMemory()) / (1024 * 1024))
          )));
}
```

 1. We work on a `Flux<String>` of all the lines in the file (lazily loaded
   thanks to the underlying `Stream`)
 2. We filter out most of the text to only keep title/author info and bookshelf
 boundaries.
 3. We use `map` to remove the `Title: ` and `Author: ` prefixes, preparing for
 the creation of a book information `String`.
 4. We use `windowWhile` to group bookshelves into windows, sub-sequences that
 include all the data except the window separator. This gives us a `Flux<Flux<String>>`.
 5. We use `flatMap` to process each bookshelf window and go back to a `Flux<String>`:
  1. Regroup title and author lines using `window(2)`
  2. Concatenate the contents of the window into a single `String` per book
  3. Collect a `List<String>` of the books in the bookshelf.
  4. Perform a GC on the side, like we did in the imperative version, using `doOnNext`.
  5. Now we have reactively collected a `List<String>` of books in the current
		bookshelf, which is entirely processed. We use `flatMapMany` to emit the 3
		Strings we'll want to print out: "Found new Bookshelf...", the actual list
		of books and a report of the memory in use.

That's it: we now have a `Flux<String>` that represents what to output.

### Running The Reactive Version
> Nothing Happens Until You Subscribe

Here, we have represented our algorithm in the form of a `Flux`. But until you
call some form of `subscribe()` (including `block*()` methods), nothing will
happen. `Flux` is lazy by default, what's called a _"Cold Sequence"_.

Since we'll be running it in a console application's `main()` method, we need
to block until the processing is finished. The best way to print the results and
wait for the end of processing is to use `Flux#blockLast()`:

```
Flux<String> books = fluxVersion(path);
books.doOnNext(System.out::println)
     .blockLast();
```

This produces the following output:

```console
Found new Bookshelf of 8 books:
[Alice’s Adventures in Wonderland by Lewis Carroll, Beowulf by  , Dracula by Bram Stoker, The Works of Edgar Allan Poe by Edgar Allan Poe, Grimms’ Fairy Tales by The Brothers Grimm, Pride and Prejudice by Jane Austen, The Adventures of Sherlock Holmes by Arthur Conan Doyle, The Republic by Plato]
Memory in use while reading: 3MB
```

3MB, yay :grin:

# Conclusion

{{% alert note %}}
One caveat of the `Stream`-to-`Flux` conversion is that a `Stream` cannot be
reused whereas a `Flux` _could_ be subscribed to several times.

The `fromStream`
factory method is currently[^1] effectively limited to a single subscription,
as it takes the `Stream` rather than a `Supplier<Stream>` that could be reused
for further subscriptions.You can work around that by using `defer` though:

```java
Flux<String> lines = Flux.defer(() -> Flux.fromStream(Files.lines(thePath)));
```

[^1]: As of this writing, [reactor-core](https://github.com/reactor/reactor-core/) v3.1.1.RELEASE

{{% /alert %}}

The full code is
available in a [:page_facing_up:gist](https://gist.github.com/simonbasle/0167a1f833a19724646bc7eb27e4346b),
complete with a `main` that asks you for a text file to load and runs the
reactive version then the imperative one :+1:

```console
Please enter a path to a large text file [/Users/sbasle/bookshelf.txt]:
Found /Users/sbasle/bookshelf.txt, of size 33MB
Memory in use before reading: 0MB
```

And... that's the end of our post. **Happy Reactive Coding!**