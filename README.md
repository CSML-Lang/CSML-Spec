# Data Oriented Markup Language - DOML (.Net)
> By Braedon Wooding

> Latest Version N/A (Approaching 0.1)

> *Note: This spec is changing a whole lot!  Until its marked at-least 1.0 you should regard it as unstable*

## Brief Introduction
DOML (Data Oriented Markup Language) is a language that enacts to stop re-inventing the wheel and start inventing wings, effectively it enacts to solve the 'problem' (as in software problem) of serialization in a different way then all other popular languages like XML/JSON/YAML/TOML/...

Its entire ABNF is only 74 lines compared to TOMLs 219 line grammar (which is ~3x larger), this is including comments and nice spacing in both.  DOML has two rules; creating new objects, and manipulating parts of those objects.  

DOML also erases the need for a middleman implicitly, no longer do you have to convert the JSON to a map then convert it to your data types, or try to use an automatic method (which often requires reflection).  DOML parses to an IR format which is ran, this means that if you have a parser that is compliant you could get IR code from it and then use it in another parser and further more it means that parsers could add new features and those new features would work in other parsers since there is a restricted set of IR commands.

All together it makes it significantly faster than JSON/XML/TOML/... parsing (in basic testing, I'll do some serious benchmarks later).  This is mainly due to the fact that the syntax and parsing doesn't have to do any look-aheads, and we create IR because its easier to parse and transfer over the web (example being that you convert your source into IR and transfer it in binary format to the server to run).  Having a computer run the code as it reads it could also be a possibility.

Also I should add that it is safe (in comparison to formats like YAML) this is mainly because it doesn't allow arbitary creation of objects, you can add new objects that it can create but it can't define them for you.

## Objectives
- Efficient (cost of speeds have to provide a high benefit of utility)
- Integrated into your project (no longer a separate part)
- Simple (arrays are implicit for example)

## A quick overview
```C
// This is a comment
// Construct a new System.Color
@ Test        = System.Color ...
;             .RGB             = 255, 64, 128 // Implicit 'array'

@ TheSame     = System.Color ...
;             .RGB(Normalised) = 1, 0.25, 0.5, // You can have trailing commas

@ AgainSame   = System.Color ...
;             .RGB(Hex)        = 0xFF4080
;             .Name            = "OtherName

/* Multi Line Comment Blocks are great
  // Note: nesting is allowed its just till I make a github syntax and it gets accepted I won't include it in examples, since I'll use the C one in the meantime and that doesn't allow nesting multi line comment blocks
  Anyways lets go and copy another previous one by just copying over the values.
*/
@ Copy        = System.Color ...
;             .RGB             = Test.RGB
;             .Name            = "Copy"
```
When you put this into a parser you'll get the below output (its standidized so you **need** to get the below output, though I've not included all the comments to keep it more concise and short);
```assembly
; This is an autogenerated IR representation of the supplied source code
;           This is a comment             ; USER COMMENT
; Setup Space
makespace   4                             ; Makes sure the stack supports 4 objects at a time
makereg     4                             ; Makes sure there are 4 registers
; Test.RGB
new         System.Color                  ; Creates a new object from System.Color
regobj      0                             ; Registers top object to register 0
push32i     255                           ; Pushes 32
push32i     64                            ; Pushes 64
push32i     128                           ; Pushes 128
pushobj     0                             ; Pushes object from register 0
set         System.Color::RGB             ; Call RGB on System.Color
; TheSame.RGB.Normalised
new         System.Color                  ; Creates a new object from System.Color
regobj      1                             ; Registers top object to register 1
push32f     1                             ; Pushes 1.0f
push32f     0.25                          ; Pushes 0.25f
push32f     0.5                           ; Pushes 0.5f
pushobj     1                             ; Pushes object from register 1
set         System.Color::RGB.Normalised  ; Call RGB.Normalised on System.Color
; AgainSame.RGB.Hex
new         System.Color                  ; Creates a new object from System.Color
regobj      2                             ; Registers top object to register 2
push32i     16728192                      ; Pushes 16728192
pushobj     2                             ; Pushes object from register 2
set         System.Color::RGB.Hex         ; Call RGB.Hex on System.Color
; AgainSame.Name
pushstr     "OtherName"                   ; Pushes "OtherName"
pushobj     2                             ; Pushes object from register 2
set         System.Color::Name            ; Call Name on System.Color
; Copy.RGB
new         System.Color                  ; Creates a new object from System.Color
regobj      3                             ; Registers top object to register 3
pushobj     0                             ; Pushes object from register 0
call        System.Color::RGB             ; Gets RGB of System.Color
pushobj     3                             ; Pushes object from register 3
set         System.Color::RGB             ; Sets RGB of System.Color
; Copy.Name
pushstr     "Copy"                        ; Pushes "Copy"
pushobj     3                             ; Pushes object from register 2
set         System.Color::Name            ; Call Name on System.Color
```
I won't go into great detail about the IR, but effectively it is similar to assembly; the `;` is a line comment, each command is seperated by a line and has one parameter (only one).

