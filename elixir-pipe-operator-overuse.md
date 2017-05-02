---
layout: post
title:  "Overuse of Elixir's Pipe Operator"
date:   2017-05-01 18:38:57 +0100
---
Next to pattern matching, one of Elixir's language features that tends to get
programmers equally as excited has to be the pipe operator. The pipe operator
takes the result of the expression on its left-hand side and inserts it as the
first argument to the function call on its right-hand side. This allows you to
write code that could otherwise only be expressed by nesting function calls or
saving intermediate results to variables in an elegant manner that resembles a
pipeline.

Take, for example, the following code which creates a normalized form of a word:

```elixir
def normalize(word) do
  word                     # "ElixirConf"
  |> String.downcase()     # "elixirconf"
  |> String.graphemes()    # ["e", "l", "i", "x", "i", "r", "c", "o", "n", "f"]
  |> Enum.sort()           # ["c", "e", "f", "i", "i", "l", "n", "o", "r", "x"]
end
```

Effectively, this is just another way of writing:

```elixir
defp normalize(word) do
  Enum.sort(String.graphemes(String.downcase(word)))
end
```

We can confirm that by examining the Abstract Syntax Tree from the body of `normalize/1`:

{% raw %}
```elixir
iex(17)> ast = quote do
...(17)>   word
...(17)>   |> String.downcase()
...(17)>   |> String.graphemes()
...(17)>   |> Enum.sort()
...(17)> end
{:|>, [context: Elixir, import: Kernel],
 [{:|>, [context: Elixir, import: Kernel],
   [{:|>, [context: Elixir, import: Kernel],
     [{:word, [], Elixir},
      {{:., [], [{:__aliases__, [alias: false], [:String]}, :downcase]}, [],
       []}]},
    {{:., [], [{:__aliases__, [alias: false], [:String]}, :graphemes]}, [],
     []}]},
  {{:., [], [{:__aliases__, [alias: false], [:Enum]}, :sort]}, [], []}]}

iex(18)> ast |> Macro.expand(__ENV__) |> Macro.to_string()
"Enum.sort(String.graphemes(String.downcase(word)))"
```
{% endraw %}

The former representation is arguably more readable than the latter since it tends
to more directly represent the way our brain thinks about the problem: I take a word,
convert it to lowercase, split it into characters and sort them.

The second version forces us to think in reverse which is much more unnatural: I sort
the result of the list of characters that were converted to lowercase of the word I
started with.

Idiomatic Erlang avoids this by storing intermediate results to variables in each step.
This is slightly more verbose, yet very clear in its intention and maintains the natural
order in which we think about the data transformation:

```erlang
normalize(Word) when is_binary(Word) ->
  String = binary:bin_to_list(Word),
  LowerCased = string:to_lower(String),
  Sorted = lists:sort(LowerCased),
  Sorted.
```


## Notes

There are multiple ways to write something down and generally, because it's Elixir, we all reach for the pipe operator when you'd otherwise nest things or use temporary variables to capture a value. Sometimes, it can be more understandable to just use a variable to store an intermediate result:

```elixir
results = view(db_name, view, keys)
results |> Mapper.results_to_structs()
```

The pipe operator tends to make things more understandable when the first argument of the function is the data that's being transformed by passing it through the pipe. Example:

```elixir
input |> String.trim() |> String.split()
```

https://github.com/christopheradams/elixir_style_guide#bare-variables
