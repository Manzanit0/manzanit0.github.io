---
layout: post
title: "WTF are streams?"
author: Javier García
description: "Streams as a concept explained, with examples in multiple languages"
category: computing
apprenticeship: false
tags: computing, streams, elixir, java, c#
---

Streams have always been a complicated topic, but with a little help from some examples, hopefully we can make
it right today.

Streams are, in essence, a sequence of data elements which are available over time. Just like an actual stream
of water, the data flows/becomes available, instead of having it all from the beginning. There are many benefits
to this although the two most important are a huge enhancement to performance and the fact that data is not
always available inmediately.

Like just pointed, one of the main reasons to use streams is because sometimes, **data is not inmediately
available**. For example, if you're listening to a weather streaming API, the current temperature is
calculated in the current moment, so (1) the data is infinite as long as you continue listening to the service,
and (2) new data is available each minute, as it's produced. This can't be modeled with a finite amount of data,
so that's when we decide to model it with an infinite stream of data.

Since the size of a stream is undefined, they are potentially unlimited, it's important to remember that we can't
operate on them as a whole, like we're used to when working with lists or arrays. This way, functions which are
applied to streams usually return another stream with the data modified. These are called *filters*, and when
chained, they form pipelines.

Another benefit mentioned was **performance**. When processing very big amounts of data, this can
usually end up in a very big memory hit to our computer. If you try to read a 20Gb file with data about cats,
process it and then send it to a friend, that would mean that apart from whatever amount of memory our application
uses, we're going to load an extra 20Gb to memory. Most laptops would die! But that doesn't mean it isn't doable.
If instead of reading the file as a whole we model it as a stream of data, we can read one line at a time,
process those lines and send them, also as a stream. This would make our application just use an extra few bytes
instead of the awful 20Gb. When thinking about streams, we always have to think about them as a nice sequence of
manageable chunks of data our application can process – once it's finished with a chunk, it can read and process
the next.

## Simplified I/O over a String, in Elixir

Our first practical approach to streams is going to be the `StringIO` module in Elixir. A `StringIO` is not really
an actual stream, but just a wrapper around a string that allows us to apply some of the standard I/O operations
over streams to the string. For us it's going to be perfect because we can use it to familiarize with the operations:

1. **Open:** obtains exclusive access over the resource.
2. **Read/Write:** basically reads or writes chunks of date from/to the stream.
3. **Close:** returns the resources to the OS (operative system).

First, we open a string IO. From the response of the `open/1` function, we can see that it gives us a reference
to the stream.

```elixir
iex>  {:ok, pid} = StringIO.open("content")
{:ok, #PID<0.470.0>}
```

If we want to inspect the contents of the stream, we can do so via the `contents/1` function, by passing it the
previously obtained reference. In elixir the `contents/1` function will always return a tuple with the input buffer
and the output buffer, such as `{"in", "out"}`. In this case, the output buffer will be empty since we haven
written anything to it.

```elixir
iex(1)> StringIO.contents(pid)
{"content", ""}
```

Since `StringIO` is a a wrapper which models the string as a stream, we can use the standard functions to read and
write to streams from the `IO` module. In this case, to write some content, we can use the `write/2` function.
Notice how we now have data both in the input buffer and the output buffer.

```elixir
iex(2)> IO.write(pid, "written")
:ok

iex(3)> StringIO.contents(pid)
{"content", "written"}
```

Most stream modules in most languages also give us a way to *flush* the content, which means it forces any bytes in
the stream to be written out. This applies to the output buffer.

```elixir
iex(4)> StringIO.flush(pid)
"written"

iex(5)> StringIO.contents(pid)
{"content", ""}
```

Lastly, if we want to read from the input buffer, we can use the `read/2` function, thus emptying the stream of data:

```elixir
iex(6)> IO.read(pid, :all)
"content"

iex(7)> StringIO.contents(pid)
{"", ""}
```

Notice how in this specific case Elixir models the `StringIO` as a tuple with both an input buffer and and output
buffer, a buffer to which we can write and one from where we can read from.

## I/O on a File, in C#

Moving to a more practical example, we're going to check out how to work with files in streams. When working with
files, opening and closing the streams starts to take much more importance, but we'll discuss that later.

In .NET land, in order to create a file, we can make use of the `File.Create` function. This will provide us with a 
`FileStream` which models our file, so we can write to it. Once we open the stream and write to it, we will have to
close it in order to *persist* those changes and free the resources the OS has given us. Furthermore, to read the
content again, we will reopen another stream with `File.OpenRead` and read byte by byte. The snippet looks as follows,
credits to the MSDN:

```csharp
using System;
using System.IO;
using System.Text;

namespace StreamTime
{
    public class FileTheStream
    {

        public static void Main()
        {
            const string path = @"/Users/jgarcia/Desktop/example.txt";

            //Create the file.
            using (FileStream fs = File.Create(path))
            {
                AddText(fs, "This is some text");
                AddText(fs, "This is some more text,");
                AddText(fs, "\r\nand this is on a new line");
                AddText(fs, "\r\n\r\nThe following is a subset of characters:\r\n");

                for (var i = 1; i < 120; i++)
                {
                    AddText(fs, Convert.ToChar(i).ToString());
                }
            }

            //Open the stream and read it back.
            using (FileStream fs = File.OpenRead(path))
            {
                var b = new byte[1024];
                var temp = new UnicodeEncoding();
                while (fs.Read(b,0,b.Length) > 0)
                {
                    Console.WriteLine(temp.GetString(b));
                }
            }
        }

        private static void AddText(Stream fs, string value)
        {
            var info = new UnicodeEncoding().GetBytes(value);
            fs.Write(info, 0, info.Length);
        }
    }
}
```

