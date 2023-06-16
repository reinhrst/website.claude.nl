---

title: "Rust's nom as a streaming parser"
tags:
  - deepdive
  - tech
  - rust

header:
  image: header.jpg

toc:
  enable: true
---

For a new project I'm working on, I need to convert an MTS file into an MP4 file, which is as simple as parsing one format and writing out the other. Since the files can be rather larger (and I would like this to work in the browser some day as well), I was looking at a streaming parser to achieve this.

<!--begin-summary-->
The go-to parser in Rust seems to be [nom][1][^go-to-parser], and it has [streaming built-in][2].
However the good news stops there, since it's not completely clear what this does and how it's supposed to work. There is an (unanswered) [github issue][3] asking for a small example, [another issue][4] explaining that there are some corner cases that cannot be solved with the streaming parser, a [stackoverflow question][5] on the subject.

It took me some time before I understood how to think about streaming parsing, so I decided to write down my thoughts, both for my own future reference, as others.

[1]: https://docs.rs/nom/latest/nom/
[2]: https://docs.rs/nom/latest/nom/#streaming--complete
[3]: https://github.com/rust-bakery/nom/issues/1160
[4]: https://github.com/rust-bakery/nom/issues/1582
[5]: https://stackoverflow.com/questions/46876879/how-do-i-create-a-streaming-parser-in-nom
<!--end-summary-->

{{< note >}}
  The work describe in this post started as reply to [a GitHub issue on the `nom` repo][3] asking for _A sample/PoC parser_; since I was interested in learning how this would work (and wanted to earn some karma-points), I decided to make a quick 10-line sample and share it as a reply in this issue.

  Before long I decided to switch tracks, and maybe reply with a longer answer and a link to a gist for the code.

  The more I dove into the question however, the more I understood that the issue had lots of sides, and any quick reply was just going to leave more questions than answers.

  So therefore I made this deep-dive post. It will show you a POC version of a parser using the `nom::*::streaming` methodology, however also explain why it works, my thoughts about it, and maybe some alternatives.

  The examples in this post are for byte-based parsers (that take `&[u8]` rather than `&str`).
  Any concepts should also work for `&str` based parsers, however you need to take some extra care that when you read a file chunk-wise, your chunk may not be valid UTF-8 (i.e. it cut halfway a multi-byte character).

[3]: https://github.com/rust-bakery/nom/issues/1160

{{< /note >}}

{{< link-button href="https://github.com/reinhrst/nom-streaming-poc" type="success" >}}
The full code of the examples in this post is available on {{< icon github-original >}} GitHub
{{< /link-button >}}

## Why would we want to stream

Before looking into _how_ to stream, let's see _why_ one would like to stream.

As far as I can see, there are two reasons for a streaming parser: data size and data availability.

### Data size

In a non-streaming parser, all data (let's say the whole file) is read into memory, and then parsed into some output format.
In the best case, this needs as much memory as the file size, worst case much more (e.g. for the output structure).
A streaming parser could avoid using all this memory, which allows creating programs that deal with files larger than system memory, or that are just good citizens (by not demanding too much memory).

Typically the output will need to be "consumed" before more output is generated, e.g.:

- Output is processed and written to disk in chunks, or streamed over the network and then dropped (either implicitly or explicitly)
- Output is reduced (e.g. summed)
- Output is heavily filtered upon (e.g. searching for 10 records in a stack of 100M)

In these cases, if the input files are (or have a possibility to become) large, it's useful to use a streaming parser.

### Data availability

The second case when a streaming parser is useful, is if you want to start parsing data before everything is available.
For instance, you're building an HTTP server, and you want to start parsing HTTP headers (and possibly closing the connection if you see something you don't like) before the whole request is in.

This may even be the case for local/network files; if you have many files of 10MB each, and you quickly want to parse the start of the file to look for something, it may be better to use a streaming parser (and discard files quickly), rather than parse the whole file each time.


### Other uses

There are other scenarios when streaming parsers could be useful, however I feel that they are more exotic, and could (with some creativity) be mapped to either or both of the above use-cases.


In this post I will mostly use the data-size reason in the examples, however the examples (with the exception of Memory Mapping) should work just as well for the data-availability reason.

## What is a streaming parser (please don't skip without reading the note)

