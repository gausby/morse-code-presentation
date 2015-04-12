[WIP] How to write a Morse code encoder and decoder in Elixir
=============================================================

This article aim to show the reader how one could write a program in the Elixir programming language that convert a text string to a morse encoded string, and the other way around.

While this article is aimed at newcomers it does not aim to be a complete guide to programming Elixir, but it should show some cool stuff, and hopefully show how one could attack and solve a task using Elixir--at the end we should have a fast and working morse code encode/decoder to boot!

I (Martin Gausby) will attempt doing a morse encode/decoder at the [Copenhagen Elixir](https://github.com/cphex/cphex/) April 2015 meet up. This text should do instead of handing out slides after the talk/live coding session.

*The state of this document is the very definition of work in progress. Please ask me questions and/or ask me to clearify if something is unclear, and feel free to make pull requests if you find spelling mistakes and/or correct the language.*


Generate a new project
----------------------

Elixir ships with a command-line tool called *mix*. It can be used to create new projects, run unit tests, install third party modules as dependencies using hex. For now, let's use it to create a new project:

```sh
$ mix new morse
```

This should create new directory with a bunch of files. Navigate into this directory and run the generated unit test suite by typing `mix test`. We should have one passing test.

Open the file *test/morse_test.exs* in an editor and remove the test called *"the truth"*.


Encoding Morse Code
-------------------
The easiest thing at this point would be to encode morse from text. We will handle decoding later. Create a test that test morse encoding. Add the following to the test file:

```elixir
    # test/morse_test.exs
    test "morse encoding" do
      assert Morse.encode("") == ""
    end
```

This will of course fail because we have yet to define the encode function in the Morse-module. Let us do that. Open the file *lib/morse.ex* and define the function:

```elixir
    # lib/morse.ex
    defmodule Morse do
      def encode(""), do: ""
    end
```

While this makes our unit tests pass it is a horrible morse code encoder. Thanks to pattern matching it only accept empty strings, so given any other input it will fail with an error saying that *no function clause matching in Morse.encode/1*.

Let's add two simple cases that will test for S and O, which in morse respectively is three dots (S = ...) and three dashes (O = ---).

```elixir
    # test/morse_test.exs (in the "morse encode" test block)
    assert Morse.encode("S") == "..."
	assert Morse.encode("O") == "---"
```

The implementation is as simple as the empty string case. Here's the entire file so far:

```elixir
    # lib/morse.ex
    defmodule Morse do
      def encode(""), do: ""
      def encode("S"), do: "..."
      def encode("O"), do: "---"
    end
```

Using pattern matching we could build the entire morse alphabet. One problem with our implementation so far is the fact that it can only encode single letters. Next up, let's handle the case where we want to send the message "SOS".


SOS. Handling entire words, recursion to the rescue!
----------------------------------------------------
Add a test for a word in the encode-test-block.

Morse letters are seperated by spaces, so when we get `"SOS"` as input we should return `"... --- ..."`.

```elixir
    # test/morse_test.exs (still in the "morse encode" test block)
    assert Morse.encode("SOS") == "... --- ..."
```

So far we might think that we would just create a function that return `"... --- ..."` when receiving the message `"SOS"`, but that would be too easy, and we want to be able to communicate anything using morse, so this solution would not scale.

We need to encode the string letter by letter, so a solution is to recursively traverse the string, encode each letter and return the result of this operation when we hit the empty string. For this we need to do some pattern matching on the input that are a bit more advanced than we have done so far.

In Elixir a string is a list of byte-codes representing the letter. This allow the pattern matching engine in Elixir to match on parts of the string and put these parts into variables that we can use in our function. Try opening an *iex* session and play with it:

```iex
iex(1)> <<"f", rest::binary>> = "foo"
"foo"
iex(2)> rest
"oo"
iex(3)>
```

As shown `<<"f", rest::binary>>` matches the string `foo`, and the resulting variable `rest` will contain *"oo"*. This pattern-matching technique is quite powerful, because we can split strings and store the results in named variables. Let's try to store the first byte in a variable:

```iex
iex(3)> <<first::binary-size(1), rest::binary>> = "foo"
"foo"
iex(4)> first
"f"
iex(5)> rest
"oo"
```

As you might have guessed; the stuff before `::` are the resulting variable name, the stuff after are the resulting data type; you are also able to match on integers, floats, bits, bytes, utf8 and so on, but for now let us just worry about binary data. In our example we specify that we are interested in one byte of binary data. The `rest` variable does not contain a length specifyer, so it will contain the rest of the string.

Knowing this, chipping off the first byte and encoding it should be rather easy, but what do we do then? We have to store the result of the morse encoded character, and then we need to call the encode function with the remainer of the message, and we need to pass the result of the data encoded so far with it. Let us extend the implementation and discuss this:

```elixir
    defmodule Morse do
      def encode(message), do: do_encode(message, "")
		
      defp do_encode(<<>>, result), do: result
      defp do_encode(<<"S", rest::binary>>, result), do: do_encode(rest, result <> "...")
      defp do_encode(<<"O", rest::binary>>, result), do: do_encode(rest, result <> "---")
    end
```

While doing an encode we need to pass in an "accumulator", an place to store the result of our operation as it unfolds and ultimately return it when we have no more bytes to chip off. We want to hide this fact from the user of our module, so we have added a private function called `do_encode` that our `encode` function can see and use to do the encode. In other words, the interface stays the same to the user, but we are free to change the implementation behind the scenes by using this helper function.

And changes are needed. First off; the result of encoded strings are wrong: When writing morse code it is common practice to seperate the letters with a white-space. If we implement a test that assert `Morse.encode("SOS")` to equal `"... --- ..."` we'll get that `"...---..."` does not equal `"... --- ..."`.

Further more, our implementation is highly inefficent, as appending strings the way we currently do is a heavy operation compared to adding an element to a list, and we should never accumulate using strings, so let us change that, and let us solve the white-space problem while we are at it:

```elixir
    defmodule Morse do
      def encode(message), do: do_encode(message, [])
		
      defp do_encode(<<>>, acc),
        do: acc |> Enum.reverse |> Enum.join(" ")

      defp do_encode(<<"S", rest::binary>>, acc), do: do_encode(rest, ["..." | acc])
      defp do_encode(<<"O", rest::binary>>, acc), do: do_encode(rest, ["---" | acc])
    end
```

We changed the accumulator from a string to a list, and we renamed it to `acc`. Notice that we append to the head of a list by `["..." | acc]`, this is because of the way lists are implemented--they are implemented as linked-lists, a list is defined by its head (the first element) and its tail, which in turn is a list with a head and a tail as shown here:

```iex
    iex(1)> [1, 2, 3] == [1 | [2 | [3 | []]]]
    true
	iex(2)> [1] == [1 | []]
	true
```

A list is a recursive data structure and it ends in an empty list.

If we care about the order of the elements we will need to reverse the list before using it. This is because we are adding elements to the front of the list, and thus the last element added to the list will show up first. Reversing an accumulator before we use it is a common Elixir pattern, and we do so in the empty string case of the `do_encode` function. We also join the elements together with a whitespace using `Enum.join/2`

The tests should pass. We have a working morse code encoder that can encode the message "SOS".


The rest of the alphabet
------------------------

Being able to encode the "SOS" message is cool, but our morse code encoder pretty much sucks at anything else. We might need to encode the name of our location if we transmit an "SOS", so let us implement the rest of the morse code alphabet.

Begin by adding a test that test a few more words:

```elixir
    # test/morse_test.exs (still in the "morse encode" test block)
    assert Morse.encode("ALPHA") == ".- .-.. .--. .... .-"
    assert Morse.encode("BETA") == "-... . - .-"
    assert Morse.encode("GAMMA") == "--. .- -- -- .-"
	assert Morse.encode("DELTA") == "-.. . .-.. - .-"
	assert Morse.encode("EPSILON") == ". .--. ... .. .-.. --- -."
```

Looking at our `do_encode` function it might get a bit tedious to copy and paste in a definition for every letter of the alphabet. So let us implement a function that makes a lookup in a Map of the alphabet instead.

```elixir
    defmodule Morse do
      @alphabet %{
        "A" => ".-", "B" => "-...", "C" => "-.-.", "D" => "-..", "E" => ".",
        "F" => "..-.", "G" => "--.", "H" => "....", "I" => "..", "J" => ".---",
        "K" => "-.-", "L" => ".-..", "M" => "--", "N" => "-.", "O" => "---",
        "P" => ".--.", "Q" => "--.-", "R" => ".-.", "S" => "...", "T" => "-",
        "U" => "..-", "V" => "...-", "W" => ".--", "X" => "-..-", "Y" => "-.--",
        "Z" => "--.."
      }
  
      def encode(message), do: do_encode(message, [])
    
      defp do_encode(<<>>, result),
        do: result |> Enum.reverse |> Enum.join(" ")

      defp do_encode(<<letter::binary-size(1), rest::binary>>, acc),
        do: do_encode(rest, [Map.get(@alphabet, letter) | acc])

    end
```

Here we replaced the *"O"* and the *"S"* cases with a function that performs a lookup in a map of letters to their code in the morse code alphabet.

We are almost there, but we only handle single words at this point. Let us fix that!


Handling sentences
------------------
Some people use forward slashes to denote new words when writing morse code. Add the following assertion to the unit tests:

```elixir
    assert Morse.encode("HELLO WORLD") == ".... . .-.. .-.. --- / .-- --- .-. .-.. -.."
```

The tests will fail. Our implementation will insert an extra space instead of a slash because the morse code map will return `nil` when it perform the lookup for the non-existent `" "` key. This will cause the extra whitespace when joining the list using `Enum.join/2`.

The solution is simple; we need to add a special case for whitespaces. The patterns are evaluated from the top to bottom, so insert the special case before the `do_encode` that extract the letter. The special case should look like this:

```elixir
    defp do_encode(<<" ", rest::binary>>, acc),
      do: do_encode(rest, ["/" | acc])
```

If you entered it correctly the unit tests should pass.

Now we have a fully working morse code encoder, and it is fairly performant, but just have performant is it? In the next section we will add a third party benchmarking framework and check out the performance.


Benchmarking the encoding
-------------------------
[Benchfella](https://github.com/alco/benchfella) is a benchmarking framework build by [alco, aka Alexei Sholik](https://github.com/alco). It is available on [hex.pm](https://hex.pm/) so it can be installed by adding it to our projects *mix.exs*-file (follow the instructions on the github readme for detailed instructions.)

When the Benchfella dependency has been added to the project and fetched from Hex we should create a *bench/* folder in the root of our project. Add a file called *encode_bench.exs* to the folder and add the following content:

```elixir
# bench/encode_bench.exs
    defmodule EncodeBench do
      use Benchfella

      setup_all do
        words = File.stream!("/usr/share/dict/words")
        {:ok, words}
      end

      bench "morse encode every word in a dictionary" do
        Enum.map(bench_context, &Morse.encode/1)
      end
    end
```

When running the `mix bench` task on my computer I get 7.50 seconds. This is okay, considering that the dictionary consist of 235886 words on my computer.


Decoding morse encoded text
---------------------------
When we transmit something using morse we might get answers back, and they might be encoded using morse. We need to make a module that decode morse code to clear text.

### Code reorganization
First off. Let us reorganize our code a bit. If we start mixing the code that perform encoding and decoding our file size might explode, and if we have a friend helping us out on the same code base merge conflicts might occur. Simple solution, create files to store the Encoder and Decoder in.

Go to the *lib/* folder and add a folder called *morse/*. Create a file called *encoder.ex* in this folder and copy the contents of the morse.ex with the following modifications:

```elixir
    # lib/morse/encoder.ex
    defmodule Morse.Encoder do
      @alphabet %{
        "A" => ".-", "B" => "-...", "C" => "-.-.", "D" => "-..", "E" => ".",
        "F" => "..-.", "G" => "--.", "H" => "....", "I" => "..", "J" => ".---",
        "K" => "-.-", "L" => ".-..", "M" => "--", "N" => "-.", "O" => "---",
        "P" => ".--.", "Q" => "--.-", "R" => ".-.", "S" => "...", "T" => "-",
        "U" => "..-", "V" => "...-", "W" => ".--", "X" => "-..-", "Y" => "-.--",
        "Z" => "--.."
      }

      def encode(message), do: do_encode(message, [])

      defp do_encode(<<>>, result),
        do: result |> Enum.reverse |> Enum.join(" ")

      defp do_encode(<<" ", rest::binary>>, acc),
        do: do_encode(rest, ["/" | acc])

      defp do_encode(<<letter::binary-size(1), rest::binary>>, acc),
        do: do_encode(rest, [Map.get(@alphabet, letter) | acc])

    end
```

The only difference is that we changed the module name (after `defmodule`) from `Morse` to `Morse.Encoder`. It could be called anything, but it is a good convention to call the name of the module by the folder structure in which it resides.

Change the *lib/morse.ex* file to look like this:

```elixir
    # lib/morse.ex
    defmodule Morse do
      defdelegate encode(message), to: Morse.Encoder
    end
```

`defdelegate` will delegate calls to `Morse.encode` to `Morse.Encoder.encode`, ensuring our API stays the same and that our unit tests should pass after this change.


### Creating the decoder
Add a file called *decoder.ex* in the *lib/morse/* folder, and add a delegate in the *lib/morse.ex* that delegates `decode(message)` to `Morse.Decoder`.

Add a new block in the test suite that uses the `Morse.decode(message)` to do the reverse of the previous tests. Start out with the first couple of tests.

```elixir
    # ...
    # test/morse_test.exs
    test "morse decoder" do
      assert Morse.decode("") == ""
      assert Morse.decode("...") == "S"
      assert Morse.decode("---") == "O"
      assert Morse.decode("... --- ...") == "SOS"
    end
```

Let us implement the decoder using the same approach as we did when we did the encoder. Start out with the simplest cases and work from there.

```elixir
    # lib/morse/decoder.ex
    defmodule Morse.Decoder do
      def decode(message), do: do_decode(message, [])

      defp do_decode(<<>>, acc),
        do: acc |> Enum.reverse |> Enum.join("")

      defp do_decode(<<" ", rest::binary>>, acc),
        do: do_decode(rest, acc)
    
      defp do_decode(<<"...", rest::binary>>, acc),
        do: do_decode(rest, ["S" | acc])

      defp do_decode(<<"---", rest::binary>>, acc),
        do: do_decode(rest, ["O" | acc])

    end
```

Nothing new is going on here. We added a special case `" "` that basically just ignore the empty string, passing the remainer of the message along with the accumulator to the `do_decode` function.

While this implementation handle letters and even words (that consist of *S* and *O*) it will cause us some problems as we continue our decode implmentaion. We need to know when a letter is complete before decoding it, becuase unlike the encode function, the letters we are encoding varies in length. Morse code *T* is a single dash, and *E* is a single dot, which would conflict with any other code that start with either.

Let it be said, to get the task done quickly we could probably just split the message on spaces using `String.split` and `Enum.map` a function on the list that decode every code, but what is the fun in that? Let us instead scan for the entire word using another accumulator.

*(to be continued)*
