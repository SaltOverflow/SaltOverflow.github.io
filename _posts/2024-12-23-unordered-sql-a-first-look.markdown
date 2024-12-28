---
layout: post
title:  "Unordered SQL, a first look"
date:   2024-12-27 09:00:00 -0500
categories: jekyll update
---
SQL is a very concise language, at least for the main query use-cases. But I've always had one problem: you can't get autocomplete for the `SELECT` columns because you haven't written the `FROM` clause yet. I understand that putting the `SELECT` clause first is more declarative, but nowadays I'd like to be able to write the `FROM` clause first instead.

I call this language Unordered SQL. The idea is to let programmers write their `SELECT`, `FROM` and `WHERE` clauses in any order, even allowing multiple of them in the same statement. A formatter can then transform it into regular SQL. The repository for this project can be found here: [Unordered SQL][Unordered SQL]. I also created a [Youtube video][Youtube video] to explain the details of the system.

For this first attempt, I opted to try writing the tokenizer and lexer from scratch. This allowed me to try any features I wanted to try out in this prototype.

# Features

## Zero-copy semantics

The Abstract Syntax Tree (AST) that is generated from parsing needs to have indexes to the original text, in order to figure out where the cursor is in the AST. This custom restriction is one of the reasons I wrote the tokenizer and lexer from scratch, because I found that existing parser generators are somewhat restrictive in what they support. Once you're storing the string indexes, there's no need to store the string itself, and the system is likely more performant due to better cache locality.

![Storing indexes to the original text in the AST. The multiply expression stores the indexes 3 and 9, rather than the string original_text\[3:9\]](/assets/zero_copy_example.png)

## Recursive descent parsers

I argue that a recursive descent parser is the best choice if you are to write a parser by hand. Clang uses recursive descent for its parser, highlighting [performance, diagnostics and error recovery, and simplicity][Clang recursive descent benefits].

Recursive descent parsers parse code very similarily to humans (my opinion). With a recursive descent parser, I know I'm parsing a `FROM` clause because the name of the function is `parse_from()`. I also know that any behaviour I put in this function will only happen if I'm parsing a `FROM` clause. This makes it easier to debug and add functionality.

Writing a parser by hand can also call attention to language designs that may be confusing for a human. A language designer could then look into ways to make the syntax more readable. In C, `char *arr[10];` is an [array of 10 pointers to char][Clockwise rule], but it takes some time to figure out. This is evidenced by the fact that many parsers need to parse this statement twice to figure out the type. In [Cforall][Cforall] (an open-source project that extends C), the same statement is written as `[10] * char arr`, which is arguably more readable and straightforward to parse.

Recursive descent parsers make it simple to handle errors, too. This is very important when performing autocomplete, as you're often dealing with slightly invalid syntax. If I'm currently parsing an expression and I encounter an unknown token, I can simply return what I currently have and let the parent function handle the rest. I can then add behaviour like adding a placeholder variable and marking off a sequence of tokens as "unknown."

The downside is that it's all custom code, so it can take longer to write. It's also possible to design it weirdly and end up with [spaghetti code][Spaghetti code], compared to using a framework like LALR grammars. There is no such thing as a free lunch - everything is a tradeoff of options.

## What counts as an operator?

I would argue the most complex part of any parser is handling expressions. Much of that complexity can be mitigated using [operator-precedence techniques][Kaleidoscope language]. Such a system provides a nice and compact way to parse an expression.

However, it can be a little tricky to figure out where each operator starts and ends in a string - I spent way too much time thinking about this. Consider `a>!b`. The issue with SQL is that there are a multitude of SQL variants out there - what if one of them has `>!` as an infix operator? I wanted to enforce readability, so I considered the operator to be the largest contiguous sequence of operator-like characters. For example, `a<~b` would be treated like `<~` was some sort of operator. This works fine for SQL, aiding readability by forcing programmers to put a space between the `+` characters in `a++b`. As a side note, this strategy does not work for C because no one wants to write `* * *b` instead of `***b`. There's more to look into here, but I'll save that for a later post.

## Array representation for expressions