{{< note type=warning >}}
The idea of a streaming parser seemed simple to me, until I started thinking about all the little details.
Unless you have experience with _streaming_ _parsers_ (emphasis on both words), I would certainly advice you to at least glance at this section.
{{< /note >}}

The idea of a streaming parser seems simple, but it's worth thinking about what it actually means.

Let's start with a standard parser, parsing some account book.
The parser has an input of `&str` and as output a `Vec::<i64>`.

```goat
+----------------+                                 +---------------+
|                |                                 |  vec![        |
|   € 1234       |                                 |    1234,      |
|   € 54354      |            +--------+           |    54354,     |
|   € 765 -      +----------->| parser +---------->|    -765,      |
|   € 84 -       |            +--------+           |    -84,       |
|   € 9          |                                 |    9,         |
|                |                                 |  ]            |
+----------------+                                 +---------------+
```

If for some reason (see previous section) we want to start streaming, we have to consider both sides of the parser:

```goat
+----------------+                                 +---------------+
|                |                                 |  vec![        |
|   € 1234       |                                 |    1234,      |
|   € 54354      |            +--------+           |    54354,     |
|   € 765 -      +----------->| parser +---------->|    -765,      |
|   € 84 -       |     ^      +--------+     ^     |    -84,       |
|   € 9          |     |                     |     |    9,         |
|                |     |                     |     |  ]            |
+----------------+  streaming            streaming +---------------+
```

For the input-side, it's easy to imagine what this means (we all have a mental image of streaming a string into some component).
However for the output-side, we cannot stream a `Vec` (a `Vec` is a single object), so we have to split it up some way.

If the thing to parse is a list of repeating items (as it is in our example), it's easy to understand how to split it up.
Rather than returning a `Vec::<i64>` we return an `Iterator {type Item=i64}`, which produces new numbers upon request.

However consider a more complex example:

```ascii
* Alice
  * current account
    € 1344
    € 54-
    ...
  * savings account
    € 76
    € 98
    ...
* Bob
  ...
```

In this case, we could make an `Iterator {type Item=User}`, so each iteration returns a single user.
This might work in most cases, until you encounter a file where a single user has 100M account lines.
In this case it might be better to have something like an `Iterator {type Item=UserAccountLine}`.
What you need exactly will depend on the assumptions you (can) make about your input.

{{< note >}}
There are also streaming parsers that return each small element as soon as it has been parsed; a good example of this is [`ijson`][7], a streaming JSON parser for Python.

If you parse something like `{"a": [1, 2, 3], "b": true}`, you will receive something like `dictStart, key(string("a")), listStart, number(1), number(2), number(3), listEnd, key(string("b")), bool(true), dictEnd`, each element as soon as the parser reaches it.

If this is the kind of parser you're trying to make, then this post may serve as a start, but much more work is needed.

The parsers in this post are more of the type that if you already know it's going to be a mapping on top-level, they stream 2 tokens: `("a", [1, 2, 3])` and `("b", true)`.

[7]: https://pypi.org/project/ijson/#lower-level-interfaces
{{< /note >}}

The important take-away from this section is that there are two things you have to fix in order to make a streaming parser: make sure you can _stream in_ the `&[u8]` (or `&str`), and decide what the thing is you want to _stream out_.

{{< note >}}
In this post I'm avoiding an additional problem of lifetime.
In a fully-optimised system, you might want to have the output-objects from your parser point to the input-stream memory (e.g. some parser parsing a logfile into lines, may want to return slices of the original input `&str`.
This is what parsers advertise as zero-copy.

For non-streaming parsers this works well, since you own the input-data at the place where you created the parser (or higher).
Any returned slices therefore have the same lifetime as the input data.

This doesn't work in (most) streaming parsers.
The parser itself will own the input-data, read more when needed, and discard old data after parsing.
If we also want the data available for the returned objects, we will have to do something with reference counting, or some other tricks.

In this post I chose the easiest solution, of just returning copied data.
Copying this data once should not take much time, however if you want to fix this, I leave this as an exercise to the reader ;).

{{< /note >}}


## The `nom` way to do streaming parsing

### How it works

The `nom` way to build a streaming parser is using `nom::bytes::streaming`, rather than `nom::bytes::complete` (or `nom::character::streaming` rather than `nom::character::complete`).
Note that other parts in `nom` (like `nom::branch`) work in both streaming and complete modes.

