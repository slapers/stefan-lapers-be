---
title: "Writing a boolean expression parser in Elixir using NimbleParsec"
date: 2018-08-19T13:56:51+02:00
draft: false
---

Lately I spent some time writing a parser using [NimbleParsec](https://github.com/plataformatec/nimble_parsec) and was really pleased with how the library works. So I decided to write up a small tutorial on building a parser for boolean logic and how I approached building the grammar and the parser.

This is what we will attempt to parse:

- `true` and `false` The boolean constants
- `()` Grouping of expressions
- `not` Negation of expressions
- `and` Logical conjunction of expressions
- `or` Logical disjunction of expressions

## Precedence rules

To correctly parse boolean expressions we need to take into account the [operator precedence rules](http://intrologic.stanford.edu/glossary/operator_precedence.html). They are listed underneath from highest to lowest precedence.

```elixir
# (highest to lowest)
() -> not -> and -> or

# Some examples how precedence rules apply:
A || B && C     ==  (A || (B && C))
(A || B) && C   ==  ((A || B) && C)
! A && B && C   ==  (((!A) && B) && C)
```

## Grammar for our language

Lets get started by defining an [EBNF](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form) for our language. To be honest for me this is where I spent most of the time. Once the grammar is defined, the implementation in NimbleParsec is just a walk in the park.

We will build it up while we go along.

### Constants

Lets define our true and false values, and begin defining our expression as one of these constants.

```bnf
<expr> :== <const>
<const> :== "true" | "false"
```

### Grouping expressions

Now that we have our constants defined we can start implementing the operators following the operator precedence. First one to tackle is: `()`, which allows grouping expressions.

```bnf
<expr> :== "(" <expr> ")" | <const>
<const> :== "true" | "false"
```

As you can see the `<expr>` just became recursive. It is not that powerful yet because we only have `<const>` in our grammar, but it will be pretty amazing later on.

### Negation

Lets make it possible to negate our `<expr>`, our grammar becomes:

```bnf
<expr> :== <not> <expr> | "(" <expr> ")" | <const>
<const> :== "true" | "false"
<not> :== "!"
```

With our grammar so far we can declare expressions like:

```javascript
!(!(!true))
!!false
```

### Conjunction

So this one is a bit more involved, basically we want to make it possible to repeat the `and` operator zero or multiple times:

```bnf
<expr> && <expr> && <expr>
```

We are going to rename the `<expr>` we had previously defined into `<factor>` so we can optionally repeat it. Furthermore we will add the `<and>` operator. 

Our grammar now becomes:

```bnf
<expr> :== <factor> {<and> <factor>}
<factor> :== <not> <factor> | "(" <expr> ")" | <const>
<const> :== "true" | "false"
<not> :== "!"
<and> :== "&&"
```

In EBNF `{...}` means 0 or multiple repetitions of what is inside the braces. So `<factor> {<and> <factor>}` reads as `<factor>` optionally followed by multiple `<and> <factor>`

Its worth noting that we did not change the `"(" <expr> ")"` inside `<factor>`...

### Disjunction

Finishing with the: `or` operator, basically this operator is just like the `<and>` operator, and like in the previous step joins the `<expr>` zero or multiple times:

```bnf
<expr> || <expr> || <expr>
```

Again we rename the `<expr>` we had previously defined, this time into `<term>`. And we add the `<or>` operator. 

Our grammar finally becomes:

```bnf
<expr> :== <term> {<or> <term>}
<term> :== <factor> {<and> <factor>}
<factor> :== <not> <factor> | "(" <expr> ")" | <const>
<const> :== "true" | "false"
<not> :== "!"
<and> :== "&&"
<or>  :== "||"
```

Again we did not change the `"(" <expr> ")"` inside `<factor>`

## Parser implementation

Ok, now we can start building our parser using the grammar we defined above.

Some remarks:

- I will not add any whitespace handling to keep the code concise. You can have a look at [https://github.com/slapers/ex_sel](https://github.com/slapers/ex_sel) which handles whitespaces and adds arithmetics and comparison operations.

- You can find the parser in the following [gist](https://gist.github.com/slapers/d8c7f7532f5d340922dd142b7d852999)

### Project setup

Create a new project using mix:

```shell
$ mix new parser
$ cd parser
```

Edit the mix.exs file and add the nimble parsec dependency:

```elixir
defp deps do
  [{:nimble_parsec, "~> 0.4"}]
end
```

Install the dependencies:

```shell
$ mix deps.get
```

### Parser code and tests

```elixir
defmodule Parser do
  import NimbleParsec

  not_ = string("!") |> label("!")
  and_ = string("&&") |> replace(:&&) |> label("&&")
  or_ = string("||") |> replace(:||) |> label("||")
  lparen = ascii_char([?(]) |> label("(")
  rparen = ascii_char([?)]) |> label(")")

  # <const> :== "true" | "false"
  true_ = string("true") |> replace(true) |> label("true")
  false_ = string("false") |> replace(false) |> label("false")
  const = choice([true_, false_]) |> label("boolean")

  # <factor> :== <not> <expr> | "(" <expr> ")" | <const>
  negation = not_ |> ignore |> parsec(:factor) |> tag(:!)
  grouping = ignore(lparen) |> parsec(:expr) |> ignore(rparen)
  defcombinatorp(:factor, choice([negation, grouping, const]))

  # <term> :== <factor> {<and> <factor>}
  defcombinatorp(
    :term,
    parsec(:factor)
    |> repeat(and_ |> parsec(:factor))
    |> reduce(:fold_infixl)
  )

  # <expr> :== <term> {<or> <term>}
  defparsec(
    :expr,
    parsec(:term)
    |> repeat(or_ |> parsec(:term))
    |> reduce(:fold_infixl)
  )

  defp fold_infixl(acc) do
    acc
    |> Enum.reverse()
    |> Enum.chunk_every(2)
    |> List.foldr([], fn
      [l], [] -> l
      [r, op], l -> {op, [l, r]}
    end)
  end
end
```

And to top it off some tests:

```elixir
defmodule ParserTest do
  use ExUnit.Case
  doctest Parser

  defp parse(input), do: input |> Parser.expr() |> unwrap
  defp unwrap({:ok, [acc], "", _, _, _}), do: acc
  defp unwrap({:ok, _, rest, _, _, _}), do: {:error, "could not parse" <> rest}
  defp unwrap({:error, reason, _rest, _, _, _}), do: {:error, reason}

  @err "expected boolean while processing !, " <>
         "followed by factor or (, followed by expr, followed by ) or boolean"

  test "parses consts" do
    assert true == parse("true")
    assert false == parse("false")
    assert {:error, @err} == parse("1")
  end

  test "parses negations" do
    assert {:!, [true]} == parse("!true")
    assert {:!, [false]} == parse("!false")
  end

  test "parses conjunctions" do
    assert {:&&, [{:!, [true]}, false]} == parse("!true&&false")
    assert {:!, [{:&&, [false, true]}]} == parse("!(false&&true)")
  end

  test "parses disjunctions" do
    assert {:||, [true, {:&&, [true, false]}]} == parse("true||true&&false")
    assert {:&&, [{:||, [true, true]}, false]} == parse("(true||true)&&false")
  end
end
```