Representing `a.b.c * d + e` in an AST is relatively tedious, but there is a trick you can use to make this expression more compact, which I call the *array representation for expressions*. The operators in the example are in descending precedence, so we can represent this as two arrays: `[a b c d e] [. . * +]` (commas omitted for readability). You could also represent prefix and ternary operators: for `+a ? b : c` we'd have `[a _ b c] [+(prefix) ?:(ternary) _]`. If the expression was not in descending precedence, you would recurse: for `a + b * c` we'd have `[a <expr2>] [+]` where `<expr2>` is `[b c] [*]`. This should provide a performance boost, but more importantly, it's much easier to work with when doing semantic analysis: `a.b.c` is all represented in one array rather than having to recurse through all of the nodes of the AST while keeping track of semantic information.

I took a quick look at Clang, and it [does not seem to use this representation][Clang Expr]. I assume it's just not worth it to rearchitect the system when parsing is far from being a performance bottleneck (it's all O(n) at the end of the day). Maybe I'll look into it at some point, or someone else can.

## The tokenizer

The tokenizer (also called lexer) exposes two functions, `peek(k)` and `consume()`, and the best part is in its simplicity. Internally, it has a queue of tokens which are generated from the text as `peek(k)` and `consume()` are called. The reason why we need `peek(k)` is because we sometimes need to look ahead: it's only at `2 + 3 *` that we realize that grouping `2 + 3` together would be wrong.

Technically speaking, you don't need a tokenizer - you could do it all in the parser. However, splitting the functionality makes sense from a modularity standpoint (test smaller parts instead of all at once) and from a conceptual standpoint (tokenizer uses regex to produce a stream of tokens, parser uses a grammar to produce an AST).

Another way to write the tokenizer is to expose `consume()` and `backtrack()` to return the next token and reset the tokenizer's position to before the last token. However, I dislike this format because it moves the tokenizer's position all over the place, and the bugs that this can cause can be quite challenging to fix.

# Unanswered questions

**How to update the AST as edits are made?** We would want to avoid reparsing the entire file every time an edit is made. It should be possible to only reparse a single SQL statement using an ad-hoc method, though if you try to go even further, there is a risk of creating a parsing mismatch. It is also probably unnecessary to go further from a performance standpoint.

**What about type-checking?** In fact, there is no semantic analysis being done in this project besides some simple matching on column names. That being said, a lot of thought has been put into setting up the AST in such a way that semantic analysis can be as streamlined as possible, as shown by this post.

**What about other SQL statements?** At the moment, Unordered SQL only concerns itself with the `SELECT` SQL statement. This would include things like subqueries and other clauses like `LIMIT` and `ORDER BY`, which are not currently implemented by my project. For the other SQL statements, I would argue against "unordering" them because many of them make edits to the database itself. In those cases, correctness is more important than saving some time writing the query.

# What now?

In hindsight, the extension for the SQL language was needlessly complex. It was originally just about switching the FROM and the SELECT clauses around, then evolved into having the clauses being in any order, then any number of clauses in any order. Part of this was my problem of needing to "justify the project": Simply switching the `FROM` and `SELECT` clauses seemed too easy to count as a viable project. I eventually ended up with a number of interesting features I wanted to implement in the parser, which expanded the scope well beyond what was reasonable for a course project.

By starting off having a language syntax that was "too flexible," I often found myself grappling with how to make many downstream features work, like semantic analysis and AST structure. I also worried a lot about whether I made my language syntax too flexible, which could encourage bad programming patterns; this would, in turn, make my language harder to read.

My next goal is to take a database client like DBeaver and flip the order of the `FROM` and `SELECT` clauses. I am particularily interested in seeing how they handled parsing, autocomplete and edits, as I found those parts to be conceptually very tricky to get right while working on my version.

[Unordered SQL]: https://github.com/SaltOverflow/unordered-sql
[Youtube Video]: https://youtu.be/yYmXDXmFFT8
[Clang recursive descent benefits]: https://stackoverflow.com/a/6382123
[Clockwise rule]: https://c-faq.com/decl/spiral.anderson.html
[Cforall]: https://cforall.uwaterloo.ca/
[Spaghetti code]: https://en.wikipedia.org/wiki/Spaghetti_code
[Kaleidoscope language]: https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/LangImpl02.html#binary-expression-parsing
[Clang Expr]: https://clang.llvm.org/doxygen/classclang_1_1Expr.html