## Quick Spec Details
- DOML is case sensitive
- DOML must be a valid UTF-8 encoded unicode document (though I don't see why you can't support other formats)
- DOML refers to the initial source code not to the produced IR, and when I refer to IR I refer to the produced output.

## Types
This will be covered in more detail elsewhere but here are all the types

| Type          | Example Values                        | Details (all suffixes are case insensitive)        |
| ------------- | ------------------------------------- | -------------------------------------------------- |
| Integer       | 12, -40, +2, 01010b, 0x40FF, 0o42310  | 0b for binary, 0x for hex, 0o for octal            |
| Double        | 10.5, 20, 5e+22, 1e6, -2.54E-2        | E for exponent                                     |
| Decimals      | $40, +$99.05, -$4e+22                 | Can use E for exponent and '$' refers to decimal   |
| String        | "This contains a \" escaped quote"    | "...", you can escape `"` with `\`                 |
| Char          | 'C', '5'                              | Maps to a character                                |
| Boolean       | true, false                           | The boolean values                                 |
| Object        | Test, X, MyColor                      | Refers to a previously defined object              |

> Note: decimals have standidised for `$` though many parsers will probably allow the various other currency signs.

## Syntax
I'll cover this quickly here you can view an indepth either under [indepth syntax](doml_formal_spec.md) or at by viewing the [abnf](doml.abnf).

#### Comments
C-Styled comments either `//` or `/*` nesting is allowed for both;
```
// This is a comment
/* This is a block and /* this is a nested block */ */
```

#### Structure
The structure is simple you have two types of 'calls' you have either 'creation calls' or 'set calls' they look like either;
```C
// Create X
@ X   = Y.Z
;    X.A = B, C, D
```
Where `B`, `C`, `D` can be anything from the above type table, they can also refer to a 'getter' like `System.Version`.

You can shorten the structure a little using the `...` operator such as;
```C
@ X   = Y.Z ...
;     .A = B, C, D
```
Note that the second X wasn't required, this is because the `...` means just presume that every thing that starts with `.` is referring to `X` UNTIL you meet another `@` (regardless if that `@` has a `...`) you can always just refer to other objects like;
```C
@ B   = W.V
@ X   = Y.Z ...
;     .A = B, C, D
;    B.E = F, G, H
```

#### That's it!
> Wait what?
Yes it may seem weird but the syntax is THAT simple!  It will probably expand a little but I want to refrain from heavy changes that break a lot of things.

#### Complex Data Types
There are no complex types in DOML for good reason, there is no need for them.  If you want nesting of objects then just merely define the sub object before the parent object and assign it like;
```C
@ B   = W.V ...
  .Q  = 2
@ A   = X.Y ...
  .C = B
```
Arrays may be supported in the future (I have a concept of some IR code) though currently multiple parameters do fit the job (though the requirement of a constricted stack size hurts parameter lists with undefined amounts).


