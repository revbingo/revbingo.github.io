---
layout: post
title:  "Kotlin: Refactoring to a DSL - Part III"
date:   2017-07-09 07:00:00 +0100
categories: kotlin
---

Welcome back to this rollercoaster ride of things that we probably shouldn't do with Kotlin, but we will, because we can! Plus, we might learn something along the way. Last time out, we got to a point where we'd pretty much taken the DSL part of the project as far as we reasonably can, at least in the scope of the MP3 example. 

But there's a couple of other things that we can explore.  Firstly, our `binaryFile` method expects a code block to be passed to it, so our specification is always going to be inline in whatever code is parsing the file. Wouldn't it be nice to make these things reusable? What if we could provide a library of different file specifications?  Remember way back in [part 1]({% post_url 2017-06-25-kotlin-refactoring-dsl %}), I mentioned that instead of passing the code block, we could pass a function reference. Let's try that. We move the code block into its own method.  Remember that we still have to agree with the type signature, so we have to declare it as an extension function on the `BinaryFile` class

{% highlight kotlin %}
fun main(args: Array<String>) {

    binaryFile("ShakingThrough.mp3", BinaryFile::asId3)
}

fun BinaryFile.asId3() {
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
{% endhighlight %}

Notice that we now pass a function reference `BinaryFile::asId3` to the `binaryFile` method. Hopefully this will help solidify the notion that a "function with receiver" is really just an anonymous extension function.  Here we've taken it from being anonymous (passed as a code block) to being named (the `asId3()` method).  We could now go and write `asZipFile()` or `asBmp()` methods if we wanted, to parse a file as a .zip or .bmp, and then we could package them up and allow others to use them in their code. 

But if your developer spidey sense is tingling yet again, you'll be thinking "isn't this what class hierarchies are for?".  And you would, of course, probably be right. These extension functions don't give us a great deal in the way of encapsulation or type safety. Conceptually, MP3s and zips and bitmaps are subtypes of `BinaryFile`. We'll create a class for MP3s, a subclass of `BinaryFile`, and move the `asId3` method there.

{% highlight kotlin %}
fun main(args: Array<String>) {
    binaryFile("ShakingThrough.mp3", MP3File::asId3)  //won't compile yet
}

class MP3File(fileName: String): BinaryFile(fileName) {
    fun asId3() {
        jump(fileLength() - 128)
        ...
    }
}
{% endhighlight %} 

At this point, you've probably got a big red blob showing a compilation error on `MP3File::asId3`. This is because the `binaryFile` method expects to be passed a reference to a method in which a `BinaryFile` is the receiver, not an `MP3File`. That's fairly simple to fix though, we just need to specify that it can be some other type, as long as that type is a subtype of `BinaryFile`. 

{% highlight kotlin %}
fun <T: BinaryFile> binaryFile(fileName: String, callback: T.() -> Unit) {

    val binaryFile = BinaryFile(fileName)
    binaryFile.callback()  //now this won't compile
}
{% endhighlight %}

We've added a generic type to the method - `<T: BinaryFile>` tells the compiler that we'll use a type `T` somewhere in the method, and the compiler should expect it to be a BinaryFile or a subtype of it. And indeed we do use it, we say that the callback method will have a receiver of type `T`. So now it happily accepts that `MP3File::asId3` is a valid argument to the `binaryFile` method.  Only now the invocation of the `callback` method doesn't compile, because we're calling it on a concrete `BinaryFile`, not an object of type `T` (which might be a subtype).  Instead of creating an instance of the `BinaryFile` supertype, we need to create an instance of `T`.  Reflection to the rescue. 

{% highlight kotlin %}
fun <T: BinaryFile> binaryFile(fileName: String, callback: T.() -> Unit) {

    val binaryFile = T::class.constructors.first().call(fileName)  //and now *this* won't compile, when will it end?!
    binaryFile.callback()
}
{% endhighlight %}

Using reflection, we can create an instance through finding its constructor. Note that, in the interests of simplicity, we make a big assumption here, which is that the only (or at least first) constructor on a subtype takes `fileName` as a parameter. But the compiler is still trying to stop us from doing something silly. The problem here is that the default behaviour of generics in the JVM is for the type to be erased at runtime, so when the program executes, it has no idea what `T` actually was when the code was compiled. Luckily, Kotlin helps you out here. By marking our generic type with the `reified` keyword, we can ask the compiler to keep hold of that type information. 

{% highlight kotlin %}
 // OMGSMHFML...
fun <reified T: BinaryFile> binaryFile(fileName: String, callback: T.() -> Unit) {  

    val binaryFile = T::class.constructors.first().call(fileName)  
    binaryFile.callback()
}
{% endhighlight %}

Nearly there, but that will also give you a compiler error. The way the compiler keeps the type information is essentially by "cutting and pasting" the code from the method everywhere that it is called, and replacing `T` with the type you specify in the call. So the final piece of this puzzle is to add the `inline` annotation to the method to allow the compiler to inline it. 

{% highlight kotlin %}
inline fun <reified T: BinaryFile> binaryFile(fileName: String, callback: T.() -> Unit) { 

    val binaryFile = T::class.constructors.first().call(fileName)  
    binaryFile.callback()
}
{% endhighlight %}

At last, the compiler is satisfied.  And if we now have subtypes to represent different types of file, you could have subtypes simply override a `parse` method, in which case you just pass the subtype class, representing the type of file you're parsing, as a generic parameter on the call to the `binaryFile` method.

{% highlight kotlin %}
fun main(args: Array<String>) {
    binaryFile<MP3File>("ShakingThrough.mp3")
}

inline fun <reified T: BinaryFile> binaryFile(fileName: String) {
    val binaryFile = T::class.constructors.first().call(fileName)
    binaryFile.parse()
}

abstract class BinaryFile(fileName: String) {
    ...
    abstract fun parse()
    ...
}
{% endhighlight %}

These changes mean that we can easily supply a "library" of file types for end users to use, all based on the DSL.  

That also means we can start providing some specialisms, depending on the file type. At the moment our DSL is a generic language for parsing different data types from a file. Whilst we're dealing with fairly simple data, that's fine, but sometimes we might encounter some rather more complex data types that are particular to the type of file we're reading. For example, in a zip file, the `lastModifiedDate` is stored in [MS-DOS date format](http://www.vsft.com/hal/dostime.htm), which allows us to store a date in just 2 bytes. Efficient, but really not very easy to parse using the tools we have in our DSL so far. We'll end up just cluttering our specification again with all sorts of parsing logic. 

But we can now extend our DSL with the ability to parse such dates, and we can keep the scope of that to our `ZipFile` parser. First, let's implement the class and parse the first few fields.  Find a zip file of your choice to test on.

{% highlight kotlin %}
fun main(args: Array<String>) {
    binaryFile<ZipFile>("archive.zip")
}

class ZipFile(fileName: String): BinaryFile(fileName) {
    override fun parse() {
        setorder(ByteOrder.LITTLE_ENDIAN)

        int() .. "localFileHeaderSignature"
        short() .. "versionNeededToExtract"
        short() .. "generalPurposeBitFlag"
        short() .. "compressionMethod"
    }
}
{% endhighlight %}

Implementations of the `setorder`, `int` and `short` instructions were exercises for the reader in [part 2]({% post_url 2017-06-25-kotlin-refactoring-dsl %}). Of course you did them. If you didn't, have a go now, I'll wait...

Now we have that thorny date time in MSDOS format. In this format, we'll read 2 bytes for the time. Bits 0-4 are the seconds, halved. Bits 5-10 are the minute, and the rest of the bits are the hour. In order to parse these, we'll take those two bytes, apply a bit mask, and then shift the bits to get the final number. We do likewise for the date, in which bits 0-4 are the day, 5-8 are the month, and 9-15 are the number of years since 1980. We introduce a method in the ZipFile class to read such dates, and turn them into a `LocalDateTime`.

{% highlight kotlin %}
class ZipFile(fileName: String): BinaryFile(fileName) {
    ...
    fun msDosDateTime(): String {
        val timeElement = buffer.short
        val dateElement = buffer.short

        val seconds =(timeElement.toInt() and 0b0000000000011111) * 2
        val minutes = timeElement.toInt() and 0b0000011111100000 shr 5
        val hours   = timeElement.toInt() and 0b1111100000000000 shr 11

        val day   = dateElement.toInt() and 0b0000000000011111
        val month = dateElement.toInt() and 0b0000000111100000 shr 5
        val year  =(dateElement.toInt() and 0b1111111000000000 shr 9) + 1980

        return LocalDateTime.of(year, month, day, hours, minutes, seconds)
    }
}
{% endhighlight %}

Now, we just have to use it in the `parse` method:

{% highlight kotlin %}
    override fun parse() {
        setorder(ByteOrder.LITTLE_ENDIAN)

        int() .. "localFileHeaderSignature"
        short() .. "versionNeededToExtract"
        short() .. "generalPurposeBitFlag"
        short() .. "compressionMethod"

        msDosDateTime() .. "lastModified"

        println(+"lastModified")

        ...draw the rest of the owl...
    }
{% endhighlight %}

In my case, I get `2016-02-29T09:11:50`, which indeed matches the last modified time when I view the file properties. It works!

One last trick up our sleeve. Given concrete subtypes for our file types, it would be nice to regain some of the type safety and code completion, rather than dealing the loose types and keys we're getting from the map when using the `+` operator. Not to mention, that operator is only valid _inside_ the DSL, whereas once we've parsed a file, we may want to reference it and its attributes in the rest of our code. First, let's make our `binaryFile` method return a reference to the object we've just created. The method specifies a return type `T`, and adds a return statement.

{% highlight kotlin %}
fun main(args: Array<String>) {
    val theZip = binaryFile<ZipFile>("archive.zip")
}

inline fun <reified T: BinaryFile> binaryFile(fileName: String): T {
    val binaryFile = T::class.constructors.first().call(fileName)
    binaryFile.parse()

    return binaryFile
}
{% endhighlight %}

Secondly, we want the ZipFile class to have fields that represent the attributes we've read from the file.  The naive way to do this would be to have methods that make a call into the map - for example `fun lastModified() = valueMap.get("lastModified")`. Thankfully, Kotlin has a terser way of doing this, using [delegated properties](https://kotlinlang.org/docs/reference/delegated-properties.html). The `Map` class implements the `Delegate` interface - classes implementing this interface implement `getValue` and `setValue` methods, that get passed the name of the property being delegated, so in the case of a map it can look up a value in the map using the property name as the key. We indicate delegated properties using the `by` keyword. 

{% highlight kotlin %}
class ZipFile(fileName: String): BinaryFile(fileName) {

    val lastModified: LocalDateTime by valueMap
    ...
}

fun main(args: Array<String>) {
    val theZip = binaryFile<ZipFile>("archive.zip")
    println(theZip.lastModified)
}
{% endhighlight %}

Run that code, and you should see that it all works splendidly.  Also note that you have a certain amount of type safety back - if you change the type of `lastModified`, it will blow up at runtime. 

That (probably) concludes this whistlestop tour of refactoring to a Kotlin DSL.  I hope you've enjoyed it and can find a way to shoehorn some of these techniques into your own projects, but use them wisely!  Some of these things are definitely clever, even fun, but nothing beats code that is simple and concise. 