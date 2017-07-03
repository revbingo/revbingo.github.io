---
layout: post
title:  "Kotlin: Refactoring to a DSL - Part II"
date:   2017-07-03 07:00:00 +0100
categories: kotlin
---
ICYMI, in [part 1](https://revbingo.github.io/kotlin/2017/06/25/kotlin-refactoring-dsl.html) we took some standard imperative code for reading tags from an MP3, and refactored that code into something approaching a domain specific language. No magic was used. For the most part, we just used some common sense, some clean code principles and the abilities of Kotlin to produce code that was concise and clear. We only introduced a few types of instruction - `string()`, `literalString()`, `byte()`, `skip()`, `jump()` and `peek()` - but it should be fairly clear how you might expand that to include other data types and operations. I leave it as an exercise for the reader to implement the `short`, `int`, `float`, `long` and `double` datatypes, and the `setorder` and `setencoding` instructions - you can find out what these do in the [SPIFF Cheat Sheet](https://github.com/revbingo/SPIFF/wiki/Cheat-Sheet).

The DSL that we have is perfectly workable, and any other day I might recommend to leave it there, but what fun would that be?  After all, our aim at the outset was to see how close we could get to the original SPIFF DSL, which previously required many hours of sweating over [JavaCC configuration](https://github.com/revbingo/SPIFF/blob/master/src/main/java/com/revbingo/spiff/parser/gen/spiff.jjt). Plus, we might learn something along the way. 

For a start, the fields in the specification are identified by the names of the variables into which they are read, not really as part of the DSL. Again, maybe that's not a bad thing, but we do these things because we can, not because we should. In the SPIFF DSL, the name of the field follows the datatype. To achieve this, we'll use some operator overloading. Kotlin's operator overloading is relatively sane - you can't make up your own operators á la Scala - but as you'll see you can still do some pretty wacky things with it if you really want to.  And I really want to, for fun at least. We'll do something I've seen refererred to as "operator punning" - using an operator because it *looks* good, not because it makes logical sense. A common example of this is using the division operator `/` as a way to compose filesystem paths. 

My operator of choice today will be the `..` operator, which translates to `rangeTo`. In this case, we'll want it to be applied to an arbitrary type (it might be a `String`, or an `Int`, or a `Float`), and the second argument will be a string specifying the name of the field.  Also don't forget that we're refactoring here, so we're looking to make the smallest change we can that moves things forward without breaking it. So the `rangeTo` will need to return the value of the left operand, so that for now we can continue to assign it to a variable. 

{% highlight kotlin %}
class BinaryFile(fileName: String) {
    ...

    operator fun Any.rangeTo(name: String): Any {
        return this
    }
}
{% endhighlight %}

To define an overloaded operator, we need to know the name of the method that it translates to.  We also need to mark the function with the `operator` keyword.  Finally, like normal extension functions, we can make them apply to a particular type simply by putting the type in the function signature.  In this case, we want it to apply to `Any` type.  However, note that we're defining it within the `BinaryFile` class. This means that this extension function is only in scope for methods within that class (i.e. in our DSL).  If you try to use it outside of the DSL block, it won't work, so user's of our code won't get any nasty suprises.

At this point, we just return `this`, we'll change that in a minute. But you can now start using the `..` operator to annotate the DSL.

{% highlight kotlin %}
binaryFile("ShakingThrough.mp3") {
    jump(fileLength() - 128)

    val tag = literalString("TAG") .. "tag"

    val title = string(30) .. "title"
    val artist = string(30) .. "artist"
    val album = string(30) .. "album"
    val year = string(4) .. "year"

    val zeroByte = peek(28) {
        byte() .. "zeroByte"
    }
{% endhighlight %}

By the way, if you wanted to use a named method, instead of an operator, you could achieve the same thing using an [infix function](https://kotlinlang.org/docs/reference/functions.html#infix-notation). The main reason I didn't do that here was because all the neat little words that make sense - `is`, `as`, `to` - are all already used in Kotlin. Then again, if you *really* wanted to, even that's possible, by using backticks around the method name. But that would just be silly.

Now we can look to remove that duplication, where we have both a variable name and the name "annotation" on the other side. But the variables are useful because we need to do some computation later - for example, checking the value of `zeroByte`.  What we need is a way to obtain the value of a field given its name. Let's store values in a map. Like the original SPIFF, there's nothing clever with scope or arrays - the last value read for a given name is the only one you can retrieve.

{% highlight kotlin %}
class BinaryFile(fileName: String) {
    val buffer: ByteBuffer
    private val valueMap = mutableMapOf<String, Any>()

    ...

    operator fun Any.rangeTo(name: String): Any {
        valueMap.put(name, this)
        return this
    }
}
{% endhighlight %}

Quite simply, when the `rangeTo` operator is invoked, we store the operand on the left (`this`) in the map with the key being the operand on the right (`name`). Now we just need a way to retrieve that value. One way you could do it would be with a simple method, maybe `getValue("name")`, but that's a little unwieldy, and no fun at all.  Luckily, some of the operators that can be overloaded are *unary*, that is, they only have one operand, so it's quite handy for doing stuff like this:

{% highlight kotlin %}
class BinaryFile(fileName: String) {
    ...

    operator fun String.unaryPlus(): Any? {
        return valueMap.get(this)
    }
}
{% endhighlight %}

This time, we apply the operator to a `String`, which will be the name that we want to retrieve from the map, and it returns an `Any?`, which is nullable because we might not find that name in the map.  Now, we can use e.g. `+"title"` to get the value of the `title` field, and remove the assignment to a `val`.

{% highlight kotlin %}
fun main(args: Array<String>) {

    binaryFile("ShakingThrough.mp3") {
        jump(fileLength() - 128)

        literalString("TAG") .. "tag"

        string(30) .. "title"
        string(30) .. "artist"
        string(30) .. "album"
        string(4) .. "year"

        peek(28) {
            byte() .. "zeroByte"
        }

        if(+"zeroByte" == 0x00) {
            string(28) .. "comment"
        } else {
            string(30) .. "comment"
        }

        if(+"zeroByte" == 0x00) {
            skip(1)
            byte() .. "trackNumber"
        } else {
            0
        }

        byte() .. "genre"

        println("""
            Title: ${+"title"}
            Artist: ${+"artist"}
            Album: ${+"album"}
            Year: ${+"year"}

            Comment: ${+"comment"}
            Track Number: ${+"trackNumber"}
            Genre: ${+"genre"}
         """)
    }
}
{% endhighlight %}

Oh boy, now we're really getting close! There's just one more thing we need to implement to be able to almost exactly recreate the original SPIFF DSL, and that's the ability to retrieve the position at which a variable was read. This will allow us to recreate the `.jump &comment` in the original. This is actually a little tricky, as when the buffer is read (and therefore you can know its starting position) you don't know the name, and once you know the name, the buffer position has moved on. We'll solve this by storing the buffer position in a variable `lastPos`, either after a value has been read (and therefore will be the position of the next piece of data), or when we jump/skip/peek to move the buffer position.  It's not particularly pretty and it's full of potential bugs (what happens if you don't assign a name to a field?), but it'll do for our purposes here:

{% highlight kotlin %}
class BinaryFile(fileName: String) {
    val buffer: ByteBuffer
    private val valueMap = mutableMapOf<String, Any>()
    private val positionMap = mutableMapOf<String, Int>()
    private var lastPos = 0;

    ...

    fun skip(length: Int) {
        buffer.position(buffer.position() + length)
        lastPos = buffer.position()
    }

    fun peek(length: Int, callback: BinaryFile.() -> Any): Any {
        buffer.mark()
        skip(length)
        return callback().apply {
            buffer.reset()
            lastPos = buffer.position()
        }
    }

    fun jump(pos: Int) {
        buffer.position(pos)
        lastPos = pos
    }

    ...

    operator fun Any.rangeTo(name: String): Any {
        valueMap.put(name, this)
        positionMap.put(name, lastPos)
        lastPos = buffer.position()
        return this
    }

    ...
}
{% endhighlight %}

And we can now add another unary operator to get the *position* of a variable instead of its value.  Unfortunately, there's no unary operator using an ampersand like the original, so I think we'll use `!`, which is a `not`:

{% highlight kotlin %}
class BinaryFile(fileName: String) {
    ...
    operator fun String.not(): Int {
        return positionMap.get(this) ?: throw RuntimeException("Variable $this not found")
    }
}
{% endhighlight %}

We throw an exception if the variable was not found in the map, so we don't have to deal with nullability issues later. Let's now use this to refactor the two separate `if` blocks.  We'll take the same approach as the original spec, which was to assume it's v1.1 (comment is 28 bytes), and only go back if we find that the 29th byte of the comment is not a null byte. 

{% highlight kotlin %}
binaryFile("ShakingThrough.mp3") {
    ...
    string(28) .. "comment"
    byte() .. "zeroByte"

    if(+"zeroByte" != 0x00) {
        jump(!"comment")
        string(30) .. "comment"
    } else {
        byte() .. "trackNumber"
    }
    ...
}
{% endhighlight %}

With that, I think we're in a really good place - our Kotlin version is really quite close to the SPIFF DSL. We've used operator overloading to take away a chunk of syntactic noise, and it's also important to note that by defining our operator overloads inside our `BinaryFile` class, we've limited their scope to just being available in our DSL. Here's the full final code:

{% highlight kotlin %}
import java.io.File
import java.io.FileInputStream
import java.nio.ByteBuffer

fun main(args: Array<String>) {

    binaryFile("ShakingThrough.mp3") {
        jump(fileLength() - 128)

        literalString("TAG") .. "tag"

        string(30) .. "title"
        string(30) .. "artist"
        string(30) .. "album"
        string(4) .. "year"

        string(28) .. "comment"
        byte() .. "zeroByte"

        if(+"zeroByte" != 0x00) {
            jump(!"comment")
            string(30) .. "comment"
        } else {
            byte() .. "trackNumber"
        }

        byte() .. "genre"

        println("""
            Title: ${+"title"}
            Artist: ${+"artist"}
            Album: ${+"album"}
            Year: ${+"year"}

            Comment: ${+"comment"}
            Track Number: ${+"trackNumber"}
            Genre: ${+"genre"}
         """)
    }
}

fun binaryFile(fileName: String, callback: BinaryFile.() -> Unit) {

    val binaryFile = BinaryFile(fileName)
    binaryFile.callback()
}

class BinaryFile(fileName: String) {
    val buffer: ByteBuffer
    private val valueMap = mutableMapOf<String, Any>()
    private val positionMap = mutableMapOf<String, Int>()
    private var lastPos = 0;

    init {
        val file = File(fileName)
        val channel = FileInputStream(file).channel

        buffer = ByteBuffer.allocate(file.length().toInt())
        channel.read(buffer)

        buffer.flip()
    }

    fun literalString(expected: String): String {
        val actual = string(expected.toByteArray().size)
        if(actual != expected) throw RuntimeException("Expected literal $expected, found $actual")
        return actual
    }

    fun string(length: Int): String {
        val array = ByteArray(length)
        buffer.get(array)
        return String(array).trim { char -> char.isWhitespace() or (char == 0x00.toChar()) }
    }

    fun byte(): Int = buffer.get().toInt()

    fun skip(length: Int) {
        buffer.position(buffer.position() + length)
        lastPos = buffer.position()
    }

    fun peek(length: Int, callback: BinaryFile.() -> Any): Any {
        buffer.mark()
        skip(length)
        return callback().apply {
            buffer.reset()
            lastPos = buffer.position()
        }
    }

    fun jump(pos: Int) {
        buffer.position(pos)
        lastPos = pos
    }

    fun fileLength() = buffer.limit()

    operator fun Any.rangeTo(name: String): Any {
        valueMap.put(name, this)
        positionMap.put(name, lastPos)
        lastPos = buffer.position()
        return this
    }

    operator fun String.unaryPlus(): Any? {
        return valueMap.get(this)
    }

    operator fun String.not(): Int {
        return positionMap.get(this) ?: throw RuntimeException("Variable $this not found")
    }
}
{% endhighlight %}

Next time, with a following wind, we'll continue to follow a path that shouldn't really be followed, and use this code to explore generics, function references and delegates.