#### But no arrays...?
There are still arrays remember `B, C, D` you passing multiple values to `.A` your effectively passing a heterogeneous array (allows any type), at its core arrays are just this (well that and they are homogeneous or the same type) modern arrays often tack on a little length variable and I guess you could always do that yourself, and the argument may be that we should handle that, but I'm still not convinced on that front since the ONLY case where that would be useful would be when you are passing multiple values to a function and you want one of the values to be an array and the others to not be an array.  Otherwise you can always just either keep popping till you get a 'false' (run out of elements) or be a good person and do a for loop using a variable that says how big the 'stack' is.

Note: this is where I should say that all the DOML implementations by me will have the same API (with only minor differences) and I suggest if other people make new implementations they follow the same API guidelines, though I'm not going to enforce that so the whole array thing is more of a parser implemented thing then anything.

## Comparison with other formats
Firstly I'll list the problems with other formats; XML has always been hard to read, hard to write (though I would claim its still easier to write then JSON by quite a landslide) and especially hard to parse and its spec is a complete mess.  JSON is easy to parse, but still suffers from being annoyingly verbose.  YAML is much improved in this manner, but however due to its tight bindings to JSON (all JSON works in YAML parsers) it stretches out its SPEC even further (and YAML's spec is sooo large that no parser is capable of supporting it all), and even further more its reliance on JSON hinders whitespace meaning that if you choose indentation you need to like Python keep everything indented right which means that its not compact enough for web use (and JSON isn't either though its often used due to its native compatibility with Javascript).

A few others;
- INI is old, and suffers from a lot (though I quite like it for its simplicity)

So what one is like DOML the most?  Well I guess I would say TOML?  TOML is a great step forward in the advancement of markup languages though I still feel it suffers from trying to create a new thing from what already is here rather than create a new thing from the core fundamentals of what it should be (which DOML was created out of), this is why I can't really compare it greatly with other formats...  Because it was built to be a different sort of tool, a different sort of markup language it accomplishes the same goal but in a way that I would *claim* has never been done before (succesfully at least).  This isn't just another markup language (damn should have included a YAML pun), but rather I'm sure some will claim isn't really even a 'markup language', and shares more similiarity with scripting languages then actual markup languages.

Though I would claim differently, the clear difference between a scripting language and a programming language is *how* you use it and in what context, and I would further claim that the biggest difference between a markup language and a scripting language is how you *can* you use (or rather *should*), since while DOML is 'turing complete' (though you would have to add functions like math and basic drawing for anyone to be like 'yeh now its turing complete') it purposefully limits actions like math and so on, since its built to represent data thus its a data oriented language.  Every action you perform is around the manipulation of data (again the clear distinction is that the 'purpose' of the language is to create data the purpose of other languages is to creation applications and while both may 'manipulate data' its clear to see what is different).

So to 'wrap up', DOML is more than just a simplistic alternative, its a 'scripting language' with the whole goal of creating data, not with using it!  Thus I would claim its a markup-language due to the similar goals and the restrictions placed on it, though I'm sure others will disagree.

## Get Involved!
Anyways, if you have any new changes you may wish to add please ask in the issues!  I will be more cautious to accept PRs if the changes aren't talked about in an issue (though the obvious exception is if you are fixing up typos/errors or if its a super small thing, OR if you are adding your implementation / project to the list all these don't require issues of course).  Further more I'll open up a discord chat somewhat soon (and maybe a mailing list, though nowadays they seem a little archaic).

## Projects using DOML
- None yet, if you know of one please send a PR adding to this list!

## Implementations of DOML
If you have an implementation, send a pull request adding to this list. Please note the version tag that your parser supports in your Readme.

## v0.1 Compatible

## In Progress
- [.Net (C#/F#/...)](https://github.com/DOML-DataOrientedMarkupLanguage/DOML.net)
- [C++](https://github.com/DOML-DataOrientedMarkupLanguage/DOML-Cxx)

## Editor Support
- None yet, but VIM/Notepad++/EMACS/Sublime Text/VS Code are all in development
