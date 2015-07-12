# The LEXER Package

The `lexer` package is a tokenizer for Common Lisp that makes heavy use of the [`re` package](http://github.com/massung/re).

## Creating a Lexer Function

The `lexer` package allows you to use regular expressions to pattern match against an input buffer and return tokens. It does this much in the same way that [Lex](https://en.wikipedia.org/wiki/Lex_%28software%29) does. Using the `deflexer` macro, you can define a function that will attempt to match a list of patterns against a buffer.

	(deflexer lexer (state-var) &body patterns)

The *state-var* is the current lexical state: it tracks the lexical buffer being parsed as well as a stack of lexical functions (defined with `lexbuf`)... more on this later.

Here is a simple example:

	CL-USER > (deflexer my-lexer (state)
	            ("%s+"   (values :next-token))
	            ("="     (values :eq))
	            ("%a%w*" (values :ident $$))
	            ("%d+"   (values :int (parse-integer $$))))
	MY-LEXER

*NOTE: If you don't understand the `$$` symbol in the example above, please see [this README](http://github.com/massung/re/README.md).*

Each pattern should either return `nil` - indicating the end of the input buffer has been reached - or (up to) two values: the class of the token and the value of the token. Returning `:next-token` is special, and indicates that this token should just be skipped.

## Tokenizing

Now that we have a lexer function, the `tokenize` function can be called to parse a source string.

	(tokenize lexer string &optional source)

The `lexer` is our function and `string` is what will be tokenized. The `source` argument can be used to identify where `string` came from (e.g. a pathname), as it will be used in error reporting.

*NOTE: A lexical state object is created for you always, and there is never a need to create one yourself.*

Let's give it a try:

	CL-USER > (tokenize 'my-lexer "x = 10")
	(#<LEXER::TOKEN IDENT "x">
	 #<LEXER::TOKEN EQ>
	 #<LEXER::TOKEN INT 10>)

If all the patterns in your lexer function fail to match, then a `lex-error` condition is signaled, letting you know exactly where the problem is located at.

	CL-USER > (tokenize 'my-lexer "x = $10" "REPL")
	Error: Lexing error on line 1 of "REPL"
	  1 (abort) Return to level 0.
	  2 Return to top loop level 0.

## Analyzing Tokens

Almost all parser generators for Common Lisp ([PARSERGEN](http://www.lispworks.com/documentation/lw61/LW/html/lw-1141.htm#pgfId-886156), [CL-YACC](http://www.pps.univ-paris-diderot.fr/~jch/software/cl-yacc/), ...) work by defining a single function, which takes a function as an argument that will return tokens to it as multiple value pairs: the class and value of the token.

To accomodate these parsers, the `lexer` package has a function: `parse`, which defines a lexer and calls the parser function with an appropriate token reader:

	(parse parser lexer string &optional source)

*NOTE: From here on we'll be using the [LispWorks](http://www.lispworks.com/) parser generator for examples.*

Let's create a simple parser grammar.

	CL-USER > (defparser my-parser
	            ((start let))
                ((let :ident :eq :int)
                 `(:let ,$1 ,$3))
                ((let :error)
                 (error "Invalid LET syntax")))
	MY-PARSER

Now let's use the parser in conjunction with the lexer to parse a string...

	CL-USER > (parse 'my-parser 'my-lexer "x = 10")
	(:LET "x" 10)
	NIL

Since each token knows where it came from, the `parse` function also will re-signal any condition thrown as a `lex-error` identifying where in the token stream the error occurred, as well as the lexeme of the token.

	CL-USER > (parse #'my-parser #'my-lexer "x = 10 20")
	Error: Invalid LET syntax near "20" on line 1
	  1 (abort) Return to level 0.
	  2 Return to top loop level 0.

And done!

## Multiple Rules

Often, you will be parsing text that has different lexical rules given the current context. For example, HTML allows embedding JavaScript between `<script>` tags, and in many languages quoted strings are a mini-DSL unto themselves.

The lexer functions you create with `deflexer` all take a `lexstate` object as a parameter. The `lexstate` actually contains a stack of lexers, the top-most which is the one being currently used to tokenize the input source. Within a lexer, you can push, pop, and swap to different lexers, while also returning tokens.

	;; push a new lexer, return a token
	(push-lexer state lexer class &optional value)
	
	;; pop the current lexer, return a token
	(pop-lexer state class &optional value)

	;; swap to a different lexer, return a token
	(swap-lexer state lexer class &optional value)
	
Each of these will change the current lexer, and also return a token at the same time! This is very useful as the token can be used to signal to the grammar to change parsing rules.

*NOTE: Remember that `:next-token` is treated special. If you return `:next-token` while also changing lexers, the new lexer will not be called until after a complete token has been returned from your current lexer! You'll almost never want to do this.*

Let's give this a spin by creating a simple CSV parser. It should be able to parse integers and strings, and strings should be able to escape characters and contain commas.

First, let's define the CSV lexer:

	CL-USER > (deflexer csv-lexer (state)
	            ("%s+"      (values :next-token))

	            ;; tokens
	            (","        (values :comma))
	            ("%-?%d+"   (values :int (parse-integer $$)))

	            ;; string lexer
	            ("\""       (push-lexer state #'string-lexer :quote)))
	CSV-LEXER

Notice how when we hit a `"` character, we're going to push a new lexer onto the the `lexstate` and return a `:quote` token. The `:quote` token will signal to the grammar that we're now parsing a string.

Next, let's define our string lexer.

	CL-USER > (deflexer string-lexer (state)
	            ("\""       (pop-lexer state :quote))

	            ;; characters
	            ("\\n"      (values :chars #\newline))
	            ("\\t"      (values :chars #\tab))
	            ("\\(.)"    (values :chars $1))
	            ("[^\\\"]+" (values :chars $$))

	            ;; end of line/source
	            ("%n|$"     (error "Unterminated string")))
	STRING-LEXER

This lexer has several interesting things going on. First, we see that when we find the next `"` character that we pop the lexer (returning to the CSV lexer) and also return a `:quote` token. Next, we can see that it handles escaped characters and then any number of characters up until the next backspace (`\\`) or quote (`"`). Finally, if it reaches the end of the line or file, it will signal an error.

Let's try tokenizing to see what we get.

	CL-USER > (tokenize #'csv-lexer "1,\"hello, world\",2")
	(#<LEXER::TOKEN INT 1>
	 #<LEXER::TOKEN COMMA>
	 #<LEXER::TOKEN QUOTE>
	 #<LEXER::TOKEN CHARS "hello, world">
	 #<LEXER::TOKEN QUOTE>
	 #<LEXER::TOKEN COMMA>
	 #<LEXER::TOKEN INT 2>)

Perfect! Now let's write the grammar...

	CL-USER > (defparser csv-parser
	            ((start values) $1)

	            ;; a list of values
	            ((values value :comma values) `(,$1 ,@$3))
	            ((values :comma values) `(nil ,@$2))
	            ((values value) `(,$1))

	            ;; a single value
	            ((value :quote chars) (coerce $2 'string))
	            ((value :int) $1)
	            ((value :error) (error "Illegal CSV"))

	            ;; strings to concatenate
	            ((chars :chars chars) (string-append $1 $2))
	            ((chars :quote) ""))

And let's give it a spin:

	CL-USER > (parse #'csv-parser #'csv-lexer "1,\"hello, world!\",,,2")
	(1 "hello, world!" NIL NIL 2)
	NIL
	
That's it!

# More (Usable) Examples

Here are some lexers used to parse various file formats. As with this package, they are released under the Apache 2.0 license and are free to use in your own projects.

* [JSON](http://github.com/massung/json)
* [XML](http://github.com/massung/xml)
* [HTTP](http://github.com/massung/http)

More examples coming as I need them...

# Thank You!

If you get some good use out of this package, please let me know; it's nice to know your work is valued by others. If you find a bug or made a nice addition (especially if you improved performance!) please feel free to drop me a line and let me know at [massung@gmail.com](mailto:massung@gmail.com).