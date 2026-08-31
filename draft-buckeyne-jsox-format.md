---
title: "The JavaScript Object eXchange (JSOX) Data Interchange Format"
abbrev: "JSOX"
category: info

docname: draft-buckeyne-jsox-format-latest
submissiontype: independent
number:
date:
v: 3
keyword:
 - JSON
 - serialization
 - data interchange
 - media type
venue:
  github: "d3x0r/jsox.rfc"
  latest: "https://d3x0r.github.io/jsox.rfc/draft-buckeyne-jsox-format.html"

author:
 -
    fullname: James Buckeyne
    organization: Independent
    email: james.buckeyne@gmail.com

normative:
  RFC2119:
  RFC5234:
  RFC8174:
  RFC8259:
  RFC6838:
  ECMA-262:
    title: "ECMAScript Language Specification"
    author:
      - org: Ecma International
    date: 2024-06
    seriesinfo:
      ECMA: "Standard ECMA-262, 15th Edition"
    target: https://www.ecma-international.org/publications-and-standards/standards/ecma-262/
  UNICODE:
    title: "The Unicode Standard"
    author:
      - org: The Unicode Consortium
    target: https://www.unicode.org/versions/latest/
  ISO8601:
    title: "Date and time -- Representations for information interchange"
    author:
      - org: International Organization for Standardization
    date: 2019
    seriesinfo:
      ISO: 8601-1:2019
    target: https://www.iso.org/standard/70907.html

informative:
  RFC4627:
  RFC7493:
  RFC4648:
  IEEE754:
    title: "IEEE Standard for Floating-Point Arithmetic"
    author:
      - org: IEEE
    date: 2019
    seriesinfo:
      IEEE: "Standard 754-2019"
    target: https://standards.ieee.org/standard/754-2019.html
  JSOX-IMPL:
    title: "JSOX reference implementation"
    author:
      - name: James Buckeyne
    target: https://github.com/d3x0r/jsox
  JSON5:
    title: "JSON5 Data Interchange Format"
    target: https://json5.org/
  JSON6:
    title: "JSON6 Data Interchange Format"
    target: https://github.com/d3x0r/JSON6

--- abstract

JavaScript Object eXchange (JSOX) is a lightweight, text-based,
language-independent data interchange format.  It is derived from JSON
and from the object literal syntax of the ECMAScript Programming
Language Standard.  Every well-formed JSON text is a well-formed JSOX
text.

JSOX extends JSON with unquoted identifiers, additional string quoting
and escape forms, comments, additional number forms including dates and
arbitrary-precision integers, binary typed arrays, user-defined types
carried by a type tag, field-name macros that remove repeated keys from
a document, and references that permit shared and cyclic structures to
be encoded.

This document defines the JSOX grammar and registers the media type
"application/jsox".

--- middle

# Introduction {#intro}

JavaScript Object eXchange (JSOX) is a text format for the
serialization of structured data.  It is derived from JSON {{RFC8259}}
and from the object literal syntax of the ECMAScript Programming
Language Standard {{ECMA-262}}.

A JSOX parser accepts every text that a JSON parser accepts, and
assigns it the same meaning.  The converse does not hold: a JSOX
generator can produce texts that a JSON parser will reject.

JSOX's design goals were to extend JSON with features that improve
clarity for human authors and reduce redundancy for machine-generated
documents, while keeping JSON's original goals of being minimal,
portable, and textual.  Specifically, JSOX adds:

* representation of the ECMAScript `Date` and `BigInt` types without
  application-level conventions;
* representation of `ArrayBuffer` and the twelve `TypedArray` classes
  as base 64 data carried by a type tag;
* a type tag on objects, arrays, and strings, so that a receiver can
  revive application types directly;
* field-name macros, which declare a set of field names once and then
  carry only values for each subsequent object of that shape;
* references, which allow a value to name another value already present
  in the same text, so that shared and cyclic object graphs survive a
  round trip;
* comments, unquoted object keys, and the additional string and number
  forms of ECMAScript.

All of these features are optional.  A JSOX text that uses none of them
is a JSON text.

A JSOX value is a primitive type (string, number, boolean, date,
arbitrary-precision integer, or binary array), a structured type
(object or array), or one of the two placeholder values `null` and
`undefined`.

A string is a sequence of zero or more Unicode characters {{UNICODE}}.
Note that this citation references the latest version of Unicode rather
than a specific release.

Many strings do not need quotation marks, and JSOX lets those be
written without them.  This is not a separate kind of token: an
unquoted string is a string, and it may appear anywhere a string may
appear.  It may be an object member name, a type tag, or a value in its
own right, so `{a:b}` and `{"a":"b"}` denote the same object.  Where
this document says "identifier", it means a string in a position where
a name is expected; the rules for when quotes may be omitted are the
same in every position, and are given in {{identifiers}}.

An object is a collection of zero or more name/value pairs, where a
name is a string.  An array is an ordered sequence of zero or more
values.

The terms "object" and "array" come from the conventions of
ECMAScript.  The term "class" is used in the general sense of a set of
values that share a common structure and are distinguished from other
values by that structure.

## Relationship to Other Formats

JSOX is a superset of JSON {{RFC8259}}.  It draws its relaxations of
JSON's punctuation and its string, number, and comment syntax from
JSON5 {{JSON5}} and JSON6 {{JSON6}}, and adds the type tags,
macros, references, dates, arbitrary-precision integers, and binary
arrays described in this document.

I-JSON {{RFC7493}} defines a restricted profile of JSON for maximum
interoperability.  JSOX moves in the opposite direction, and a JSOX
text that uses any JSOX-specific construct is not an I-JSON message.