The idea of the `nom::*::streaming` parsers is that they will let you know when you run out of data by returning an `Err::Incomplete()`.
As soon as you (as programmer) receive this error, you know that you should try again with more data (read in more data from the file, append that to the current unparsed buffer, and call the parse function again).

Functions in other parsing modules such as `nom::branch::*` will just pass on the `Err::Incomplete()` if found, so that if one of the branches needs more info, all branches will have to be evaluated again.

It makes most sense (as in how much work it takes) to put the check for `Err::Incomplete` at the top-level (if, in the example above with the account-books, you have an iterator that returns `User`s, then put the "if-err-incomplete-then-refill-input-and-try-again" at the place where you call `parse_user`.
If parsing the user fails based on the available data, get more data and call `parse_user` again.

Note that you don't have to worry that the parser already "consumed" the data on the first parse.
If you look at your `parse_user()` function (as well as all the `nom::bytes::*::*` functions), they take a `&[u8]` as input.
This is an immutable borrow[^immutable-borrow], so (per definition / Rust compiler security rules) the parser _cannot_ change (consume or otherwise) anything here.

#### So how does it look in practice

When you call `take(5)` on a slice of length 4 in one of the `nom::*::complete` parsers, it will return a `Err(Err::Error(Error::new(_, ErrorKind::Eof)))`, signaling that it cannot parse because the input-data ran out.
The `nom::*::streaming` parsers will rather return a `Err(Err::Incomplete())`, telling you that more data is needed.

There are larger differences however in variable-length parsers (parser like `take_till()`, but also `nom::branch::alt(take(6), take(4))`, or `nom::sequence::delimited()`).
If you run `nom::bytes::complete::take_till((|b| b == 0))` on a sequence `0x01 0x02 0x03`, it will return the whole sequence as a parse result.
However `nom::bytes::streaming::take_till((|b| b == 0))` will return an `Err::Incomplete` error, which makes sense, because maybe there are more non-NULL-bytes available.

#### The issue with the `nom::*::streaming` functions

There is a big problem with the `nom::*::streaming` functions (also reported [in this GitHub issue][4]).
As soon as all data is read, you expect the `nom::*::streaming` parser to work like a `nom::*::complete` parser; after all, there is no more data to stream.
There is, however, no way to signal the streaming parser that you reached the end of the stream. 

This would be a problem in our example above, where we want to parse `UsersAccount`s: after the last line is read, it's unhelpful for the streaming parser to keep asking for more data, and it should just parse the final user.

There are workarounds for this issue (i.e. manually inserting stuff at the end of your stream that makes your last parsed object end, if possible, or duplicating your whole `parse_user()` function to use `nom::*::complete` parsers (possibly a macro could do this?) which you use as soon as you reach the EOF).

Either way feels hacky to me and not necessary, and makes this method less than ideal.
In the second part of this post, I also describe some other ways how one might be able to achieve the advantages of streaming parsing, without this limitation.


### Example

Let's take a look at a working proof-of-concept of a parser using `nom::bytes::streaming`.

Be aware though that building a streaming parser this way in `nom` actually requires a lot of scaffolding!

{{< note type=warning >}}
I am a very experienced programmer in many languages, however Rust is not one of them.
This might mean that (although all code shown here works, and does not use the `unsafe` keyword nor the `'static` scope), some parts of it might not be Rustic.
I would be very happy to get feedback from experienced Rustaceans on the code!
{{< /note >}}


#### The simple parts

For this example I use a trivial parser, that returns bytes delimited by `NULL` bytes (writing the parser is not the hard part here, it's the same as a `nom::*::complete` parser).

{{< figure-with-caption caption="parse until the next `NULL` byte, then consume at most one `NULL` byte. Finally, return a `Vec` with the retrieved bytes." >}}
```rust
fn parse_until_null_byte(unparsed_data: &[u8]) -> IResult<&[u8], Vec<u8>> {
    let (unparsed_data, parse_result) = bytes::streaming::take_till::<_, _, error::Error<_>>(|b| b == 0x00)(unparsed_data)?;
    // make sure we always process _something_
    let minimal_null_bytes = if parse_result.len() == 0 { 1 } else { 0 };
    let (unparsed_data, _) = bytes::streaming::take_while_m_n(
        minimal_null_bytes, 1, |b| b == 0x00)(unparsed_data)?;
    Ok((unparsed_data, parse_result.to_vec()))
}
```
{{< /figure-with-caption >}}


In order to demonstrate how it works, we create a file which contains a byte structure like (newlines for illustration only):
```ascii
0x00
0x01 0x00
0x01 0x02 0x00
0x00
0x01 0x02 0x03 0x00
0x01 0x02 0x03 0x04 0x00
0x00
...
```

File is created using the following code:

{{< figure-with-caption caption="Create a (temp) file, fill with test data, and then rewind." >}}
```rust
pub fn create_rewound_file(iterations: usize) -> Result<File, Box<dyn Error>> {
    let mut file = tempfile::tempfile()?;
    let mut single_iteration_data: Vec<u8> = vec![];

    for i in 1u8..0x10 {
        let mut range: Vec<u8> = (0u8..=i).collect();
        single_iteration_data.append(&mut range);
        if i % 2 == 0 {
            // extra 0 byte every two lines
            single_iteration_data.push(0);
        }
    }
    for _ in 0..iterations {
        file.write(&single_iteration_data)?;
    }
    file.rewind()?;
    return Ok(file);
}
}
```
{{< /figure-with-caption >}}

Finally, what we want to achieve is to iterate over the returned vectors, as this:

{{< figure-with-caption caption="The main function: create the file, feed to parser and loop over result." >}}
```rust
fn main() -> Result<(), Box<dyn Error>> {
    let file = create_rewound_file(1)?;
    for bs in NullDelimitedVectorParser::new(Box::new(FileIterator { file })) {
        println!("Found {:x?}", bs)
    }
    Ok(())
}
```
{{< /figure-with-caption >}}

and get the following output:

```ascii
More data needed
Found []
Found [1]
Found [1, 2]
Found []
More data needed
Found [1, 2, 3]
Found [1, 2, 3, 4]
More data needed
Found []
Found [1, 2, 3, 4, 5]
More data needed
Found [1, 2, 3, 4, 5, 6]
Found []
...
```

(the `More data needed` strings here are showing where more of the input-file is streamed in.)

The two pieces missing in the puzzle now are the are the `FileIterator`, which streams in the input, and the `NullDelimitedVectorParser`, which streams out the output.
The input is relatively easy, make the file into something that can provide data-chunks as in iterator:

{{< figure-with-caption caption="An `Iterator {type = Vec<u8>}` to read files chunk-by-chunk." >}}
```rust
const CHUNK_SIZE: usize = 8; // ridiculously small chunk size for example purposes
struct FileIterator {
    file: File
}

impl Iterator for FileIterator {
    type Item = Vec<u8>;
    
    fn next(&mut self) -> Option<Self::Item> {
        let mut buffer: Vec<u8> = vec![0u8; CHUNK_SIZE];
        let len = self.file.read(&mut buffer).expect("Cannot read file");
        if len == 0 {
            None
        } else {
            buffer.truncate(len);
            Some(buffer)
        }
    }
}
```
{{< /figure-with-caption >}}

#### The output iterator

The work-horse of the whole system is the output-iterator:

{{< figure-with-caption caption="The `NullDelimitedVectorParser` parses vectors of `NULL` delimited bytes out of an iterator input." >}}
```rust
pub struct NullDelimitedVectorParser {
    input_iterator: Box<dyn Iterator<Item = Vec<u8>>>,
    parsing_data: Vec<u8>, // store the current data-chunk here
    unparsed_data_offset: usize,
}

impl NullDelimitedVectorParser {
    pub fn new(input_iterator: Box<dyn Iterator<Item = Vec<u8>>>) -> NullDelimitedVectorParser {
        return Self {
            input_iterator,
            parsing_data: vec![],
            unparsed_data_offset: 0,
        };
    }

    pub fn get_slice(&self) -> &[u8] {
        &self.parsing_data[self.unparsed_data_offset..]
    }

    pub fn get_slice_offset(&self, slice: &[u8]) -> usize {
        let data_begin = self.parsing_data.as_ptr() as usize;
        let data_end = data_begin + self.parsing_data.len();
        let slice_begin = slice.as_ptr() as usize;
        let slice_end = slice_begin + slice.len();
        let slice_offset = slice_begin - data_begin;
        assert_eq!(data_end, slice_end);
        assert!(slice_offset <= self.parsing_data.len());
        slice_offset
    }

    fn read_more_data_from_source(&mut self) -> Result<(), Box<dyn Error>> {
        match self.input_iterator.next() {
            Some(new_data) => {
                self.parsing_data = [self.get_slice(), &new_data].concat().to_vec();
                self.unparsed_data_offset = 0;
                Ok(())
            }
            None => Err("EOF")?, // string error OK for POC but should be proper custom error in production
        }
    }
}

impl Iterator for NullDelimitedVectorParser {
    type Item = Vec<u8>;

    fn next(&mut self) -> Option<Self::Item> {
        loop {
            match parse_until_null_byte(self.get_slice()) {
                Ok((new_unparsed_data, return_value)) => {
                    self.unparsed_data_offset = self.get_slice_offset(new_unparsed_data);
                    return Some(return_value.to_vec());
                }
                Err(Err::Incomplete(_)) => {
                    println!("More data needed");
                    match self.read_more_data_from_source() {
                        Ok(_) => continue,
                        Err(_) => {
                            if self.get_slice().len() == 0 {
                                println!("Done");
                            } else {
                                println!("There are {} bytes remaining", self.get_slice().len());
                            }
                            return None
                        }
                    }
                }
                Err(e) => {
                    panic!("Parse error: {}", e);
                }
            };
        }
    }
}

```
{{< /figure-with-caption >}}

The `NullDelimitedVectorParser` takes an input iterator, and has two extra fields: a `parsing_data: Vec<u8>` to store the part of the input-data currently in memory, and an `unparsed_data_offset: usize` pointer that keeps track of until where the `parsing_data` has been parsed. It implements an `Iterator {type=Vec<u8>}`, where the parsing results are returned.

Since `nom` works with `&[u8]` slices (and not vectors) we need some way to convert back and forth between our `parsing_data + unparsed_data_offset` and `&[u8]`.
`get_slice()` gets us a slice from the struct fields, whereas `get_slice_offset` gives the correct offset, given a slice gotten back from the `nom` parsing functions.
(there may be a more elegant solution with storing the slices directly, rather than going back-and-forth through the offset; if so I would love to hear it :))

When running out of data, `read_more_data_from_source` is called. It asks the input iterator for more data, constructs a new `parsing_data` of the unused part of the previous `parsing_data` and the new data gotten from the iterator. When no more data is available on the iterator an End-Of-File error is returned.

Finally the iterator implementation. When asked for the next item, we enter a loop.
We try to `parse_until_null_byte()` , which either succeeds (in which case we store the new `unparsed_data_offset` and return the result), or an `Err::Incomplete()` is received.
In the latter case, we load more data and try again.
If we run out of data, we either are done parsing (`slice.len() == 0`), or there is leftover data (which is what happens if the stream doesn't end in a NULL byte; this illustrates [the problem described above](#the-issue-with-the-nomstreaming-functions).

You can see this happening in the last line of the full output: `There are 15 bytes remaining`.
The file ends in `0x01 0x02 0x03 ... 0x0e 0x0f`, and this can not be parsed by the streaming parser (even though a complete parser _would_ parse it, since we made a trailing `NULL` byte optional).


## Alternatives

As promised, I will look at some alternative ways to achieve the same result, with other trade-offs.
These alternatives all use a much simpler parser (non-streaming, using `nom::bytes::complete`) which still outputs streaming data.
Streaming in the the data is done using other methods than `nom::*::streaming`.

{{< figure-with-caption caption="The complete code for the `nom::bytes::complete` version of the streaming parser." >}}
```rust
use nom::{bytes, error, IResult};

fn parse_until_null_byte(unparsed_data: &[u8]) -> IResult<&[u8], Vec<u8>> {
    let (unparsed_data, parse_result) =
        bytes::complete::take_till::<_, _, error::Error<_>>(|b| b == 0x00)(unparsed_data)?;
    // make sure we always process _something_
    let minimal_null_bytes = if parse_result.len() == 0 { 1 } else { 0 };
    let (unparsed_data, _) = bytes::complete::take_while_m_n(
        minimal_null_bytes, 1, |b| b == 0x00)(unparsed_data)?;
    Ok((unparsed_data, parse_result.to_vec()))
}

pub struct NullDelimitedVectorParser<'a> {
    data: &'a [u8],
}

impl<'a> NullDelimitedVectorParser<'a> {
    pub fn new(data: &'a [u8]) -> Self {
        Self {
            data,
        }
    }
}

impl<'a> Iterator for NullDelimitedVectorParser<'a> {
    type Item = Vec<u8>;

    fn next(&mut self) -> Option<Self::Item> {
        if self.data.len() == 0 {
            return None;
        }
        let (new_unparsed_data, return_value) = parse_until_null_byte(self.data).expect("Parse error");
        self.data = new_unparsed_data;
        return Some(return_value.to_vec());
    }
}
```
{{< /figure-with-caption >}}

The code is much simpler than the `nom::*::streaming` version.
The `parse_until_null_byte()` function is the same (except that it uses the `nom::bytes::complete` version of the parser).

Our `NullDelimitedVectorParser` expects a `&[u8]` with all data (the data-field will update to contain the unparsed data slice only).

The iterator first checks if parsing is done (and if so, completes).

If not, it calls the parser, storing the `new_unparsed_data` in `self.data`, and returning the `return_value` -- as simple as can be!

### Memory mapped file

By far the easiest way (from the point of view of the programmer) is a [Memory Mapped File][8], using [the `memmap2` crate][12] (there is also the (unmaintained) [`memmap` crate][9]; either seems to work with similar results).
The idea behind this is to leave all complexity of getting to the data to the parser to the OS.
The parser just sees one big `&[u8]`, however when a certain range of data is needed, the OS quickly reads it from disk (and discards it when it's not needed anymore).

#### Downsides

Before going on, there are certainly downsides to this approach.
This is not an exhaustive list; I just want to show that this is not _One Solution To Rule Them All_.

##### "unsafe"

First of all, the code depends on the `unsafe` keyword; memory maps are inherently unsafe (or at least, what Rust calls unsafe).

There is a [long discussion on users.rust-lang.org][10] talking about what this means exactly.

As far as I can tell, the `unsafe` part comes from the fact that the underlying mapped file can change, whereas Rust expects underlying memory not to change while borrowed.
This could lead to [Undefined Behaviour][13] (which is a Very Bad Thing).
What this means for you (and whether you need to care about it) is something for you to decide.

{{< note type=danger >}}
I expect that anyone who made it this far into this post, will not need this warning, but still.
There is a reason that `unsafe` is called "unsafe".
Don't use it unless you really understand its consequences and decide you're OK with them!
{{< /note >}}

##### Bad experiences

In addition, please read [the experiences of the Sublime team][11] when they added `mmap` to their code.
They were less than happy with the result, and in the end removed the code again.

Whether this is an issue for you, you have to decide for yourself, but it's certainly a cautionary tale.

##### Availability

Memory Mapping files (obviously) only works for files. In addition, I'm not sure that this technique is available on all platforms (for instance, would it work in WebAssembly?).


#### Implementation

The code to run this is quite straight-forward.

{{< figure-with-caption caption="Code to use the complete parser with a memory mapped file" >}}
```rust
fn do_nom_memmap() -> Result<(), Box<dyn Error>> {
    println!("======== NOM MEMMAP ======");
    const ITERATIONS: usize = 200_000;
    let file = filecreator::create_rewound_file(ITERATIONS)?;
    let data = unsafe { MmapOptions::new().map(&file)? };
    println!(
        "Loaded all data ({} MB) into vitual memory",
        data.len() / 1024 / 1024
    );
    let parser = nomcomplete::NullDelimitedVectorParser::new(&data);
    for (i, bs) in parser.enumerate() {
        if i % ITERATIONS == 0 {
            println!("Found {:x?} ({})", bs, i)
        }
    }
    Ok(())
}
```
{{< /figure-with-caption >}}

In order to test the memory mapping capabilities, we make a 200k iterations of the parsing file (it will be about 270MB of data).
We then use `unsafe` to get a memory mapped file, which we can hand to our parser without a problem.
We only print one in 200k result rows, to keep the output in check.

Running this (on my M2 MacBook) with `/usr/bin/time -l cargo run`, results in the following output:
```ascii
======== NOM MEMMAP ======
Loaded all data (27 MB) into vitual memory
Found [] (0)
Found [1, 2, 3, 4, 5, 6, 7, 8, 9, a, b, c, d, e] (200000)
Found [] (400000)
  ...
Found [1, 2, 3] (4000000)
Found [1, 2] (4200000)
Found [1, 2, 3, 4, 5, 6, 7, 8, 9, a, b, c, d, e, f] (4400000)
        2.53 real         1.86 user         0.29 sys
            30015488  maximum resident set size
                  ...
             1229568  peak memory footprint
```

It shows that a 27MB large file was created, and then parsed. It took a bit over 2 seconds, and although maximum RSS was ~30MB, the peak memory footprint was just over 1MB.
It's hard to find exactly what these numbers mean, but (afaik) peak memory footprint is the important one.
(I can also confirm that this way I can parse a 160GB file on my (32GB RAM) MacBook. Trying to load that file into memory fails after 50GB).

{{< figure-with-caption caption="`/usr/bin/time -l` info for a run parsing a 160GB file (actually 40 video files concatenated) using the Memory Mapping method. It did use 86MB of memory in the end (removed all lines that were 0, and added `'`s to numbers for readability)." >}}

```ascii
     2150.77 real      1828.52 user        68.13 sys
          19'379'601'408  maximum resident set size
                 695'238  page reclaims
               9'396'579  page faults
                   3'986  voluntary context switches
               5'268'657  involuntary context switches
      26'591'696'769'545  instructions retired
       6'353'835'034'072  cycles elapsed
              86'103'040  peak memory footprint
```
{{< /figure-with-caption >}}


#### Conclusion
This is a quick and (depending on your definition) either dirty or acceptable way to get a streaming parser that doesn't suffer from the [cannot end problem](#the-issue-with-the-nomstreaming-functions).

It only works for files, and you will have to check if the method is available on your target platform.
It also suffers from a number of issues, chief among them how to make sure that the underlying file doesn't change while being accessed.

### Custom input type

So far we've used the parser as getting `&[u8]` as input, however `nom` [supports custom input types][14].
This system allows one to throw in other kinds of data; so (let's be creative) say that you have a `Vec<Keystrokes>`, you could use `nom` to parse for [sequences that would do special moves in Mortal Kombat II][15] (not sure that `nom` would be the best tool here, just making an example).

In theory this should allow us to make an input type that loads data only when it's requested.

In practice however, this seems to become an extremely convoluted and hacky solution, for a couple of reasons:

1. The `nom::bytes::complete::*` parsers, all return a `IResult<Input, Input, Error>`, meaning that the thing you parse from, and the parsing result (the first and second parameter to `IResult`), are the same type.
This works when both input and output to the parser are `&str` or `&[u8]`, but if you want the input to be some `ByteStreamer` object, but the output still be `&[u8]` (or maybe `Vec<u8>`), it's harder (not impossible, but takes more effort).
2. The thing you parse from (`&[u8]` normally) gets copied around a lot (every time you parse 2 bytes from a slice, you get a new slice that is 2 bytes shorter).
This works well for `&[u8]` and `&str` because they are very cheap to make a new slice from (and immutable).
If you use a more complex object here, you might lose a lot of performance.
3. Normally parsing works like this: you have a bunch of data, and throw this into the parser. Your calling function owns all the data, and nobody has to worry about lifetimes, etc.
With the envisioned streaming parser, you lose this advantage; now the Custom Input Object owns the data, and may evict it (e.g. because it was parsed and not needed in the input object anymore). This can probably be solved using reference counting; it just doesn't make things faster or less complex.
4. There is no nice way to deal with I/O-errors. If halfway a parse, you need to load more data, and somehow the load fails, there is little you can do except `panic!`.
5. I'm not sure you can actually implement a custom type that _properly_ implements the traits described on [the custom input type page][14].
What you _can_ (probably) do is implement something that will work for (a subset of) `nom`-based parsers, however maybe a next version of `nom`, or third party code, will assume that the functions required in a trait actually work (rather than panic).

#### Conclusion

I do see that it should be possible to make something like this, but I don't see it be worth the effort on the current version of `nom`[^nom-version].
It's possible that with some improvements on the `nom` side, things could start to look more promising here.
It's also possible that one day someone will implement such a custom input type (with a lot of effort) and it can be easily plugged-in.

For now, I think that this approach is giving more problems than it's worth.

### Fake it till you make it

Another solution for your problems may be to not do streaming parsing at all, however whether this is an option at all, depends on the problem you're trying to solve.

Often you want to "get out" from your parser certain "blocks"; this could be lines in a logfile, frames in a video, or (to get back to our example of users and accounts) `UserAccount`s.
These blocks will often be easily identifiable in some other way, either they are of fixed length (packets in an MTS media stream), will be delimited (eg by newline) or may be prefixed with a length (H264 frames in AAVC encoding).
For instance, in the case of our `UserAccount`s, a user ends when there is a newline followed by a non-whitespace character.

In this case, it will be easy to just load enough of your file (or data from the stream) until you know you have enough for the thing you're trying to parse.
Then parse that thing and return it on the iterator.
It will not suffer from the finishing-issue with `nom::*::streaming`, and the code will be much easier than the official streaming solution.

However at the same time, it does really depend on whether you have an easy way to detect when the block you're interested in ends (e.g. it will be much harder to something like: detect a newline, _iff_ the newline is not between `""` (or something like that)).
You have to consider if the complexity (and future maintainability) of finding the block is not larger than the complexity of one of the other solutions.

## TL;DR -- just give me the conclusion

{{< note type=update >}}
Just as I was about to put this post live, I found [`winnow`][16]. 
`winnow` is a fork of `nom`, with (in their own words) different priorities:

> 1. [`nom` has] lower churn for existing users while `winnow` is trying to find ways to make things better for the parsers yet to be written.
> 2. [`nom` has] a small core, relying on external crates like `nom-locate` and `nom-supreme,` encouraging flexibility among users and to not block users on new features being merged while `winnow` aims to include all the fundamentals for parsing to ensure the experience is cohesive and high quality.

I will certainly need to take a look at `winnow` in the near future, it might be a winner.

[16]: https://docs.rs/winnow/latest/winnow/index.html
{{< /note >}}

Making a streaming parser (using `nom`, or probably using any other method) is more complex than I thought before starting this.
Where you can have a non-streaming parser with `nom` in minutes, to get a streaming parser you have to start by actually understanding what it is that you're trying to do, and what exactly you need to stream.
Then you need to do quite some work to make it happen.

This post describes some ways how to do it (with example code). Each method has different advantages and disadvantages, and there is no clear winner.
The best choice depends on your exact needs.

Personally, I would use the "Fake it till you make it" method if it would work (easily) for my specific problem.
The reason here is that the code needed will be very straight-forward, and will be easily understandable by others (including "Me In 2 Years").

If that doesn't work, I would consider the Memory Map method, however only in small, self-contained projects on the desktop.
The [Sublime article][11] convinced me to at least think very hard before using this in larger projects, and I would have to test if it would work in something like WebAssembly (I expect not).

The `nom::*::streaming` method will work well in all cases, **if** you find a solution for parsing the last item (if this is a problem in your situation).


[^go-to-parser]: Before I started this project, I spent about 20 minutes Googling for Rust parsers, and then came to the conclusion that `nom` was (at least one of) the most popular.
However while writing this post, I ran into many others, and (as I mention at the bottom of this post), I will certainly want to try `winnow` soon to see how it holds up.

I may very well have missed some other very good parsers during my search, so I will always be happy to hear about alternatives!


[^immutable-borrow]: I have to say, I don't really know if this is the correct terminology.
The parsing functions can also take other objects (see [this section](#custom-input-type)), which (when passed in) will be owned by the parsing function (and therefore cannot be used subsequently in case of `Err::Incomplete`.
Regardless, in the case of `&[u8]`, it's clear that the parsing function cannot change (or consume) the data.

[^nom-version]: All my work was done on `nom-7.1.3`.


[4]: https://github.com/rust-bakery/nom/issues/1582
[6]: https://www.agama.tv/demystifying-the-mp4-container-format/#:~:text=What%20is%20MP4%3F,used%20in%20streaming%20video%20services.
[8]: https://en.wikipedia.org/wiki/Memory-mapped_file
[9]: https://crates.io/crates/memmap
[10]: https://users.rust-lang.org/t/how-unsafe-is-mmap/19635
[11]: https://www.sublimetext.com/blog/articles/use-mmap-with-care
[12]: https://crates.io/crates/memmap2
[13]: https://en.wikipedia.org/wiki/Undefined_behavior
[14]: https://github.com/rust-bakery/nom/blob/main/doc/custom_input_types.md
[15]: https://www.mksecrets.net/mk2/eng/mk2-specialmoves.php