As you can see, code-wise, we're just opening a stream, reading, writing, and closing it. Well, you might be wondering
where the closing is happening. Take into account that in C#, whenever we use the `using` keyword with a resource,
once it's finished using it, it closes the resource – Just like Java's *try with resources*. But let's talk further
about the opening and closing of streams.

### Why do streams have to be opened or closed?

We mentioned that every time we open a stream the OS needs to dedicate resources, but we never talked about which,
or how. Most of the time, it depends on the nature of the stream which we have opened – it's not the same to open
a socket than to open a file, etc. For the time being, we can concentrate on the files.

Whenever a new file stream is open, the OS dedicates a file descriptor, commonly known as a file handle, to the
application. A file descriptor basically is a number that uniquely identifies an open file in a computer's OS – it
describes a data source, and how it can be accessed. File descriptors point towards the kernel's global file table,
which contains information such as the offset and the access restrictions of the stream.

As you can imagine, file descriptors isn't like memory – if you don't return them to the OS the situation can grow
ugly fast. It's just a matter of time until the application crashes in a long running application. This is usually
called a file-handle leak. In Windows computers, when you try to delete a file, usually one of the consequences is 
that it says it's being used by another program. The resource hasn't been freed properly.

Luckily, most languages nowadays provide us with constructs which allow us to free those resources appropiately.
Like mentioned, we have statements like `using` in C# or `try` with resources in Java in our toolbox. In Elixir
though, you have to close it with the `IO.close/1` function.

## I/O over a socket, in Java

Another scenario where streams are used is when an application opens a socket. In order for two processes to
communicate, they each need to open a socket, and then send their messages through there – sockets send and receive
messages through streams. Showing up next is a snippet which contains a very simple echo server developed in Java.
Notice how the socket contains both an input stream and an output stream, and how we read data the data sent through
the input stream, to then write it to the output stream.

```java
package com.manzanit0;

import java.io.*;
import java.net.ServerSocket;

public class EchoServer {
    public static void main(String[] args) {
        int portNumber = 4098;

        try (
                var serverSocket = new ServerSocket(portNumber);
                var clientSocket = serverSocket.accept();
                var outputStream = clientSocket.getOutputStream();
                var inputStream = clientSocket.getInputStream()
        ) {
            while(true) {
                var bytes = readAllBytes(inputStream);
                outputStream.write(bytes);
            }

        } catch (IOException e) {
            System.out.println(e.getMessage());
        }
    }

    private static byte[] readAllBytes(InputStream stream) throws IOException {
        StringBuilder data = new StringBuilder();

        // available only returns a value after reading at least 1 character -> do/while.
        do {
            data.append((char) stream.read());
        } while (stream.available() > 0);

        return data.toString().getBytes();
    }
}
```

The other thing I want to mention is that since streams work with undefined amounts of data, potentially infinite,
if you tried executing the program, you will have noticed that it doesn't stop running until you feed it an abort
command (Ctrl/Cmd + C). We can continue inputing data for as long as we wont and the stream will continue feeding it.
In this particular example I have elaborated a specific `bytes[] readAllBytes(InputStream stream)` function which reads
only the available bytes and returns them, but the `InputStream` class provides us with a `readAllBytes()` method which
blocks until the stream is closed, then returning all the bytes received.

You might be wondering though, *what if I want to read the data of my stream a second time? Is it possible?*. Indeed it's
possible, but for understanding how, we must introduce one last concept: **seeking**. 

## Seeking a stream – understanding the side-effects of reading

If you've tried reading a stream a second time, you might have found yourself not being able to read previous data, but
just reading new data. Some streams don't support seeking, but assuming they do, the reason behind this is that streams have
a cursor which points to the last byte read. Every time we read a new byte, that cursor is advanced to the new position.
In order to read already processed bytes we would need to rewind that cursor all the way to the beggining. This is called
seeking.

In Java, the way to do this is using the `mark(int)` and `reset()` method of the
[`InputStream`](https://docs.oracle.com/javase/6/docs/api/java/io/InputStream.html) class, in the C# example,
we would simply set [`file.InputStream.Position = 0`](https://stackoverflow.com/questions/4266448/reading-stream-twice).
These are the side effects of reading a stream. If a stream doesn't support seeking, another solution would be copying
our read bytes to another array and maintain a copy. Nonetheless, take into account that sometimes one of the purposes of
using streams is to go easy on memory consumption, and we're copying all the read data in a memory array, then we're
anulling this completely.

## Wrapping up

We've covered a lot of things, but if we were to do a really quick TLDR and summarize some of the key points,
I would just go with:

1. Streams manage undefined amounts of data – sometimes infinite.
2. Streams enable a huge performance boost for applications since they don't have to load all the data in memory
before processing it.
3. Just as streams have to be opened, they must be closed, otherwise the OS' resources aren't freed and that can
potentially be a very expensive cost, be it sockets or file handles.
4. Some languages provide constructs to handle the disposal of resources, like `using` in C# or `try` in Java.
5. Streams have a cursor which points to the last read byte. In some cases, we can rewind that cursor in order to read
the same data a second time. This is called seeking.

Following up, in case you want to delve a little more into sockets, as a concept, feel free to check out this other
post I have about them: [URL](https://manzanit0.github.io/networks/2018/11/09/sockets-explained.html).