A reference implementation is available at {{JSOX-IMPL}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The grammatical rules in this document are to be interpreted as
described in ABNF {{RFC5234}}, with the extension that numeric values
above %xFF denote Unicode scalar values rather than octets, as is done
in {{RFC8259}}.

Angle-bracketed prose values are used where a rule is more clearly
stated in English than in ABNF.

The grammar describes the text after comments have been removed
({{ws}}).  Comments are therefore absent from every rule below,
including `ws`, and a comment may appear at any point that is not
inside a quoted string.

The number rules of {{numbers}} are the one place where the ABNF is
deliberately looser than the language.  A number is recognized by a
closed set of characters, and several of its forms -- dates in
particular -- are then constrained by prose and by the references
rather than by the grammar.  Where the two disagree, the prose governs.

# JSOX Grammar

A JSOX text is a sequence of tokens.  The set of tokens includes six
structural characters, identifiers, strings, numbers, and six literal
names.

A JSOX text is a single serialized value, optionally preceded by one or
more macro definitions ({{macros}}).

~~~ abnf
JSOX-text = ws *( macro-definition ws ) value ws
~~~

These are the six structural characters:

~~~ abnf
begin-array     = ws %x5B ws  ; [ left square bracket
begin-object    = ws %x7B ws  ; { left curly bracket
end-array       = ws %x5D ws  ; ] right square bracket
end-object      = ws %x7D ws  ; } right curly bracket
name-separator  = ws %x3A ws  ; : colon
value-separator = ws %x2C ws  ; , comma
~~~

Two additional rules are used in the places where a structural
character MUST NOT be preceded by whitespace: after the tag of a macro
definition ({{macros}}), and after the tag of a typed value at the top
level ({{typed}}).

~~~ abnf
open-object     = %x7B ws     ; { with no preceding whitespace
open-array      = %x5B ws     ; [ with no preceding whitespace
~~~

## Whitespace and Comments {#ws}

Insignificant whitespace is allowed before or after any of the six
structural characters.  A whitespace character explicitly ends the
preceding token; the next non-whitespace character begins a new token.

All whitespace is optional except where it is needed to separate two
tokens that would otherwise be read as one.  For example, the text
`1 2 3` is three numbers and requires the separating whitespace, while
`[1,2,3]` does not.

~~~ abnf
ws = *(
        %x09 /              ; Horizontal tab
        %x0A /              ; Line feed or New line
        %x0D /              ; Carriage return
        %x20 /              ; Space
        %x2028 /            ; U+2028 Line separator
        %x2029 /            ; U+2029 Paragraph separator
        %xFEFF )            ; U+FEFF Zero width no-break space
~~~

Note that U+00A0 NO-BREAK SPACE is deliberately not whitespace in
JSOX.  Its purpose in text is to join words that must not be separated,
and JSOX preserves that meaning rather than overriding it.  U+00A0 is therefore an ordinary
string character with one exception, in numbers; see {{nbsp}}.

U+FEFF is treated as whitespace wherever it occurs, which has the
effect that a byte order mark at the start of a text is ignored; see
{{encoding}}.

Comments are removed from the character stream before it is divided
into tokens; they are not whitespace, and they are not values.  Two
forms are defined:

~~~ abnf
comment       = line-comment / block-comment

line-comment  = ( %x2F.2F / %x23 )
                *( %x00-09 / %x0B-0C / %x0E-2027 /
                   %x202A-10FFFF )
                ; "//" or "#" up to but not including the next
                ; %x0A, %x0D, %x2028 or %x2029

block-comment = %x2F.2A <any sequence of characters that does not
                contain the two-character sequence "*/"> %x2A.2F
                ; "/*" ... "*/"
~~~

A line comment is terminated by any of the four line terminators of
{{ECMA-262}}: U+000A LINE FEED, U+000D CARRIAGE RETURN, U+2028 LINE
SEPARATOR, and U+2029 PARAGRAPH SEPARATOR.  A line comment MUST NOT be treated as
continuing past any of them.  The terminator is not part of the
comment, so it remains in the stream as whitespace once the comment
has been removed.

A line comment that is not terminated before the end of the text is an
error.  A block comment that is not closed before the end of the text
is likewise an error.

A comment is recognized only outside a quoted string.  Inside one there
is no comment: `//`, `/*`, `*/`, and `#` are
ordinary characters and are part of the string's value.  The text
`"blah /* whatever */ "` is a single string of twenty characters,
and because a string may span lines ({{strings}}), the same is true of
comment delimiters that appear on a later line of a multi-line string.

Because a comment is removed rather than replaced by a separator, a
comment does not divide the text around it.  `[ab/*x*/cd]` is the
single string `abcd`, and `[tr/*c*/ue]` is the boolean `true`, exactly
as if the comment had not been written.

Numbers are the exception, and not because of the comment.  A number
is a whitelist: it is made only of the characters given in
{{numbers}}, all of which are below U+0080.  `/` is not among them, so
a number ends at the `/` that opens the comment, and the digits after
the comment begin a second number.  `0.123/*or*/456` is therefore two
numbers and never `0.123456`, and inside an array it is the adjacency
error of {{adjacency}}.  An unquoted string is the opposite kind of
rule -- anything not excluded -- which is why its two halves rejoin
and a number's do not.

Because `#` always introduces a comment, a generator MUST quote any
string containing one.  `/` introduces a comment only as `//` or `/*`,
so a generator MUST quote a string in which a `/` is followed by `/`
or `*`, and need not quote one otherwise; see {{identifiers}}.

## Strings, Identifiers, and Quoting {#identifiers}

A string may be written with or without quotation marks.  A string
written without them is called an unquoted string, and the ABNF rule
`unquoted-string` below describes which strings may be written that
way.

There is no separate identifier type.  Wherever this document writes
`identifier` -- for an object member name ({{objects}}) or a type tag
({{typed}}) -- the token is a string, and it may be written quoted or
unquoted under exactly the rules given here.  The word "identifier" is
retained only because it reads more naturally in those positions.

~~~ abnf
string          = quoted-string / unquoted-string

identifier      = string

unquoted-string = id-begin *id-char

id-begin = <any id-char that is not DIGIT, "+", "-", or ".">

id-char  = <any Unicode scalar value that is not matched by ws, is
            not a structural character, is not a quotation-mark,
            and is not "#"; and "/" only where the character that
            follows it is neither "/" nor "*">
~~~

`.` is excluded from `id-begin` but not from `id-char`, so it may
appear anywhere in an unquoted string except at the start.
`this.is.a.test` and `M.C.Fields` are each a single unquoted string,
while `.5` is a number ({{numbers}}).  The same holds for a member
name: `{this.is.a.test:1}` needs no quotes.

`#` is excluded everywhere, because it always introduces a comment
({{ws}}).  `/` introduces one only as `//` or `/*`, so a solitary
solidus is an ordinary character and needs no quotes:
`www.example.com/file.name` is one unquoted string.

A `/` that is followed by `/` or `*` opens a comment and so ends the
string, which is why a URL carrying a scheme still MUST be quoted:
in `"http://example.com"` the `//` would open a comment, and the `:`
is a structural character in any case.

Nothing else is reserved.  A reverse solidus, for instance, is an
ordinary `id-char`, so `\` on its own is a valid unquoted string; it
carries no escape meaning outside a quoted string ({{strings}}).

`quoted-string` is defined in {{strings}}.

Note that an unquoted string is a value in its own right, not only a
member name.  In `{a:b}` both `a` and `b` are strings, and the object
is the same one denoted by the JSON text `{"a":"b"}`.  Likewise
`[abc]` is an array of one string, and a JSOX text consisting of the
single token `hello` is a string.

A token that matches one of the six literal names ({{values}}) is that
literal and not a string.  To write one of those words as a string, it
MUST be quoted: `true` is the boolean, and `"true"` is the string.
Similarly, a token that begins with a digit, `+`, `-`, or `.` is a
number, which is why `id-begin` excludes those characters.

Quotes are OPTIONAL around a string that matches `unquoted-string`.
Generators MUST quote any string that does not.

Generators SHOULD additionally quote a string that:

* is one of the literal names `true`, `false`, `null`, `NaN`,
  `Infinity`, or `undefined` -- here quoting is REQUIRED, per the
  paragraph above; or
* contains any of the characters matched by the regular expression
  `/[\n\r\t {}()<>!+*/.:,-]/`.

These characters are not all reserved by this document, but quoting
them avoids ambiguity with syntax that a future revision may define.
One of them, `#`, is not merely advisory: it is not an `id-char` at
all, so the MUST above already requires quoting any string containing
one.  A `/` is required to be quoted only where it is followed by `/`
or `*`; elsewhere quoting it is the advice of this list, not a rule.

### U+00A0 in an Unquoted String {#nbsp}

U+00A0 NO-BREAK SPACE is an ordinary `id-char`, and this is deliberate.
Its purpose in text is to join words that must not be separated, and
JSOX preserves that meaning rather than overriding it: `th<NBSP>ng` is
one string of five characters, not two strings.  Once a token is known
to be a string, U+00A0 is collected into it like any other character.

A generator is NOT REQUIRED to quote a string containing U+00A0, and
the reference implementation {{JSOX-IMPL}} does not.  Because U+00A0 is
an `id-char`, such a string round-trips unquoted without ambiguity.

The consequence for a leading U+00A0 follows from the same rule.  It is
an `id-begin` character, so it starts a string: `<NBSP>thing` is one
six-character string, while `<NBSP> thing` is two strings, the first of
which is one character long, because the space between them is
whitespace and U+00A0 is not.

U+00A0 does, however, terminate a number ({{numbers}}).  Numbers have
their own non-breaking separator, `_`, and a number's characters are
all below U+0080.

### Adjacent Values {#adjacency}

Whether two adjacent values may appear without a separator depends on
whether the surrounding context bounds them.

The interior of an object or an array is a *bounded* context: the
closing `}` or `]` and the `,` and `:` separators mark where each value
ends, so no token is ambiguous.  The top level of a text is
*unbounded*: nothing follows a complete value except, in a stream
({{streams}}), the next value.

In a bounded context, two adjacent values with only whitespace between
them are a typed value if the first is a string, and an error
otherwise.  `[ab cd]` is the string `cd` tagged `ab` ({{typed}}), while
`[1 2]` is an error, since a number is never a tag.

In an unbounded context, a complete value ends there.  `"abc" {a:1}` at
the top level is two values, not one tagged value; in a single-value
text the second is extra data, and in a stream it is the next item.

# Values {#values}

A JSOX value MUST be an object, an array, a number, a string, a typed
value, or one of the following six literal names:

~~~
false null true Infinity NaN undefined
~~~

The literal names MUST be lowercase except as spelled here, and MUST
match exactly.  No other literal names are allowed.

~~~ abnf
value = false / null / true / undefined / NaN / Infinity
      / number / string / object / array / typed-value

false     = %x66.61.6c.73.65             ; false
null      = %x6e.75.6c.6c                ; null
true      = %x74.72.75.65                ; true
undefined = %x75.6e.64.65.66.69.6e.65.64 ; undefined
NaN       = [ minus / plus ] %x4e.61.4e  ; [-]NaN
Infinity  = [ minus / plus ]
            %x49.6e.66.69.6e.69.74.79    ; [-]Infinity
~~~

The alternatives of `value` are ordered: a token is tested against the
six literal names first, then as a number, and only then as a string.
Because `string` includes `unquoted-string` ({{identifiers}}), a bare
word such as `hello` is a value -- a string -- and not an error.

`-NaN` and `+NaN` are accepted and denote the same value as `NaN`.

`undefined` denotes the absence of a value.  A member whose value is
`undefined` is equivalent to a member that is not present, and a
generator MAY omit such a member.  Receivers that have no
representation for `undefined` SHOULD treat it as `null`.

# Objects {#objects}

An object structure is represented as a pair of curly brackets
surrounding zero or more name/value pairs (or members).  A name is an
identifier.  A single colon comes after each name, separating the name
from the value.  A single comma separates a value from a following
name.  The names within an object SHOULD be unique.

~~~ abnf
object = begin-object
         [ member *( value-separator member )
           [ value-separator ] ]
         end-object

member = identifier name-separator value
~~~

A single trailing comma is permitted after the last member and has no
effect.  More than one consecutive comma between members is an error.

An object whose names are all unique is interoperable in the sense that
all software implementations receiving that object will agree on the
name-value mappings.  When the names within an object are not unique,
the behavior of software that receives such an object is unpredictable.
Many implementations report the last name/value pair only.  Other
implementations report an error or fail to parse the object, and some
implementations report all of the name/value pairs, including
duplicates.

JSOX parsing libraries MAY differ as to whether or not they make the
ordering of object members visible to calling software.
Implementations whose behavior does not depend on member ordering will
be interoperable in the sense that they will not be affected by these
differences.

# Arrays {#arrays}

An array structure is represented as square brackets surrounding zero
or more values (or elements).  Elements are separated by commas.

~~~ abnf
array   = begin-array
          [ element *( value-separator element ) ]
          end-array

element = ws [ value ] ws
~~~

There is no requirement that the values in an array be of the same
type.

An element MAY be empty, in which case the array has a member at that
position whose value is `undefined`.  Consecutive commas therefore
produce elisions: `[1,,3]` is an array of three elements whose second
element is `undefined`.  A single trailing comma does not add an
element; `[1,2,]` and `[1,2]` denote the same array, while `[1,2,,]`
denotes an array of three elements.

# Numbers {#numbers}

The representation of numbers is similar to that used in most
programming languages, and follows {{ECMA-262}}.  A number is
represented in base 10 unless a radix prefix selects another base.  It
contains an integer component that may be prefixed with an optional
sign, which may be followed by a fraction part and/or an exponent part.

Four further forms are recognized that JSON does not have: radix
numbers, arbitrary-precision integers, dates ({{dates}}), and the
non-finite literals `NaN` and `Infinity` ({{values}}).

~~~ abnf
number        = plain-number / radix-number / bigint / date

plain-number  = [ sign ] ( int [ frac ] / frac ) [ exp ]

radix-number  = [ sign ] zero radix 1*radix-digit

bigint        = [ sign ] int bigint-suffix

sign          = plus / minus

int           = DIGIT *DIGIT

frac          = decimal-point *DIGIT

exp           = e [ sign ] 1*DIGIT

decimal-point = %x2E                ; .

e             = %x65 / %x45         ; e E

minus         = %x2D                ; -

plus          = %x2B                ; +

zero          = %x30                ; 0

radix         = %x58 / %x78         ; X x  -- hexadecimal
              / %x4F / %x6F         ; O o  -- octal
              / %x42 / %x62         ; B b  -- binary

radix-digit   = DIGIT / %x41-46 / %x61-66
                ; restricted to the digits of the selected radix

bigint-suffix = %x6E                ; n

underscore    = %x5F                ; _
~~~

A number begins with a digit, `+`, `-`, or `.`.  Once a number has
started, only the characters named by the rules above may follow, and
all of them are below U+0080.

A fraction part is a decimal point followed by zero or more digits, so
both a leading and a trailing decimal point are permitted: `.5` and
`1.` are valid numbers denoting 0.5 and 1.

An exponent part begins with the letter E in upper or lower case, which
may be followed by a plus or minus sign.  The E and optional sign are
followed by one or more digits.

Unlike JSON, leading zeros are permitted in a decimal number and do not
change its value: `0123` denotes 123.  A base other than 10 MUST be
selected with an explicit radix prefix.

Earlier descriptions of JSOX followed C in reading a leading zero as
selecting octal, so that `0123` denoted 83.  That reading is
deprecated.  Implementations MUST NOT interpret a leading zero as
selecting octal, and MUST use the `0o` prefix to write an octal
number.  Generators SHOULD avoid emitting a leading zero at all, since
a reader familiar with C may misread it.

Only one sign may appear, and it MUST be the first character.  A
sequence such as `----123` is an error.

An underscore (`_`) MAY appear anywhere within a number, including
before or after the decimal point and at the end.  It is treated as a
zero-width separator: it is not stored and does not modify the value.
`1_000_000` and `1000000` denote the same number.  Note that a leading
underscore does not begin a number; it begins a string.

`_` is the only non-breaking separator a number has.  U+00A0 NO-BREAK
SPACE, which joins characters into a single string elsewhere
({{nbsp}}), terminates a number instead: every character of a number is
below U+0080, so U+00A0 cannot be part of one.  `1234<NBSP>` is the
number 1234, and `12<NBSP>34` is two numbers -- allowed at the top
level of a stream, and an error in a bounded context ({{adjacency}}).

This specification allows implementations to set limits on the range
and precision of numbers accepted.  Since software that implements IEEE
754 binary64 (double precision) numbers {{IEEE754}} is generally
available and widely used, good interoperability can be achieved by
implementations that expect no more precision or range than these
provide, in the sense that implementations will approximate JSOX
numbers within the expected precision.  A JSOX number such as `1E400`
or `3.141592653589793238462643383279` may indicate potential
interoperability problems, since it suggests that the software that
created it expects receiving software to have greater capabilities for
numeric magnitude and precision than is widely available.

Note that when such software is used, numbers that are integers and are
in the range `[-(2**53)+1, (2**53)-1]` are interoperable in the
sense that implementations will agree exactly on their numeric values.

## Arbitrary-Precision Integers

An integer suffixed with `n` is an arbitrary-precision integer,
corresponding to the ECMAScript BigInt type {{ECMA-262}}.  Its value is
exact and is not subject to the binary64 range and precision
considerations above.

A receiver that has no arbitrary-precision integer type SHOULD report
an error rather than silently approximating the value, unless the
application has determined that approximation is acceptable.

The suffix MUST NOT be combined with a fraction part or an exponent
part.

## Dates {#dates}

A number that contains one of the characters `T`, `Z`, or `:`, or that
contains `+` or `-` in a position other than the leading sign, is a
date rather than an ordinary number.

~~~ abnf
date        = [ sign ] 1*DIGIT time-symbol
              *( DIGIT / time-symbol )

time-symbol = %x54 / %x5A           ; T Z
            / %x2B / %x2D / %x3A    ; + - :
~~~

The value of a date is determined by interpreting its characters as an
ISO 8601 {{ISO8601}} date and time, as profiled by the Date Time String
Format of {{ECMA-262}}.  A date is written without quotation marks.

Generators SHOULD emit a date that carries as much of the original
timestamp as is available, including the local time zone offset, so
that a receiver can recover it.  A date that is not a valid ISO 8601
date and time is an error; implementations that follow {{ECMA-262}}
will produce an "Invalid Date" value for it.

The lexical rule above is deliberately permissive; it identifies which
token is a date, and {{ISO8601}} determines whether that token is
valid.  Generators MUST NOT rely on any interpretation outside
{{ISO8601}}.

# Quoted Strings {#strings}

This section defines the quoted form of a string.  The unquoted form,
and the relationship between the two, are defined in
{{identifiers}}.

The representation of quoted strings follows the conventions of
{{ECMA-262}}.  A quoted string begins and ends with a quotation mark.
Three quotation marks are available: U+0022 QUOTATION MARK, U+0027
APOSTROPHE, and U+0060 GRAVE ACCENT.  The closing quotation mark MUST be the same character as
the opening quotation mark.  No template substitution is performed
inside a grave-accent-quoted string; the grave accent is a third
quotation mark and nothing more.

All Unicode characters may be placed within the quotation marks, except
for the quotation mark used to open the string and the reverse solidus,
which MUST be escaped.  The other two quotation marks need not be
escaped.

A string MAY span more than one line.  Every character between the
opening and the closing quotation mark is retained, including line
terminators.  To continue a string onto the next line without including
the line terminator in its value, escape the line terminator with a
reverse solidus.

~~~ abnf
quoted-string   = quotation-mark *char quotation-mark
                  ; the closing mark is the same character as
                  ; the opening mark

quotation-mark  = %x22 / %x27 / %x60   ; " ' `

char            = unescaped / escape escape-sequence

escape          = %x5C                 ; \

escape-sequence = %x22 /               ; "  quotation mark
                  %x27 /               ; '  quotation mark
                  %x60 /               ; `  quotation mark
                  %x5C /               ; \  reverse solidus
                  %x2F /               ; /  solidus
                  %x62 /               ; b  backspace  U+0008
                  %x66 /               ; f  form feed  U+000C
                  %x6E /               ; n  line feed  U+000A
                  %x72 /               ; r  carriage return
                  %x74 /               ; t  tab        U+0009
                  %x76 /               ; v  vert. tab  U+000B
                  %x78 2HEXDIG /       ; xXX
                  %x30-32 2OCTDIG /    ; 0NN  octal, U+0000-U+00FF
                  %x75 4HEXDIG /       ; uXXXX
                  %x75 %x7B 1*HEXDIG %x7D /
                                       ; u{XXXXXX}
                  %x0A / %x0D /        ; line continuation
                  %x2028 / %x2029

unescaped       = <any Unicode scalar value other than the
                   quotation mark that opened this string and
                   %x5C reverse solidus>

OCTDIG          = %x30-37              ; 0-7
~~~

Any character may be escaped.  If the character is in the Basic
Multilingual Plane (U+0000 through U+FFFF), then it may be represented
as a six-character sequence: a reverse solidus, followed by the
lowercase letter u, followed by four hexadecimal digits that encode the
character's code point.  The hexadecimal letters A through F can be
upper or lower case.

A character whose code point is below U+0100 may also be written as
`\x` followed by two hexadecimal digits, or as `\0` followed by two
octal digits.

A character at any code point may be written as `\u{` followed by one
or more hexadecimal digits and `}`.

Alternatively, there are two-character sequence escape representations
of some popular characters.  So, for example, a string containing only
a single reverse solidus character may be written as the four
characters `"\\"`.

To escape an extended character that is not in the Basic Multilingual
Plane using the four-digit form, the character is represented as a
12-character sequence encoding the UTF-16 surrogate pair.  So, for
example, a string containing only the G clef character (U+1D11E) may be
represented as `"𝄞"`, or equivalently as `"\u{1D11E}"`.

An escape sequence that is not listed above is an error.  Note in
particular that a reverse solidus followed by a decimal digit other
than `0`, `1`, or `2` is reserved and MUST NOT be generated.

# Typed Values {#typed}

A typed value is a value that carries a type tag: an identifier that
names an application type, immediately followed by the value's
representation.  A receiver that recognizes the tag may revive the
value as an instance of that type; a receiver that does not recognize
the tag MUST still be able to parse the value, and SHOULD make the tag
and the underlying value available to the application.

~~~ abnf
typed-value  = typed-object / typed-array / typed-string
             / macro-reference

typed-object = identifier begin-object
               [ member *( value-separator member )
                 [ value-separator ] ]
               end-object

typed-array  = identifier begin-array
               [ element *( value-separator element ) ]
               end-array

typed-string = identifier ws quoted-string
~~~

The `ws` these three rules admit before the value is permitted only in
a bounded context.  At the top level it MUST be absent, so that
`open-object` and `open-array` apply and a typed string's quotation
mark is adjacent to its tag.  See below.

Whether whitespace may separate the tag from the value it carries
depends on the surrounding context, as described in {{adjacency}}.

In a bounded context -- inside an object or an array -- whitespace MAY
appear between the tag and the `{`, `[`, or quotation mark that follows
it.  The enclosing `}` or `]` and the `,` and `:` separators already
mark where each value ends, so nothing is made ambiguous by the gap.
`[ab"cd"]` and `[ab "cd"]` both denote an array of one string, `cd`,
tagged `ab`.

A comment between the two does not arise as a separate case: it is
removed before tokenizing ({{ws}}), so `[ab/*x*/"cd"]` is `[ab"cd"]`
and the tag is adjacent after all.  This is why the restriction below
can be stated in terms of whitespace alone.

In an unbounded context -- the top level of a text or of a stream --
the tag MUST be adjacent to its value, with no whitespace
between them.  Nothing there marks where a value ends, so a complete
value ends at its last character: `"abc" {a:1}` is two values, and
`"abc"{a:1}` is one tagged value.

A generator SHOULD NOT emit whitespace between a tag and its value in
either context, since output written that way is valid in only one of
them.

A tag is a string ({{identifiers}}), so it may be quoted.  Quoting is
what allows a tag to contain a character that `unquoted-string` does
not admit; `"o:1"{a:1}` is a typed object whose tag is the three-
character name `o:1`.  A quoted tag names an application type in
exactly the same way an unquoted one does, and carries no other
meaning: the colon in `o:1` is part of the name, and does not make the
tag a name/value pair.

A typed string is exactly two strings: the tag and the value.  A third
adjacent string matches no rule and is an error.  `"a""b"` is the
string `b` tagged `a` -- the same construct as `color"#333"`, with the
tag quoted -- while `"a""b""c"` is not a JSOX text.

Because a quoted string is self-delimiting, a typed string at the top
level of a text MUST use an unquoted tag; otherwise the tag would be
read as a complete string value in its own right.  This restriction
applies only to typed strings, since `{` and `[` are unambiguous after
a closing quotation mark.

A tag that the receiver has not registered does not make the text
invalid.  The value is parsed normally and the tag is reported to the
application; see {{security}}.

## Typed Arrays and Binary Data {#typed-arrays}

The following tags are reserved and denote binary data rather than an
application type.  The content of such an array is a single base 64
token ({{base64}}), written without quotation marks, that encodes the
octets of the array.

| Tag   | Type                                          |
|-------|-----------------------------------------------|
| `ab`  | untyped octet buffer (ArrayBuffer)            |
| `u8`  | unsigned 8-bit integers                       |
| `cu8` | unsigned 8-bit integers, clamped              |
| `s8`  | signed 8-bit integers                         |
| `u16` | unsigned 16-bit integers                      |
| `s16` | signed 16-bit integers                        |
| `u32` | unsigned 32-bit integers                      |
| `s32` | signed 32-bit integers                        |
| `f32` | IEEE 754 binary32 numbers                     |
| `f64` | IEEE 754 binary64 numbers                     |
{: #tab-array-types title="Binary typed-array tags"}

The tags `u64` and `s64` are reserved for unsigned and signed 64-bit
integer arrays.  Generators MUST NOT emit them until their
representation is defined.

For all tags other than `ab`, the encoded octets are the little-endian
in-memory representation of the array's elements, and the number of
octets MUST be a multiple of the element size.

~~~ abnf
binary-array = array-type open-array [ base64-data ] end-array

array-type   = "ab" / "u8" / "cu8" / "s8" / "u16" / "s16"
             / "u32" / "s32" / "f32" / "f64"
~~~

These ten tags MUST NOT be used by an application for any other
purpose.

## References {#references}

The reserved tag `ref` denotes a reference to a value that appears
elsewhere in the same text.  Its content is the path from the root of
the text to that value: a sequence of member names and array indices.

~~~ abnf
reference = "ref" open-array
            [ ref-step *( value-separator ref-step ) ]
            end-array

ref-step  = string / int
~~~

A string step selects an object member by name; an integer step selects
an array element by position, counting from zero.  An empty path
denotes the root value of the text.

A reference is not a value in its own right: it is replaced, during
parsing, by the value that its path selects.  A `ref` array MUST NOT be
used where the referenced value has not yet been fully determined by
the text, except that a reference MAY point to an enclosing structure,
which is how cyclic structures are encoded.

References are how JSOX preserves object identity.  Two members whose
values are the same object are encoded once, with the second member
carrying a `ref` to the first.

Because a reference names a position in the text, a receiver MUST
resolve references against the text as received, and MUST NOT resolve
a path that leaves the text.  See {{security}}.

## Macros {#macros}

A macro declares a list of field names once, so that objects of that
shape can afterwards be written as a list of values.  This removes
repeated member names from a document; it does not change the value
that the document denotes.

~~~ abnf
macro-definition = identifier open-object
                   identifier *( value-separator identifier )
                   end-object

macro-reference  = identifier open-object
                   [ element *( value-separator element ) ]
                   end-object
~~~

A macro definition is distinguished from a typed object ({{typed}}) by
what follows the first name inside the braces: if a colon appears
before the first comma, the construct is a typed object; otherwise it
is a macro definition.

Both halves of that test matter at the top level, where either
construct may appear.  `x{a}` declares a macro with the single field
name `a`; it is a definition, not a value, so a text containing only
that is a text with no value and is incomplete.  `x{a:1}` has a colon
before the first comma and is therefore a typed object, which is a
value.  The same applies to a quoted tag: `"o:1"{a}` is a macro
definition tagged `o:1`, and `"o:1"{a:1}` is a typed object.

A macro definition MUST declare at least one name.  Empty braces
therefore never form a macro definition: `x{}` is a typed object with
no members, and it is a value -- an empty object carrying the tag `x`,
which a receiver that recognizes `x` may revive as a default-
constructed instance of that type.  A zero-name macro could declare
nothing and could never be referenced, so resolving the ambiguity the
other way would make `x{}` useless rather than merely empty.

A macro definition MUST appear at the top level of the text, before the
value that uses it.  It is not itself a value.

A macro reference produces an object whose members are the declared
names paired, in order, with the values supplied.  If fewer values are
supplied than names were declared, the remaining members are absent, as
if their value were `undefined`.  Supplying more values than names were
declared is an error.

A macro tag and a type tag MAY share a name.  When they do, objects
produced from the macro are also revived as instances of that type.

A receiver MUST reject a macro reference whose tag has not been
defined earlier in the same text.

# String and Character Issues

## Character Encoding {#encoding}

JSOX text exchanged between systems that are not part of a closed
ecosystem MUST be encoded using UTF-8 {{RFC8259}}.  Implementations MAY
additionally support UTF-16 and UTF-32; texts encoded in UTF-8 are
interoperable in the sense that they will be read successfully by the
maximum number of implementations.

Implementations MUST NOT add a byte order mark to the beginning of a
JSOX text.  Implementations that parse JSOX texts MUST ignore a byte
order mark rather than treating it as an error, because U+FEFF is
whitespace in JSOX ({{ws}}).

## Unicode Characters

When all the strings represented in a JSOX text are composed entirely
of Unicode characters {{UNICODE}} (however escaped), then that JSOX
text is interoperable in the sense that all software implementations
that parse it will agree on the contents of names and of string values
in objects and arrays.

However, the ABNF in this specification allows member names and string
values to contain bit sequences that cannot encode Unicode characters;
for example, `"\uDEAD"` (a single unpaired UTF-16 surrogate).
Instances of this have been observed, for example, when a library
truncates a UTF-16 string without checking whether the truncation split
a surrogate pair.  The behavior of software that receives JSOX texts
containing such values is unpredictable; for example, implementations
might return different values for the length of a string value or even
suffer fatal runtime exceptions.

## String Comparison

Software implementations are typically required to test names of object
members for equality.  Implementations that transform the textual
representation into sequences of Unicode code units and then perform
the comparison numerically, code unit by code unit, are interoperable
in the sense that implementations will agree in all cases on equality
or inequality of two strings.  For example, implementations that
compare strings with escaped characters unconverted may incorrectly
find that two spellings of the same string are not equal.

Note that this applies to unquoted identifiers as well as to quoted
strings, and that an identifier containing U+00A0 is not equal to the
visually identical identifier containing U+0020 ({{ws}}).

# Base 64 Encoding {#base64}

The binary typed arrays of {{typed-arrays}} carry their octets as base
64 data.  JSOX uses the alphabet below rather than the alphabets of
{{RFC4648}}, because the two characters it selects for values 62 and 63
are permitted in an unquoted identifier and so require no quoting:

~~~
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789$_
~~~

Generators MUST use this alphabet.

For interoperability with data produced by other base 64
implementations, parsers MUST additionally accept the following
characters on input:

| Character | Value                    |
|-----------|--------------------------|
| `$`       | 62                       |
| `+`       | 62                       |
| `-`       | 62                       |
| `.`       | 62                       |
| `_`       | 63                       |
| `/`       | 63                       |
| `,`       | 63                       |
| `=`       | end of data              |
| `~`       | end of data              |
{: #tab-base64 title="Base 64 characters accepted on input"}

Note that `+`, `-`, `.`, `/`, and `,` require quoting when they appear
in a value that is otherwise an unquoted identifier, and that `,` and
`/` are ambiguous inside an array; a parser reading base 64 data
between `[` and `]` reads to the closing bracket and so is not affected,
but a generator MUST NOT emit these characters.

Padding is not required.  A `=` or `~` character terminates the data.
Trailing bits that do not complete an octet MUST be zero and MUST be
discarded.

# Parsers

A JSOX parser transforms a JSOX text into another representation.  A
JSOX parser MUST accept all texts that conform to the JSOX grammar.  A
JSOX parser MAY accept non-JSOX forms or extensions.

An implementation may set limits on the size of texts that it accepts,
on the maximum depth of nesting, on the range and precision of numbers,
on the length and character contents of strings, and on the number of
macro and type tags it will track.  See {{security}}.

## Streams {#streams}

A JSOX text is a single value, but a sequence of JSOX texts may be
concatenated to form a stream, and an implementation MAY provide an
interface that reports each value as it completes.

Within a stream, whitespace between values is significant wherever the
two values would otherwise be read as one token.  The stream `1 2 3`
is three numbers; without the separating whitespace it would be one.
Values that end with a structural character or a quotation mark are
self-delimiting and need no separator.

A number, an unquoted identifier, or a literal name at the end of the
available input cannot be known to be complete, because more characters
may follow in a later portion of the stream.  An implementation MUST
NOT report such a value until either a delimiter has been seen or the
application has declared the input to be complete.

Macro and type definitions are scoped to the stream, not to the
individual value: a definition made in one value of a stream remains in
effect for later values.

# Generators

A JSOX generator produces JSOX text.  The resulting text MUST strictly
conform to the JSOX grammar.

A generator that is asked to produce a value that has no JSOX
representation, such as a function, MUST either omit it or report an
error, and MUST NOT emit an approximation that would parse as a
different value.

# Security Considerations {#security}

Generally, there are security issues with scripting languages.  JSOX is
a subset of ECMAScript that excludes code expressions, so a JSOX text
cannot by itself cause code to run.

However, JSOX's syntax deviates from ECMAScript for type tags, macro
definitions, and references.  It is therefore not generally possible to
use a scripting language's `eval()` function to parse a JSOX text, and
implementations MUST NOT do so.  A subset of JSOX texts -- those that
use no type tag, macro, reference, date, or binary array -- would be
accepted by `eval()`, but an implementation cannot know that a text is
in that subset without first parsing it.  Using `eval()` on data from
an untrusted source allows arbitrary code execution.

Reviving application types from a type tag is a deserialization
primitive, and carries the risks common to such primitives.  A receiver
MUST NOT construct an arbitrary type named by an incoming tag.  It
SHOULD revive only types that the application has explicitly
registered, and it MUST treat the values supplied for a revived type as
untrusted input, validating them exactly as it would validate the same
values arriving in a plain object.  A tag that has not been registered
MUST NOT cause any application code to run.

References ({{references}}) let a text name values elsewhere in itself.
Two hazards follow.  First, a reference can create a cyclic structure;
software that walks a parsed JSOX value without tracking which objects
it has already visited will not terminate.  Receivers that pass parsed
values to code expecting a tree -- including code that re-serializes
them -- MUST detect cycles.  Second, a small text can name the same
large substructure many times, so the size of the parsed value is not
bounded by the size of the text.  Implementations SHOULD bound the
number of references they will resolve.

Macros ({{macros}}) and type tags accumulate state for the lifetime of
a stream.  An attacker who can write to a long-lived stream can define
tags without limit.  Implementations SHOULD bound the number of
definitions they will retain.

Binary typed arrays ({{typed-arrays}}) allow a text to request an
allocation whose size is proportional to the length of the base 64
data.  This is a bounded expansion, but implementations SHOULD still
apply a limit on allocation size.

Arbitrary-precision integers are unbounded in length.  Implementations
SHOULD limit the length of such a number, since arithmetic on very
large integers can consume disproportionate time.

Because JSOX permits comments and several equivalent spellings of the
same value -- three quotation marks, several escape forms, several
number bases, underscores in numbers, macros -- two texts that denote
the same value may differ.  JSOX is therefore not suitable as-is for
applications that require a canonical form, such as computing a
signature over serialized data.  Such applications MUST define their
own canonicalization, or sign the octets received rather than a
re-serialization of them.

Implementations SHOULD apply limits on nesting depth, text size, string
length, and number magnitude.  A parser that recurses per level of
nesting can be made to exhaust its stack by a text consisting only of
open brackets.

A stream of JSON or JSOX texts may generate the sequence `][` where two
adjacent array values meet.  Software that scans for such sequences
must not treat them as significant.

# IANA Considerations {#iana}

IANA is requested to register the media type "application/jsox" in the
"Media Types" registry, using the template below and following the
procedures of {{RFC6838}}.

Type name:
: application

Subtype name:
: jsox

Required parameters:
: N/A

Optional parameters:
: N/A.  No "charset" parameter is defined for this registration.
  Adding one has no effect on compliant recipients; see
  {{encoding}}.

Encoding considerations:
: binary

Security considerations:
: See {{security}} of RFC XXXX.

Interoperability considerations:
: See {{intro}}, {{numbers}}, and {{encoding}} of RFC XXXX.

Published specification:
: RFC XXXX

Applications that use this media type:
: JSOX has been used to exchange data between applications written in
  C, C++, and JavaScript.

Fragment identifier considerations:
: N/A

Additional information:
: Deprecated alias names for this type: N/A
: Magic number(s): N/A
: File extension(s): .jsox
: Macintosh file type code(s): TEXT

Person & email address to contact for further information:
: James Buckeyne <james.buckeyne@gmail.com>

Intended usage:
: COMMON

Restrictions on usage:
: None

Author:
: James Buckeyne <james.buckeyne@gmail.com>

Change controller:
: James Buckeyne <james.buckeyne@gmail.com>

RFC Editor: please replace XXXX above with the RFC number assigned to
this document, and remove this paragraph.

# Examples {#examples}

This is a JSOX object, and also a JSON object:

~~~ json
{
  "Image": {
    "Width":  800,
    "Height": 600,
    "Title":  "View from 15th Floor",
    "Thumbnail": {
      "Url":    "http://www.example.com/image/481989943",
      "Height": 125,
      "Width":  100
    },
    "Animated": false,
    "IDs": [116, 943, 234, 38793]
  }
}
~~~

Its Image member is an object whose Thumbnail member is an object and
whose IDs member is an array of numbers.

This is a JSOX array containing two objects.  It uses unquoted member
names, dates, arbitrary-precision integers, binary arrays, and both
comment forms:

~~~
[
  {
     "precision": "zip",
     "ident":     123594985n,
     "Latitude":  37.7668,
     "Longitude": -122.3959,
     created:     2018-09-11T03:43:53.345-07:00,
     binary:      u8[U2VjcmV0],   /* Secret */
     Address:     "",
     City:        "SAN FRANCISCO",
     State:       "CA",
     Zip:         "94107",
     Country:     "US"
  },
  {
     precision:   "zip",
     "ident":     123594986n,
     Latitude:    37.371991,
     "Longitude": -122.026020,
     created:     2018-09-11T10:43:52.437Z,
     "binary":    u8[SGVsbG8sIFdvcmxkIQ],   // Hello, World!
     "Address":   "",
     City:        "SUNNYVALE",
     "State":     "CA",
     Zip:         "94085",
     "Country":   "US"
  }
]
~~~

The same data, using a macro to declare the member names once:

~~~
locale{ precision,ident,Latitude,Longitude,
        created,binary,Address,City,State,Zip,Country}
[ locale{ "zip",123594985n,37.7668,-122.3959,
          2018-09-11T03:43:53.345-07:00,
          u8[U2VjcmV0],
          "","SAN FRANCISCO","CA","94107","US"},
  locale{ "zip",123594986n,37.371991,-122.026020,
          2018-09-11T10:43:52.437Z,
          u8[SGVsbG8sIFdvcmxkIQ],
          "","SUNNYVALE","CA","94085","US"}
]
~~~

Once defined, the macro remains in effect for later values in the same
stream:

~~~
locale{ "zip",123594988n,36.1699,-115.1398,
        2018-09-11T13:23:23.636Z,
        u8[SGVsbG8sIFdvcmxkIQ],
        "","LAS VEGAS","NV","89109","US"}
~~~

This is an example of a reference.  The manager member and the first
element of the employees array denote the same object:

~~~
{
   company: { name: "Example.com",
              employees: [ { name:"bob" }, { name:"tom" } ],
              manager: ref["company","employees",0]
            }
}
~~~

Here are several small JSOX texts containing only values:

~~~
"Hello world!"
42
123n
true
Infinity
2018-09-11T13:23:23.636Z
~~~

--- back

# Changes from JSON {#changes}
{:numbered="false"}

This appendix summarizes, without normative force, how JSOX differs
from JSON {{RFC8259}}.  Every one of these constructs is optional; a
JSOX text that uses none of them is a JSON text.

Additions taken from JSON5 {{JSON5}} and JSON6 {{JSON6}}:

* Object member names may be unquoted ({{identifiers}}).
* Strings may be quoted with `'` or `` ` `` as well as `"`, may span
  lines, and accept the full set of ECMAScript escapes ({{strings}}).
* Numbers may be hexadecimal, octal, or binary; may have leading zeros,
  a leading or trailing decimal point, a leading plus sign, or
  underscore separators; and may be `NaN` or `Infinity`
  ({{numbers}}).
* Objects and arrays may have a trailing comma, and arrays may have
  elisions ({{objects}}, {{arrays}}).
* `undefined` is a literal name ({{values}}).
* Line and block comments are removed before tokenizing ({{ws}}).

Additions specific to JSOX:

* Dates are written unquoted, as a form of number ({{dates}}).
* Arbitrary-precision integers are written with an `n` suffix
  ({{numbers}}).
* Binary buffers and typed arrays are written as base 64 data carried
  by a reserved tag ({{typed-arrays}}).
* Objects, arrays, and strings may carry a type tag so that a receiver
  can revive an application type ({{typed}}).
* Macros declare a set of member names once and then carry only values
  ({{macros}}).
* References allow shared and cyclic structures to be encoded
  ({{references}}).
* `#` introduces a line comment ({{ws}}).
* A string may be unquoted in value position as well as in a member
  name ({{identifiers}}).

One earlier JSOX behavior has been withdrawn: a leading zero once
selected octal, as it does in C.  That reading is deprecated, and a
leading zero now denotes a decimal number.  Octal is written with the
`0o` prefix ({{numbers}}).

# Acknowledgments
{:numbered="false"}

RFC 4627 {{RFC4627}} was written by Douglas Crockford, and was revised
as RFC 7159 and then RFC 8259 {{RFC8259}} by Tim Bray.  This document
reuses a substantial amount of text from those documents, particularly
in the descriptions of objects, arrays, numbers, strings, and character
encoding.  The debt is gratefully acknowledged.

The relaxations of JSON's punctuation, string, and number syntax
adopted here were developed by the JSON5 {{JSON5}} and JSON6
{{JSON6}} communities.
