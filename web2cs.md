# **tex.web** transformed to **tex.cs**

## Abstract
**tex.web** transformed into C#; not another port, rather the `WEB` is preserved in IDE-friendly C#.

## Introduction

I've transformed **tex.web** into C# (and C++).  Note the wording; this is not another of port TeX to port
_some language_.  Rather, **tex.cs** follows the presentation of the `WEB` while still being normal
IDE-friendly C#.  The result is a version of TeX that is structurally faithful to **tex.web**,
usable in a modern IDE, and passes the TRIP test; without using `tangle`/`ctangle`.

## Why and why now?

The 'why now?' is easy: Knuth was born in 1938; regrettably, that means we're likely getting close to
TeX version `$\pi$` \cite{Knuth:TB11-4-489}.  TeX is extremely stable, **tex.web** isn't going to change much;
so it's transformation into C# (or C++) can be done manually one time without fear of a moving target.
Should we be fortunate enough to get TeX 3.1415926539 (currently scheduled for 2029~\cite{Knuth:2021:TT})
any changes to **tex.web** can readily be incorporated 'by hand.'

'Why?' is a bit less clear more multi-faceted.  In no particular order, reasons might include:

* **tex.web** might be one of the more important programming artifacts. But its form
makes it increasingly inaccessible to modern developers. This project preserves **tex.web**'s structure and
narrative while expressing the code in C#.

* Demonstrating that a 40-year old Pascal program can be modernized;
perhaps software engineers want to rewrite a bit too much? 

* There are few non-trivial programs (other than METAFONT) which have been published as a book , it's quite unusual to
have such a canonical reference for a program. And while Knuth's dream of literate programming never took off,
we still struggle with understanding code today.

