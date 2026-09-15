# NAME

Perl::Critic::Policy::ProhibitRegexForSimpleSubstring - Use index() to look for literal text, and leave split its pattern.

# VERSION

version 1.001

# Perl::Critic::Policy::ProhibitRegexForSimpleSubstring

A regular expression made of nothing but literal characters is a substring
search, and `index` does that without compiling and running a pattern:

```
if ( $str =~ m/foo/ ) { ... }               # reported
if ( index( $str, 'foo' ) >= 0 ) { ... }    # what it means
```

Except where the regex is not a search at all.  The first argument of `split`
is the pattern it splits on, and there is no `index` that splits.
[Perl::Critic::Policy::BuiltinFunctions::ProhibitStringySplit](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy%3A%3ABuiltinFunctions%3A%3AProhibitStringySplit) requires that
argument to be a regex rather than a string, so reporting `split m/\n/, $text`
leaves nothing to write that both policies accept.  This policy lets `split`,
and anything else named in `allow`, take a literal regex as its first
argument.

This is a fork of
[Perl::Critic::Policy::Performance::ProhibitRegexForSimpleSubstring](https://metacpan.org/pod/Perl::Critic::Policy::Performance::ProhibitRegexForSimpleSubstring)
by Dean Hamstead.  What is new is the list of calls whose first argument is
exempt, and which modifiers exempt a pattern: see ["MODIFIERS"](#modifiers).

## PROHIBITED

```
if ( $str =~ m/foo/ ) { ... }
if ( $str =~ /bar\.baz/ ) { ... }       # an escaped dot is still a literal
print if m/foo/;
split $sep, $str =~ m/foo/;              # a match, not split's pattern
grep { m/foo/ } @lines;
```

## ALLOWED

```perl
split m/\n/, $text;
split /,/, $line;
my @fields = split( m/\t/, $row );
CORE::split( m/:/, $path );
```

And, as in the original, any regex that is not only literal text:

```perl
$str =~ m/foo/i;          # /i, which index() cannot do
$str =~ m/^foo/;          # an anchor
$str =~ m/fo+/;           # a quantifier
$str =~ m/[ab]c/;         # a character class
$str =~ m/(foo)/;         # a group, even of literal text
$str =~ m/foo|bar/;       # alternation
$str =~ m/$foo/;          # interpolation
$str =~ s/foo/bar/;       # a substitution
my $rx = qr/foo/;         # a compiled regex
```

## MODIFIERS

Only `/i` exempts a pattern, because only `/i` changes what a pattern of
nothing but literals matches: `index` is case sensitive.

```
$str =~ m/foo/m;          # reported: /m changes ^ and $, and there are none
$str =~ m/foo/s;          # reported: /s changes ., and there is none
$str =~ m/foo bar/x;      # reported: under /x this is the string foobar
```

A modifier in scope from a `use re` counts the same as one written on the match,
so a file under `use re '/sx'` is checked like any other, and one under
`use re '/i'` is exempt throughout.

## CONFIGURATION

- `allow`

    Space separated list of functions and methods whose first argument may be a
    literal regex.  It **adds to** the built-in list rather than replacing it, so you
    name only your own:

    ```
    [ProhibitRegexForSimpleSubstring]
    allow = grep_lines My::Util::match_all
    ```

    The built-in list is:

    ```
    split
    ```

What a name matches, which is the same rule as
[Perl::Critic::Policy::ProhibitLeadingZeros](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy%3A%3AProhibitLeadingZeros)'s `allow`:

- A name with no package, `split`

    Any call to a function or method of that name, however it is reached:
    `split(...)`, `CORE::split(...)`, `$obj->split(...)` and
    `Some::Class->split(...)`.

- A name with a package, `My::Util::match_all`

    Only a call that names that package: `My::Util::match_all(...)`, or the class
    method `My::Util->match_all(...)`.  Not a bare `match_all(...)`, and not
    `$object->match_all(...)`, since the policy cannot know what package either
    one ends up in.

Where in the call the regex may be:

- As the first argument

    Which is where `split` takes its pattern.  A literal regex anywhere else in the
    arguments is a match against `$_` whose result is being passed, and that is
    reported like any other: `split m/,/, m/x/` is a violation, for the second one.

- As the whole of that argument

    `split m/\n/, $text` is allowed; `split $sep, $str =~ m/x/` is not, since
    the regex there is an operand of `=~`.  Next to any operator but a comma, a fat
    comma or a low-precedence `or`, `and` or `xor`, a regex is part of an
    expression rather than an argument.

- Of the nearest call

    `foo( split m/\n/, $text )` is `split`'s argument, not `foo`'s, and
    `split` decides.  Parentheses around the regex are looked through,
    `split( ( m/x/ ), $s )`; a subscript, a block or an anonymous array or hash is
    not, and a regex inside one is reported.

## CAVEATS

A pattern chosen by a ternary, `split $tab ? m/\t/ : m/,/, $line`, is two
operands of `?:` rather than an argument, and both are reported.  Write the
choice as a `qr//` first, or say `## no critic (ProhibitRegexForSimpleSubstring)`.

A name with no package matches any method of that name on any object, since
there is no knowing what class an invocant is.  Name the package if that is too
broad.

## DIFFERENCES FROM THE ORIGINAL

What moving from `[Performance::ProhibitRegexForSimpleSubstring]` changes:

- A literal regex as the first argument of `split`, or of anything named in
`allow`, is not reported.
- `/m`, `/s` and `/x` no longer exempt a pattern, and neither does a `use re`
that turns them on.  The original exempted all three, which under
`use re '/sx'` meant it reported nothing at all.  See ["MODIFIERS"](#modifiers).
- A `## no critic (Performance::ProhibitRegexForSimpleSubstring)` does not match
this policy's name, so an annotation that is still needed has to be renamed to
`## no critic (ProhibitRegexForSimpleSubstring)`.

## SEE ALSO

[Perl::Critic::Policy::BuiltinFunctions::ProhibitStringySplit](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy%3A%3ABuiltinFunctions%3A%3AProhibitStringySplit), which is why
`split`'s pattern has to be a regex in the first place.

## METHODS

### supported\_parameters

`allow`, the functions and methods whose first argument may be a literal regex,
added to the built-in list.

### initialize\_if\_enabled

Folds the built-in names back into whatever `allow` was configured with, so a
user's list adds to the defaults instead of replacing them.

### default\_severity

SEVERITY\_MEDIUM

### default\_themes

performance

### applies\_to

PPI::Token::Regexp::Match

### violates

Standard [Perl::Critic::Policy](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy) interface.  Returns a violation for a match of
nothing but literal text, unless it is the whole of the first argument to a call
that `allow` names.

# BUGS

Please report any bugs or feature requests on the bugtracker website
[https://github.com/teodesian/perl-critic-policy-prohibitregexforsimplesubstring/issues](https://github.com/teodesian/perl-critic-policy-prohibitregexforsimplesubstring/issues)

When submitting a bug or request, please include a test-file or a
patch to an existing test-file that illustrates the bug or desired
feature.

# AUTHORS

Current Maintainers:

- George S. Baugh <teodesian@gmail.com>

Original author, of Perl::Critic::Policy::Performance::ProhibitRegexForSimpleSubstring:

- Dean Hamstead <dean@fragfest.com.au>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Dean Hamstead, as
Perl::Critic::Policy::Performance::ProhibitRegexForSimpleSubstring in
Perl-Critic-Policy-Performance-ProhibitRegexForSimpleSubstring.

Modifications are copyright (c) 2026 Troglodyne LLC.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
