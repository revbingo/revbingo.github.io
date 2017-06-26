---
layout: post
title:  "Kotlin: Refactoring to a DSL"
date:   2017-06-25 20:47:46 +0100
categories: kotlin
---
There's a *lot* of interest in Kotlin right now, thanks to [Google's involvement](https://developer.android.com/kotlin/index.html), and rightfully so. My own voyage with Kotlin started a couple of years ago, when I tried it out by porting an old side project, [SPIFF](https://github.com/revbingo/SPIFF), the Simple Parser for Interesting File Formats. I was very quickly sold on Kotlin as whole chunks of code disappeared, and what was left became much more concise. How overjoyed I was, then, to discover over the last couple of months that for once I seem to have backed the right language horse. [Blog](https://android.jlelse.eu/kotlin-best-features-a3facac4d6fd) [posts](https://blog.codecentric.de/en/2016/04/kotlins-killer-features/) [about](https://fossbytes.com/kotlin-features-android-programming/) [the](https://medium.com/@octskyward/why-kotlin-is-my-next-programming-language-c25c001e26e3) [killer](http://petersommerhoff.com/dev/kotlin/kotlin-for-java-devs/) [features](https://m.signalvnoise.com/kotlin-its-the-little-things-8c0f501bc6ea) [are](https://frozenfractal.com/blog/2017/2/10/10-cool-things-about-kotlin/) [ten-a-penny](http://blog.danlew.net/2017/05/17/why-kotlin/), so today we're going to actually get hands-on, and see how we can refactor some imperative Kotlin code into something resembling a Domain Specific Language. 

The aim of SPIFF is to provide a DSL for an executable specification of binary file formats. Examples of these sorts of files would be bitmaps, MP3s, or zip files. Parsing these files requires one to know the specification e.g. the file starts with a literal string, followed by 2 bytes that represent the length of the header, followed by a byte for the length of the description, followed by a string of that length etc. You often need to do some element of conditional computation, so the DSL needs to support some level of logic, branching and looping. In the original SPIFF, this DSL was implemented using JavaCC, and the specification file was "compiled" to a set of instruction objects that were executed at "runtime".  Here's an example of reading ID3v1 tags from an MP3 file, using SPIFF's DSL. 

```
# fileLength is a special variable that is made available

# Jump to 128 bytes before the end of the file
.jump fileLength - 128

string('TAG')  tagLiteral
string(30) title
string(30) artist
string(30) album
string(4)  year

# Tricky! ID3v1.1 hijacks the last two bytes of the comment field, one for a null byte
# and one for the track number

# Assume it's v1.1
string(28) comment
byte       zeroByte

# if the last byte was not a null byte, it must have been a v1.0 comment
.if(zeroByte != 0) {
	# go back to where the comment started, and read 30 bytes instead
	.jump &comment
	string(30) comment
} .else {
	# otherwise just carry on
	byte		trackNumber
}
byte 		genre
```

Hopefully it's straightforward to understand. The only thing that is perhaps not immediately obvious is that prefixing a variable with an ampersand, such as in `&comment`, returns the "address" from which that variable was last read. This can be useful to jump back to a particular place in the file, or to calculate how many bytes have been read since a particular point, for instance if a section is padded to a particular length. For the record, SPIFF does not have any variable scope - if the same name is used for a variable in two places, or it's used in a loop, it simply holds the last value that was read.

Our aim today is to implement the same code, but in Kotlin. Better than that, we're going to start from a naïve piece of code for reading such files, and refactor our way to greatness!  Let's see how close we can get to the original SPIFF format. 

Here's that first implementation. Hopefully you'll see at least one or two very obvious improvements we can make. If you're following along, change the MP3 to a song of your choosing (but you won't regret it if you seek out a version of [Shaking Through](https://www.youtube.com/watch?v=wdTujLHFsSU)).

{% highlight kotlin %}
fun main(args: Array<String>) {

    // get the file, and use NIO to read it into a ByteBuffer
    val file = File("ShakingThrough.mp3")
    val channel = FileInputStream(file).channel

    val buffer = ByteBuffer.allocate(file.length().toInt())
    channel.read(buffer)

    //flip() puts us back at the start of the buffer, ready to read
    buffer.flip()

    //ID3 tags occupy the last 128 bytes of the file
    buffer.position(file.length().toInt() - 128)

    val tagArray = ByteArray(3)
    buffer.get(tagArray)
    val tag = String(tagArray).trim { char -> char.isWhitespace() or (char == 0x00.toChar()) }
    if(tag != "TAG") exitProcess(1)

    val titleArray = ByteArray(30)
    buffer.get(titleArray)
    val title = String(titleArray).trim { char -> char.isWhitespace() or (char == 0x00.toChar()) }

    val artistArray = ByteArray(30)
    buffer.get(artistArray)
    val artist = String(artistArray).trim { char -> char.isWhitespace() or (char == 0x00.toChar()) }

    val albumArray = ByteArray(30)
    buffer.get(albumArray)
    val album = String(albumArray).trim { char -> char.isWhitespace() or (char == 0x00.toChar()) }

    val yearArray = ByteArray(4)
    buffer.get(yearArray)
    val year = String(yearArray).trim { char -> char.isWhitespace() or (char == 0x00.toChar()) }

    //mark() sets a point we can reset() to later
    buffer.mark()

    //skip ahead and get the 29th byte
    buffer.position(buffer.position() + 28)
    val zeroByte = buffer.get()

    //go back to where we were
    buffer.reset();

    //if it's zero, this is (or might be) ID3 v1.1, and the comment is 28 bytes
    val comment = if(zeroByte.toInt() == 0x00) {
        val commentArray = ByteArray(28)
        buffer.get(commentArray)
        String(commentArray).trim { char -> char.isWhitespace() or (char == 0x00.toChar()) }
    } else {
        //if it's not zero, it's definitely ID3 v1, and the comment is 30 bytes
        val commentArray = ByteArray(30)
        buffer.get(commentArray)
        String(commentArray).trim { char -> char.isWhitespace() or (char == 0x00.toChar()) }
    }

    //for ID3 v1.1, we get a byte for the track number after the comment
    val trackNumber = if(zeroByte.toInt() == 0x00) {
        buffer.position(buffer.position() + 1)
        buffer.get().toInt()
    } else {
        0
    }

    val genre = buffer.get().toInt()

    println("""
        Title: $title
        Artist: $artist
        Album: $album
        Year: $year

        Comment: $comment
        Track Number: $trackNumber
        Genre: $genre
    """)
}
{% endhighlight %}

Run it, and you'll get this (or something similar for your own file)

```
        Title: Shaking Through
        Artist: R.E.M.
        Album: Murmur
        Year: 1983

        Comment: 
        Track Number: 10
        Genre: 17
```

Hey, it works!  You can sort of see how the code maps to the original SPIFF specification, with some slight changes when reading the comment so we can make use of the fact that, in Kotlin, `if` statements return values, which means we can make `comment` immutable.

If you've got any sort of [spidey sense](https://www.youtube.com/watch?v=5Kek3GqbsTk), you'll already be itching to refactor that duplicated lump of code that reads a string from the buffer.  `ByteBuffer` already has methods to read primitive datatypes - `getLong()`, `getInt()` etc. - but doesn't have any methods for reading `String`s, and by now you're possibly yelling "[Extension functions](https://kotlinlang.org/docs/reference/extensions.html)" at the screen.  Let's do it. 

{% highlight kotlin %}
fun ByteBuffer.readString(length: Int): String {
    val array = ByteArray(length)
    this.get(array)
    return String(array).trim { char -> char.isWhitespace() or (char == 0x00.toChar()) }
} 
{% endhighlight %}

And we can implement it in the main function

{% highlight kotlin %}
    ...
    val tag = buffer.readString(3)
    if(tag != "TAG") exitProcess(1)

    val title = buffer.readString(30)
    val artist = buffer.readString(30)
    val album = buffer.readString(30)
    val year = buffer.readString(4)
    ...
{% endhighlight %}

So far, so un-DSL-y, but much more terse and readable.  Next, we'll attack that block of set up code at the top.  We want our DSL to be the specification of the file, not the nuts and bolts of how to actually get the content from a file. Let's hide it in another method. 

{% highlight kotlin %}
fun binaryFile(fileName: String, callback: (ByteBuffer) -> Unit) {
    val file = File(fileName)
    val channel = FileInputStream(file).channel

    val buffer = ByteBuffer.allocate(file.length().toInt())
    channel.read(buffer)

    buffer.flip()

    callback(buffer)
}
{% endhighlight %}

and we'll use it like this

{% highlight kotlin %}
binaryFile("ShakingThrough.mp3") { buffer ->
    buffer.position(buffer.limit() - 128)

    ... do your stuff ...
}
{% endhighlight %}

In Java, you might have this method return the `ByteBuffer`, and then use it.  In Kotlin, we can make use of the fact that functions are first-class, so we can pass them around.  More than that, if a function is the last argument, you can take it outside the parentheses.  So `binaryFile` takes a function, into which we pass the resulting `ByteBuffer` for it to work on.  That function is really just the code that was already in our main function, only now it's within the scope of the `binaryFile` that we're working on, and that's not a bad thing, right?  Note that because we only have access to the `buffer` object, we need to use `buffer.limit()` instead of `file.length()`, but for our purposes they are the same thing.

If you're playing along at home, you'll see that the code in that block is starting to look a bit more like a specification.  But we still have a lot of references to `buffer` everywhere, which is an implementation detail that the specification shouldn't really care about.  [Functions with receivers](https://kotlinlang.org/docs/reference/lambdas.html#function-literals-with-receiver) to the rescue! A function with receiver is just a function that assumes that `this` refers to the type of object preceding the dot. You already do this without knowing it when you use extension functions. So `String.() -> Int` is a function in which `this` will be a `String`, and which will return an Int. It doesn't need to be a method already defined on that type - because you're generally passing these as code blocks to another function, you can consider them as anonymous extension functions.  Anywhere you pass a function with receiver, you could instead pass a function literal that refers to a matching method already defined on the receiver type. In the case of `String.() -> Int`, you could pass `String::toInt`.

Instead of passing the `ByteBuffer` as a parameter to the code block, let's make it the receiver. We just change the type signature of the `callback` parameter, and instead of `callback(buffer)`, we do `buffer.callback()`. 

{% highlight kotlin %}
fun binaryFile(fileName: String, callback: ByteBuffer.() -> Unit) {
    val file = File(fileName)
    val channel = FileInputStream(file).channel

    val buffer = ByteBuffer.allocate(file.length().toInt())
    channel.read(buffer)

    buffer.flip()

    buffer.callback()
}
{% endhighlight %}

Because the `ByteBuffer` is now `this` in the code block, and `this` is implicit, we can just take away all the references to `buffer`

{% highlight kotlin %}
binaryFile("ShakingThrough.mp3") {
    position(limit() - 128)

    val tag = readString(3)
    if(tag != "TAG") exitProcess(1)

    val title = readString(30)
    val artist = readString(30)
    val album = readString(30)
    val year = readString(4)

    mark()
    position(position() + 28)
    val zeroByte = get()
    reset();

    ...
}
{% endhighlight %}

Okay, now we're really starting to get somewhere! What next?  Well, that `get()` is a bit obscure now. It gets a single byte from the buffer. In the original SPIFF DSL, it's a `byte` instruction. Methods that read from the buffer should represent the datatype you're fetching, so `get()` becomes `byte()`, `readString()` becomes just `string()` etc.  We can just define these as extensions on `ByteBuffer`. We'll also add a `skip()` instruction to replace that unwieldy statement that moves 28 bytes ahead. 

{% highlight kotlin %}
fun ByteBuffer.string(length: Int): String {
    val array = ByteArray(length)
    this.get(array)
    return String(array).trim { char -> char.isWhitespace() or (char == 0x00.toChar()) }
}

fun ByteBuffer.byte(): Int = get().toInt()

fun ByteBuffer.skip(length: Int) = position(position() + length)
{% endhighlight %}

which gives us:

{% highlight kotlin %}
...
val title = string(30)
val artist = string(30)
val album = string(30)
val year = string(4)

mark()
skip(28)
val zeroByte = byte()
reset()
...
{% endhighlight %}

There's a couple more useful instructions we can introduce here. That mark-skip-read-reset lump of code is a bit ugly.  What we're really doing there is taking a peek a few bytes ahead. To make it interesting, we can genericise that to allow a caller to move in the stream, do anything they want (read a byte, a string, an int etc.), and then return a value, having reset the position to where you were before you started. That calls for passing another function with receiver.  In that function, the caller should be able to write the same DSL, so the type signature for that function will stay the same as we use in the `binaryFile` method, except now that block will return an `Any` value instead of `Unit`

{% highlight kotlin %}
fun ByteBuffer.peek(length: Int, callback: ByteBuffer.() -> Any): Any {
    mark()
    skip(length)
    return callback().apply {
        reset()
    }
}
{% endhighlight %}

We use the `apply` idiom here. Without it, we would store the return value from the callback block in another variable, then do the `reset()`, then return the value.  With `apply`, the `reset()` is performed before the value from `callback` is returned.  Note that `callback` doesn't have an explicit receiver - the receiver is `this` (the ByteBuffer), so it can be omitted to taste. Also note that inside the `apply` block, strictly `this` is the instance of `Any` returned from `callback()`, __not__ the `ByteBuffer`. If we called `this.reset()`, the compiler borks. But it's also clever enough to infer that we're calling `reset()` on `this` from the outer scope (the `peek` method). If you wanted to be explicit about it, you can use `this@peek.reset()`.  Seeing as a large part of writing DSLs is controlling which receiver is in scope at any given point, it's worth taking some time to ensure you understand these ideas.

We use it like this:

{% highlight kotlin %}
val zeroByte = peek(28) {
    byte()
}
{% endhighlight %}

At this juncture, we'll take a slight detour. So far, we've just been defining extensions on `ByteBuffer`. But soon we may need to store some state of our own, and perhaps methods that don't really relate to the buffer directly. Also, our extension functions will be available outside of our DSL, so we're leaking scope a bit. We're going to define our own class that holds on to the instance of the buffer, and delegates calls accordingly. The `binaryFile` method changes to make the new `BinaryFile` class the receiver of the callback function, and the extension methods that we defined on `ByteBuffer` just become normal members of the `BinaryFile` class. This means that the `BinaryFile` class will encapsulate the methods of our DSL, which seems like The Right Thing. 

{% highlight kotlin %}
fun binaryFile(fileName: String, callback: BinaryFile.() -> Unit) {
    val binaryFile = BinaryFile(fileName)
    binaryFile.callback()
}

class BinaryFile(fileName: String) {
    val buffer: ByteBuffer

    init {
        val file = File(fileName)
        val channel = FileInputStream(file).channel

        buffer = ByteBuffer.allocate(file.length().toInt())
        channel.read(buffer)

        buffer.flip()
    }

    fun string(length: Int): String {
        val array = ByteArray(length)
        buffer.get(array)
        return String(array).trim { char -> char.isWhitespace() or (char == 0x00.toChar()) }
    }

    fun byte(): Int = buffer.get().toInt()

    fun skip(length: Int) = buffer.position(buffer.position() + length)

    fun peek(length: Int, callback: BinaryFile.() -> Any): Any {
        buffer.mark()
        skip(length)
        return callback().apply {
            buffer.reset()
        }
    }

    fun jump(pos: Int) = buffer.position(pos)

    fun fileLength() = buffer.limit()
}
{% endhighlight %}

By adding passthrough methods for `position()` and `limit()`, we can make this refactoring without needing to change the DSL in our main method. But because `limit()` isn't very domain specific, we'll rename it to `fileLength()`, and in the original SPIFF, `position()` was called `.jump`, so we'll copy that.

The final thing we'll do for today is add a datatype for a literal string. If the specification calls for a literal string at a position, and that string isn't found, we should clearly stop parsing and throw an exception. 

{% highlight kotlin %}
fun literalString(expected: String): String {
    val actual = string(expected.toByteArray().size)
    if(actual != expected) throw RuntimeException("Expected literal $expected, found $actual")
    return actual
}
{% endhighlight %}

which we can use thus:

{% highlight kotlin %}
binaryFile("ShakingThrough.mp3") {
    jump(fileLength() - 128)

    val tag = literalString("TAG")

    val title = string(30)
    ...
}
{% endhighlight %}

Try changing "TAG" to something else, and you should see it fail.

Here's our final DSL for now:

{% highlight kotlin %}
binaryFile("ShakingThrough.mp3") {
    jump(fileLength() - 128)

    val tag = literalString("TAG")

    val title = string(30)
    val artist = string(30)
    val album = string(30)
    val year = string(4)

    val zeroByte = peek(28) {
        byte()
    }

    val comment = if(zeroByte == 0x00) {
        string(28)
    } else {
        string(30)
    }

    val trackNumber = if(zeroByte == 0x00) {
        skip(1)
        byte()
    } else {
        0
    }

    val genre = byte()

    println("""
        Title: $title
        Artist: $artist
        Album: $album
        Year: $year

        Comment: $comment
        Track Number: $trackNumber
        Genre: $genre
      """)
}
{% endhighlight %}

Hopefully you'll agree that it's not a million miles away from the original SPIFF version. Bear in mind that we haven't done any "DSL magic" here. Simply using common features of the language, namely extension functions and passing functions with receivers, we have achieved code that is concise and readable, which is really all a DSL is.

Next time, we'll edge things a little bit closer to the SPIFF version, using some [operator overloading](https://kotlinlang.org/docs/reference/operator-overloading.html). 