* Knuth once remarked that he was slightly disappointed in the lack of experimentation withTeX itself~\cite{knuth:new-versions};
outside of a few widely used and well-supported extensions, most people achieve the desired behavior
through TeX's own macros rather than changing the source code.  This, despite clear hooks and suggestions
in the code itself (and even Knuth's own TeX-XeT~\cite{Knuth:TB8-1-14}). TeX 3.0 was released in 1990 with everything essentially
frozen; this was just as IDEs were starting to become available, `tangle`-like preprocessing just isn't done anymore.

* While function-call overhead is still important, in most cases, these decisions are better left to the compiler rather
than the programmer; especially for a program like TeX which is unlikely to be on a critical hot path.

* Programming languages now have constructs such as lambdas and reflection which
make it possible to do `tangle`-like code manipulation in the language itself.

* **tex.web** is a modest 25,011 lines, making such an undertaking approachable for one
person using a manual process.

* The TRIP test to helps ensure that the result is truly TeX, not just something that `compiles and runs.'

## Creating **tex.cs**

What follows is a description of the mostly manual process of creating a readable **tex.cs** from **tex.web**.
No tools were created to enable the process to be repeated; there likely isn't much interest in converting
other TeXware to C# in this manner.  (This isn't quite true; I first transformed `DVIType`,
both as a proof-of-concept and for eventual use in running the `TRIP` test.)

Undoubtedly, the single most important tool used was Martin Ruckert's `web2w`~\cite{Ruckert:2017:CTW}.
This removed the tedium of munging Pascal syntax into C; it also largely implicitly answered the question of
how much `WEB`/Pascal syntax should be preserved.  For example, using the C-preprocessor,
`help2("a")("b")` preserved while C# all but requires `help2("a", "b")`. 
`web2w` also removes the \texttt{TEX.POOL} file~\cite{Ruckert:2021:WCC}, even C has better support for strings
than Standard Pascal did in 1982. Thus, much of the C# code is simply the C code from
**tex.w** rather than hand-ported Pascal.

The next step was simple: all `WEB` modules were turned into comment blocks, and I added
back the original Pascal for easy comparison with the C# code.  To enable
possible future `weave`-like processing, I began each comment block with `/*[tex.web]`
and ended it with `[tex.web]*/`.
```
/*[tex.web]
@* \[1] Introduction.
This is \TeX, a document compiler intended to produce typesetting of high
...
@d banner=='This is TeX, Version 3.141592653' {printed when TeX starts}
[tex.web]*/
```
(This module has an empty Pascal part.)  No further attempt was made to 
preserve any other `weave` features; in particular, cross-reference
and index tags in code were removed or commented-out as they are invalid
code; IDEs have a host of code spelunking features.

For example, at line 948 of **tex.web** the '`@!`' before `bypass_eoln`
was just removed from the `web2w` transformed code:
```
boolean input_ln(alpha_file f, boolean bypass_eoln)
```

Much of the C code produced by `web2w` can be compiled as C# with minimal effort;
that's because  **tex.w** is a straight-forward transformation of the Pascal code, it doesn't use
a lot of C features. We get our first array at line 588, using an `ArrayOf` utility class, 
the C# is
```
/*[tex.web]
@ The TeX processor converts between ASCII code and
[...]

@<Glob...@>=
@!xord: array [text_char] of ASCII_code;
	{specifies conversion of input characters}
[...]
[tex.web]*/

ArrayOf<ASCII_code> xord = new(text_char);
    /*specifies conversion of input characters*/
```
which also nicely corresponds to the original `WEB` code.

Implementing a C-runtime in C# targeted at the specific needs of **tex.w** made
it possible to have the C# version passing the TRIP test with minimal changes to
**tex.w** (i.e., using `ctangle` to produce **tex.c**
and then renaming that file to **tex.cs**).

`ArrayOf` is one of several pieces of a minimal Pascal runtime intended to preserve the original
`WEB` code rather than convert to language-specific routines; e.g., Pascal's \texttt{reset()} instead
of .NET's \texttt{File.Open()} or C's \texttt{fopen()}.

## Preserving the structure of **tex.web**
At this point, we have 'just' a port of TeX to C#; the 'C' code in 
**tex.w** is now actually C#.  Yes, this is interesting
and useful (is there an actual TeX implantation in .NET?), but with `ctangle`
in the picture it is cumbersome to use in an IDE that is expecting
real C# source files.

### One large class
The key insight at this point is that both C# and C++ (as well as several other languages)
allow most class members to appear in _any order_, a member can be used before it is declared.
By putting all the code into a single class, the effect of `tangle`'s code-shuffling is
achieved by the compiler.  Doing something similar in C (no classes), requires forward declarations
which detracts from the `WEB` presentation.  The result is that `@<Glob...@>=` (line 460)
and `@<Types...@>=` (line 550) just become in-place code:
```
/*[tex.web]
@ In case somebody has inadvertently made bad settings of the ``constants,''
[...]

@<Glob...@>=
@!bad:integer; {is some ``constant'' wrong?}
[tex.web]*/

integer bad; /*is some ``constant'' wrong?*/
```
and
```
/*[tex.web]
@ Characters of text that have been converted to \TeX's internal form

@<Types...@>=
@!ASCII_code=0..255; {eight-bit numbers}
[tex.web]*/

static readonly Range ASCII_code = 0..255; // eight-bit numbers
```

After that, many of the 'C' parts can become member functions as they are standalone
pieces of code which only use global variables (member data).  The named
'C' part becomes the name of a member function, after replacing spaces
with underscores and removing other characters that can't be part of an identifier.
Yes, this becomes a bit unwieldly in places because `WEB` treats the names
as \texttt{weave} text with full formatting and potentially long names.
```
/*[tex.web]
@ We need a special routine to read the first line of TeX input from
[...]
@<Report overflow of the input buffer, and abort@>=
if format_ident=0 then
	begin write_ln(term_out,'Buffer size exceeded!'); goto final_end;
@.Buffer size exceeded@>
	end
else begin cur_input.loc_field:=first; cur_input.limit_field:=last-1;
	overflow("buffer size",buf_size);
@:TeX capacity exceeded buffer size}{\quad buffer size@>
	end
[tex.web]*/

void Report_overflow_of_the_input_buffer_and_abort()
{
	if (format_ident is null)
	{
		write_ln(term_out, "Buffer size exceeded!"); goto_final_end();
		/*@.Buffer size exceeded@>*/
	}
	else
	{
		cur_input.loc_field = first; cur_input.limit_field = last - 1;
		overflow("buffer size", buf_size);
		/*@:TeX capacity exceeded buffer size}{\quad buffer size@>*/
	}
}
void goto_final_end() { System.Environment.Exit(1); }
```

### Local functions in C#
The first real complication is when a procedure declares some local variables
which are then used in multiple subsequent parts.  In C# this is
handled with local functions~\cite{mslearn:local-functions}, in C++ it's necessary to replicate the 
work of the C# compiler manually. See `short_display` at line 3541:
```
/*[tex.web]
@ Boxes, rules, inserts, whatsits, marks, and things in general that are
sort of ``complicated'' are indicated only by printing `\.{[]}'.

@p procedure short_display(@!p:integer); {prints highlights of list |p|}
    [...]
	else @<Print a short indication of the contents of node |p|@>;
    [...]
[tex.web]*/

void short_display(integer p) /*prints highlights of list |p|*/
{
   [...]
   else Print_a_short_indication_of_the_contents_of_node_p();
   [...]

   /*[tex.web]
   @ @<Print a short indication of the contents of node |p|@>=
   [...]
   [tex.web]*/

   void Print_a_short_indication_of_the_contents_of_node_p()
   {
   [...]
	}
}
```

### Reflection and `[Module]`
Some `WEB` sections are spread throughout **tex.web**, `tangle` combines these
into one block of code; initialization is one clear example, line 736
```
@<Set init...@>=
for i:=0 to @'37 do xchr[i]:=' ';
for i:=@'177 to @'377 do xchr[i]:=' ';
```
and line 1806
```
@<Set init...@>=
deletions_allowed:=true; set_box_allowed:=true;
error_count:=0; {|history| is initialized elsewhere}
```

Using reflection in C# (lambdas in C++), we can arrange for these disparate blocks
of code to be called all at once (\texttt{[Module]} is a custom attribute~\cite{mslearn:custom-attributes}):
```
/*[tex.web]
@ The ASCII code is ``standard'' only to a certain extent, since many
[...]
@<Set init...@>=
for i:=0 to @'37 do xchr[i]:=' ';
for i:=@'177 to @'377 do xchr[i]:=' ';
[tex.web]*/

[Module("Set init...")]
void Set_initial_values_of_key_variables_2_2()
{
	for (i = 0; i <= 31; i++) xchr[i] = ' ';
	for (i = 127; i <= 255; i++) xchr[i] = ' ';
}
```

Initialization often takes place immediately after declaration. Both
C# and C++ (as well as other lanugages) allow ('prefer', and sometimes
even 'require') initialization to occur at the same time as declaration.  (In C# it is
customary to omit initialization that is identical to default initialization).
```
@<Glob...@>=
@!deletions_allowed:boolean; {is it safe for |error| to call |get_token|?}
@!set_box_allowed:boolean; {is it safe to do a \.{\\setbox} assignment?}
@!history:spotless..fatal_error_stop; {has the source input been clean so far?}
@!error_count:-1..100; {the number of scrolled errors since the
  last paragraph ended}
[tex.web]*/

int history; /*has the source input been clean so far?*/
int error_count; /*the number of scrolled errors since the
	last paragraph ended*/

/*[tex.web]
@ The value of |history| is initially |fatal_error_stop|, but it will
be changed to |spotless| if TeX survives the initialization process.

@<Set init...@>=
deletions_allowed:=true; set_box_allowed:=true;
error_count:=0; {|history| is initialized elsewhere}
[tex.web]*/

boolean deletions_allowed = true; /*is it safe for |error| to call |get_token|?*/
boolean set_box_allowed = true; /*is it safe to do a \.{\\setbox} assignment?*/
```

### Non-local `goto`s
Another complication is that many `WEB` modules contain `goto`s.
When `tangle` combines the all the modules, they are local `goto`s;
however, as a separate functions this is no longer the case. Exceptions can be used
to emulate these non-local `goto`s. Although this sounds
bad, it is normally rather structured jumping, usually to a spot near the beginning or end of a
significant block.

The code for catching exceptions is verbose
compared 'nothing' in `WEB`, but thankfully is rather limited.  Here is the code at
line 1896:
```
/*[tex.web]
@ @<Get user's advice...@>=
loop@+begin continue: [...]
  end
[tex.web]*/

void Get_users_advice()
{
	while (true)
	{
	@continue: try
		{
		[...]
		}
		catch (GotoException ex) when (ex.Label is "continue") { goto @continue; }
	}
}
```

### Partial classes in C#
There are a few functions/procedures in **tex.web**
that are too large to be presented all at once, some of
them are even spread across multiple starred modules.  Because of other
intervening code, C#'s local functions won't work.

The solution is to use a feature rather unique to C#: `partial` classes~\cite{mslearn:partial-classes}.
This allows the contents of the class to be spread across multiple places in the source
code, each writing \texttt{partial class \emph{name}}, the C# compiler
combines these pieces into a single class.  That, together with a helper
`Ref` class to reference the same member data as the containing
`TEX` class make it possible to write the following for the code at line 12319:
```
/*[tex.web]
@ The recursive procedures |hlist_out| and |vlist_out| each have local variables
[...]
procedure hlist_out; {output an |hlist_node| box}
label reswitch, move_past, fin_rule, next_p;
var base_line: scaled; {the baseline coordinate for this box}
[...]
begin cur_g:=0; cur_glue:=float_constant(0);
[...]
end;
[tex.web]*/

void hlist_out() { new hlist_out_(this).Invoke(); }
sealed partial class hlist_out_(TEX tex) : Ref(tex) /*output an |hlist_node| box*/
{
	scaled base_line; /*the baseline coordinate for this box*/
	[...]
	public void Invoke()
	{
		cur_g = 0; cur_glue = float_constant(0);
		[...]
	}
```

The implementation of `hlist_out` continues a bit later at line 12438 after a
few `WEB` macros.
```
/*[tex.web]
@ @d billion==float_constant(1000000000)
@d vet_glue(#)== glue_temp:=#;
	[...]

@<Move right or output leaders@>=
begin g:=glue_ptr(p); rule_wd:=width(g)-cur_g;
[...]
goto move_past;
end
[tex.web]*/

static readonly double billion = float_constant(1_000_000_000);
void vet_glue(double A, out double glue_temp)
{
	glue_temp = A;
	[...]
}
partial class hlist_out_
{
	void Move_right_or_output_leaders(pointer p)
	{
		g = glue_ptr(p); rule_wd = width(g) - cur_g;
		[...]
		Goto("move_past");
	}
}
```

## What's next?
Is it a C# version of TeX?  Well, the code really isn't very idiomatic C#;
and nobody would write a function name like `Make\_a\_partial\_...\_copied()`. 
It's almost certain that if Knuth were to write TeX today from scratch in C# (or C++), he would use the
dynamic memory facilities provided by the language, i.e., `new()`, as well as many other standardized features;
the result would look nothing like **tex.cs**.  (Although his more recent work on MMIX for
\titleref{The Art of Computer Programming} shows he still has a penchant for literate programming, so he
might never give up `CWEB`).

While retaining the original Pascal parts is very useful for comparison, once you accept the C# as
'canonical', there's little need to retain it, even if it is just comments.  Perhaps making all functions
`virtual` would be interesting?

A similar approach could probably be used for nearly any language (Rust~\cite{rust:homepage} anyone?)
that supports classes (arbitrary ordering of member names and implicit \texttt{this}) and exceptions
(for non-local `goto`s).  Even C might work if one is willing to tolerate a lot of 'noise' from
forward declarations and other such things to preserve the presentation of **tex.web**.

In any case, whatever would be gained by not using 1982 Pascal would surely be lost to PDF, Unicode,
OpenType and numerous other features that are part of today's robust TeX implementations.
Nevertheless, having a canonical TeX available in a modern development environment could enable 
experimentation, education and who knows what else?

## Try it out for yourself
Experience the thrill of running TeX directly in the debugger, no `#line` directives needed!
First, download \titleref{Visual Studio Community} from \url{visualstudio.microsoft.com/downloads/}.
Then, either C# or C++ source code from \url{github.com/TeX-82}.
