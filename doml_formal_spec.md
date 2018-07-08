# DOML Formal Specification

## What is this

This is the formal specification document for *DOML*, mainly aimed at developers who wish to implement a parser.

## Terminology

The following are key words and their definitions;

- **must**/**should**/**may**/**must not**/**should not** : Are defined as [such](https://tools.ietf.org/html/rfc2119).
- *IR* : Refers to the bytecode produced through the 'compilation' of the *DOML* source file (stands for Intermediate Representation).
- *DOML* : Refers to Data Oriented Markup Language (this 'language')

## Table of Contents

- [File Format](#file-format)
- [Other Details](#other-details)
- [Syntax](#syntax)
- [Comments](#comments)
- [Types](#types)
- [Collection Types](#collection-types)
- [DOML Specifics](#doml-specifics)
  - [Constructors](#constructors)
  - [Assignments](#assignments)
  - [Arrays](#arrays-of-objects)
  - [Maps](#maps-of-objects)
  - [Calls](#calls)
  - [Embed IR](#embed-ir)
- [IR Specifics](#ir-specifics)
  - [Whitespace](#whitespace)
  - [Architecture](#architecture)
  - [Required Commands](#required-commands)
- [Interfacing DOML](#interfacing-doml)
  - [Static vs Reflection Bindings](#static-vs-reflection-bindings)

## File Format

The file ending **must** be `.doml` for *DOML* files, and if a file is purely IR it can either have `.doml` or `.odoml`. Furthermore, they **must** be encoded with standard UTF-8, though parsers **could** support other formats.

## Other Details

- *DOML*
  - **Must** be case sensitive
  - **Must** be white space insensitive
- *IR*
  - **Must** be case sensitive
  - **Must** be whitespace sensitive
  - **Should** use 2 digit numbers for their opcodes (i.e. 09 instead of 9) since the count right now is 18 (which has an opcode value of 17) and it won't go over 99 in the foreseeable future. This just makes alignment look nice.
  
Throughout this document I'll use the word equivalent for IR, word equivalents don't 'have' to be supported and just the opcode variants have to, but it is much harder to show complex IR without the word values.

## Syntax

The following details relate to the syntax.

### Comments

The following comments **must** be supported for all *DOML* code;

```C
// A
X : Y() // B
```

The following comment **must** be supported for all *IR* code;

```Assembly
; A
push 0 50 ; B
```

### Types

Both *DOML* and *IR* share the same type system, all the following types **must** be implemented.

- Integers (type ID: `0`, type name: `int`)
  - signed 64 bit (8 bytes) by default
  - 0x prefix to make it hexadecimal, 0b prefix to make it binary, and 0o to make it octal (case insensitive)
  - Can have `_` in numbers. Though they can't however occur at the beginning or ending of a number and you can't have two next to each other.
- Floating Point (type ID: `1`, type name: `flt`)
  - Double Precision 64 bits (8 bytes), by default
  - IEEE 754
  - Can have `_` in numbers. Though they can't however occur at the beginning or ending of a number and you can't have two next to each other and can't occur next to the `.` (if there is one).
- Decimals (type ID: `2`, type name: `dec`)
  - `$` prefix (note the `+` or `-` goes before the `$` prefix i.e. `-$40.95` or `+$59.54`/`$59.54`).
  - Double Precision 64 bit (8 bytes) by default.
  - Decimal typed, C# shows it well [here](https://docs.microsoft.com/en-us/dotnet/api/system.decimal?view=netframework-4.7)
  - Can have `_` in numbers. Though they can't however occur at the beginning or ending of a number and you can't have two next to each other, also have to occur after the `$` and can't occur next to the `.` (if there is one).
- String (type ID: `3`, type name: `str`)
  - Begins with `"` ends with `"`
  - You can escape quotes only using `\"`
  - You can insert a unicode character like `\u4e00` (insert unicode character 4e00 which is; 一)
- Boolean (type ID: `4`, type name: `bool`)
  - `true` and `false`
  - Thus **can't** have any objects with names that match `true` or `false`.
- Objects (type ID: `5`, type name: `obj`)
  - Represents any creation object for *DOML* and for *IR* refers to a register ID.
  - **Should** be represented as a pointer or similar (i.e. a reference in C#).
  
### Collection Types

Both *DOML* and *IR* have support for collections and thus the following types **must** be implemented.

- Arrays (type ID: `6`, collection name: `vec`)
  - Array elements **must** all have the same type in strict mode.
- Dictionaries/Maps (type ID: `7`, collection name: `map`)
  - The key type **must** remain constant, and so **must** the value type in strict mode.
> Note: this is only when 'used' they can remain with different types in literal definition but as soon as they are used they must just have the same type.

### DOML Specifics

The following sets of rules apply only to *DOML* code.

#### Style of syntax

All DOML sections below will follow the following style;
```C
// #. <Description>
<Example1>; <Example2>; ...; <ExampleN>;
```
Then below each number will be explained thorougly, the `;` is to indicate a new way to present that example that has the SAME result.

All IR sections below will follow the following style;
```assembly
; #. <Description>
<IR Code> <Parameters>
```
There is typically just one way to do things in IR, so no need to chain multiple to show off the syntatical stuff.  Also a few symbols may pop up on the IR that mean different things;

- `#`: means a register ID (dependent on the previous IR commands before)

#### Constructors

*DOML* has the following syntax for constructors;
```C
// 1. Standard
A : B::(); A : B; A : B::B();

// 2. Using a constructor
A : B::C(n1: p1, n2: p2, n3: p3, ..., nn: pn);
```
1) Creates an object A of type B with standard constructor.
2) Creates an object A of type B with constructor C (with parameters a...n, with labels n1..nn).

Notes:
  - Constructor overloading is NOT supported.
  
##### Resultant IR

The above code generates the following IR;

```assembly
; 1. Standard
newobj # B B
; 2. Using a constructor
; ... Pushes
newobj # B C
```
1) Creates an object of type B calling constructor B (aka the default) at register '#' (at runtime register '#' or -1 means any, but in this case it does mean dependent on previous values).
2) Creates an object of type B calling constructor C at register '#' (same reason as above).
  - note: the labels aren't expressed in IR.
Note: This presumes you have used attributes to define the objects.

#### Assignments

```C
// 1. Accessing and setting an element
A.B = p1, p2, ..., pn
A.{ B = p1, p2, ..., pn }
A : C() { B = p1, p2, ..., pn }

// 2. Performing a single assignment
A.B = p1
A.{ B = p1 }
A : C() { B = p1 }
```
1) Assigns objects p1, p2, ..., pn to B of A.  They all do the same thing in the end (that is if A is always of type D).
  - Note: p1..pn don't have to be of the same type.

##### Resultant IR

```assembly
; 1. Accessing and setting an element
push typeID p1
push typeID p2
...
push typeID pn
call # C B

; 2. Performing a single assignment
quickpush typeID p1
quickcall # C B
```
1) Pushes 'n' objects onto the stack before collapsing 'n' objects to perform call.
  - If typeID are all the same it can be collapsed into; `push typeID p1, p2, ..., pn`
2) Performs a smart push on an object then performs a quick call with just one previous object.

A few notes about this structure;
- The type ID follows the ones stated in the [Types](#types) section, note: this doesn't work for arrays/maps a different command is for that.

#### Arrays of Objects
There are two cases where arrays come into form in DOML; the first is arrays of objects and the second is parameter arrays.

The first, arrays of objects;
```C
// 1. Implicit Size
A : []B { 
  ::(){ D = a1, ..., an },
  ::C(p1, ..., pn){ D = a1, ..., an },
  ...
  { D = a1, ..., an },
}

// 2. Explicit Size and assignment
A : [3]B

A[0] : B::() {
  D = a1, ..., an
}
A[1] : B::C(p1, ..., pn) {
  D = a1, ..., an
}
A[2] : B {
  D = a1, ..., an
}
```
1) Assigns an implicitly sized array and assigns each index to a member; you can also assign a length if you want and set just some of the elements.  Elements are typesafe and bounded, so you can't go outside the array or any silliness like that.

2) Assigns an explicit size and the assigns each element follows same rules as above.

##### Resultant IR

Now interestingly enough the IR produced by above actually doesn't create an array, and follows the same as if you defined something like `A1, A2, A3, ... An`.  HOWEVER when referenced it is still treated like an array just the IR has no concept of object arrays (does have concept of array objects however) this means that something like `D = A` where A is defined as something like above, would decompose to `D = [A1, A2, ..., An]` in terms of the IR (below will be how arrays exist in IR).

Now this is done because in reality array objects don't really need to be an array since one it complicates the IR and two its just a nice way to express multiple objects as belonging to a single 'set'.  When referring to each object in IR comments you should reffer to it as `A-x` or `A[x]` with the dash/brackets to signify it is an array.

#### Object Arrays
You can create an object array by declaring it as such;
```C
// 1. Declaring implicit size
A.B = [o1, ..., on]
// 2. Declaring an explicit size
A.B : [n]Type {
  o1, ..., on
}
A.B : [n]Type
A.B[0] = o1
...
A.B[n] = on
```

Both declares an array of objects of the same type, this **should** implemented as a static array but in languages like Python which don't understand the concept of static arrays implementing it as a dynamic is still compliant.  However, going outside the explicit bounds or implicit ones (n in this case) **must** not be supported.

Note: they should decompose to the same code however optimisations may be performed converting the first option to a single push call that can be cached till you push the array saving a set of copies, or that you can perform array like copies that are more efficient then typical ways.  The second one could be optimised.  Optimisations don't interact with compliance so you are free to optimise calls as long as the result is the same.

You can edit elements of an array like;
```C
A.B[0] = 2
A.B[1] = A.B[2]
```

##### Resultant IR

This is a little more complicated then other IR due to the level of optimisation possible SO I'll outline the basic structure, and in the IR section I'll cover some of the optimisations possible;

```assembly
pusharray typeID n
setarray typeID 0 o1
...
setarray typeID n on
```

You can perform a wide 'setarray' by doing a 'arraycpy' like, which is significantly more efficient;
```assembly
pusharray typeID n
arraycpy typeID n o1 o2 ... on
```

Note: if you want array of arrays that are n dimensional and square then you should either use the `setarraynth` command (like `setarraynth typeID x y val`) or the `arraynthcpy` command (like `arraynthcpy typeID N x1 ... xn y1 ... yn z1 ... zn` where N is the row count * height count).  You can also use the compacting method that is (this is required for arrays of maps);
```assembly
pusharray typeID n
arraycpy typeID n o1 o2 ... on
pusharray typeID n
arraycpy typeID n p1 p2 ... pn
... ; 'N' times
compact typeID N
```
Compact takes type and number of arguments, it then pops those arguments from the stack and places those into the array.  This is the way to build jagged arrays.

Accessing arrays are discussed under accessing values.

#### Maps of objects
You can define maps of objects like;
```C
A : [B : C] {
  {
    b : C::E(k1, ..., kn) {
      X = x1, ..., xn
    }
  },
  ...
  // And so on
}

// Accessing elements
P : C = A[b]
```

This is extremely similar to how arrays of objects works so I won't redescribe, it should produce NO IR and purely should be for the user to simplify their code.  Note: the key can only a literal type (excluding objects) so like a string, int, float and so on (using the type names specified in the type section).

#### Map Objects
You can define map objects in code like;
```C
// Implicit
A.B = {
  c1 : D::E(p1, ..., pn) {
    F = f1, ..., fn
  },
  ...,
  c2 : D::G(k1, ..., kn) {
    F = f2_1, ..., f2_n
  },
}

// Explicit
A.B : [C : D]
```
Note: you can combine the explicit and implicit into one.

##### Resultant IR
Maps have somewhat similar IR to arrays.

```assembly
pushmap typeID typeID
setmap typeID typeID key value
```
Setting multiple values can be shorted to
```assembly
pushmap typeID typeID
quicksetmap typeID typeID k1 v1, ..., kn vn
```

typeID can only contain the types stated in the type section.  If you want a map of arrays (and same for map of maps) then you have to do the following;

For map of maps that are one dimensional
```assembly
pushmap typeID typeID
quicksetmap typeID typeID k1 v1, ..., kn vn
push typeID new_key1
pushmap typeID typeID
push typeID new_key2
quicksetmap typeID typeID k2_1 v2_1, ..., k2_n v2_n
... ; 'N' times
zipmap 1 typeID typeID typeID N
```
The above takes a collection type along with the new key type and the old map types that is you can view it like `zipmap 1 [typeID : [typeID : typeID]] N`, note how I had to push elements after each one, if you don't want to do that then you can create a complex map.

Complex maps are actually quite hard to express so there are two commands, the first is through `#IR_type` to express your map type then later on you can refer to that type by its ID you provide.

That is to create a string map of a int to array of floats map that is `[str : [int : []flt]]` you would do;
```assembly
#IR_type 2 1 3 1 1 0 1 0 0 1
; Or in simple mode
#IR_type myType 3 map str map int vec flt
```
That is you create a type and call it ID `2` with depth `3` and that it is a map (1) then you state its a `string` (id 3) and its collection type is map (1) then you state the key value is int (0) and the value is array (0) then you state the type is float (1).  So yeh... it is quite complex to define them but once they are defined you can just use them like any other collection type i.e.;
```assembly
; If not in simple would refer to it as '2' as that is what the type was defined as
pushcollection myType
setcollection myType string_value int_value n float_0 ... float_n
```
where `n` is the number of items in the array.  As you can see the type system is quite sophisticated, once more try not to abuse collections, and if you can completely avoid them that is preferred.  This is more down to the user then you but still trying to expand the uses of zip map to more types if you can is important.  Doing maps of maps just gets icky and for most users they will just define simple map structures, though maps of arrays are also a bit icky so in the future having a nicer command for them could exist.

#### Calls

Take the following example;
```C
A : B::C() {
  d = p1
}

D : B {
  // 1
  d = A.d
}

// 2
D.d(A.d())

// 3
D.d = A.d
```

There are a few function call 'types' above and they effectively all do the same thing.  1, 2, and 3 do the exact same thing and are semantically the same, we allow you to shorten things as to make them nicer in terms of a call.

###### Resultant IR

The IR resulting from 1, 2, and 3 is;
```assembly
get #A B d
call #D B d
```
You could use a quick version here of course, or whatever optimisations lead to the same 'result'.

#### Embedding IR

Embedding *IR* **should** be supported by a parser to allow for more complex operations to occur that aren't supported by the grammar (and in most cases will eventually become supported).  Using IR does NOT void the compatibility promise, however don't expect IR to always be the most efficient in future versions (as better commands may be introduced).

```C
Example : Color {
  RGB = 4, 55, 255
}
// `RGB = 4, 55, 255` is equivalent to:
#IR simple {
  #IR_obj Color Color;
  #IR_ctor Color Color::Color;
  #IR_set RGB Color.RGB;
  newobj Color Color #Example;
  push int 4, 55, 255;
  call #Example Color RGB; '0' as in register '0'
}
```
Noting the use of `simple` which allows me to use words instead of integer values (as well as using `#` for registers allowing alphanumeric) for almost everything, making the code look very readable, it also allows me to define IR objects/setters/getters/contructors through attributes.  This is covered in the [attributes](#attributes) section.

#### Attributes

You can apply a few attributes to your DOML code to provide various different benefits;
- `#Version X.Y.Z` you can state your version like `#Version 5` or `#Version 5.1` or even `#Version 5.5.9` compilers are required to check this and make sure that your version will work with the current compiler this will help people upgrade their DOML code if any compatability stuff breaks.  Lowest version allowed is `#Version 0.3` as before that the language was different (in really just terms of syntax and complexity of IR) and very few compilers ever fully supported `0.2`.  Compilers **must** support this.
- `#IR { ... }` allows you to embed IR, an optional mode is `#IR simple` which allows you to use simple words such as `push` or `int` instead of their value equivalents, noting of course that the extra work to analyse will slow down parsing.  Compilers **should** support this and **could** support simple mode.
- `#Strict true/false` supplying a true value should require the compiler enforce only **must** and **should** requirements and definitely turn off all **could** or any ones not stated in this document.  Compilers **should** support `#Strict true` and **must** support `#Strict false`.
- `#NoKeywords true/false` compilers **must** support `#NoKeywords false` and **should** support `#NoKeywords true` which turns true and false into `#true` and `#false` for all cases except for attribute statements.
- `#IR_obj` defines an object for the rest of the scope
  - If used within a block only defines for that block if used outside a block defined for entire script.
- `#IR_set` defines a setter for the rest of the scope
  - If used within a block only defines for that block if used outside a block defined for entire script.
- `#IR_get` defines a getter for the rest of the scope
  - If used within a block only defines for that block if used outside a block defined for entire script.
- `#IR_ctor` defines a constructor for the rest of the scope
  - If used within a block only defines for that block if used outside a block defined for entire script.
- `#IR_type` used to define a type.
  - i.e. `#IR_type 100 1 3 1 0 0 1` makes a map of string to another map which is int key to a float array.  This can be read like `[str : [int : []flt]]`, and create type would often be expressed like; `#IR_type myType map str map int vec flt`.  This type can be used in place of more complex types such as `getmap myType 3, 5.0, "Hey"`
  
### The Type System (more relevantly `#IR_type` and its relation to complex types)

How the type system works is quite simple there are two preface types (which are always 'combined' with the type after them) such as if you wanted to use 'getmap' where you had a structure like `[str : [int : []flt]]` you would define it like `getmap str map int vec flt` followed by the arguments.  The type system can be simplified through defining types (the first 100 from 0-99 are reserved by the system) and in simple mode you can use a named type.  Effectively it is broken down like this; when you state `map` you state that you are going to give it two types after it forming the type of the map such as `map str int` (map of `str : int`), and if you give it `vec` you will follow it with a single type for the vector class.  Other structures like tuples may exist in the future.

Note: that vec/map don't count other 'collections' as part of their requirement i.e. `map vec map int flt map flt vec str` is the same as `[[][int : flt] : [flt: []str]]` you can if you want in simple mode use brackets to group things like `map (vec (map (int flt)) map (flt vec (str)))` noting that all brackets are ignored though you can't miscount them, furthermore you could extend it to multiple lines if you wanted but its not suggested you do.  You can use `#IR_type` to define a type allowing you to use it later, you can combine multiple IR_types as they are effectively just 'copied' into their callsite i.e.

```assembly
#IR_type fltToInt map (flt int)
#IR_Type strArr vec str
#IR_type i int
#IR_type intToStrArr map (i strArr)
#IR_type fltToInt_To_IntToStrArr map (fltToInt intToStrArr)
```
Of course above is an extreme example.  Generally we suggest to avoid doing weird types like arrays as keys and keep it to primatives as some systems won't allow you to actually define a weird type like a vector as a key type (or recommend not too).

### IR Specifics

#### Whitespace

- Whitespace is important in *IR*
- At least one space has to exist between the *IR* command and the parameter.
- At least one newline between commands (i.e. commands can't exist on the same line)

#### Architecture

The Architecture of *IR* has been standardized just to maintain consistency.

- All pushing/popping operations should operate on a stack based model
  - It **shouldn't** be dynamically allocated since the `init` command initialize the stack and registers.
  - You **must** expose both the current size of the stack and the max size.
    - The current size represents how many elements have currently been pushed through `cursize` (pushing size onto stack)
    - The max size represents the total space available for elements (equal to `init` parameter) through `maxsize` (pushing size onto stack)
  - It **should** *ONLY* allow pushing/popping/peeking operations (though note : hat a peek can be implemented through popping and then push that value as well as return it, though this is inefficient and a more native method should be implemented)
  - You **could** implement it using either a `stack` or a static array with pointer/index
- `call` ooperations operate on a register based model
  - It **shouldn't** be dynamically allocated since the `init` command grants the size.
  - You **must** expose the current size of the registers which represents the total size available for objects (equal to `init` parameter) through `regsize` (pushing size onto stack)
  - It **should** *ONLY* allow indexing operations (both to set and to 'unset' - or set to null)
- All the IR commands **should** be implemented through an 'opcode' enum like structure which has been defined to be from 0-255 or 256 unique values (i.e. an unsigned 8 bit integer or a byte).

#### Required Commands

The following are all the required commands.  All parsers **need** to support the following.  The name can only be used in 'simple' mode.

They will be in the format `<command>(opcode) < < parameterName: parameterType >, < ... > >`.

- `nop(00) void`: does explicitly nothing
- `init(01) <Stack Size: int> <Register Size: int>`: sets up the stack and registers
  - The parameter represents the new size not the difference
  - Objects aren't carried across so effectively a wipe
  - If either size < current size then that initilization doesn't occur.
- `deinit(02) void` useful in some rare cases, just deinitialize all the memory freeing it.
- `newobj(10) <Register: int> <Type: type> <Constructor: ctor>`: creates a new object 
  - The register refers to what register this object is created in.
  - The count refers to how many parameters there are.
- `push(11) <Type: typeID> < <Parameter: parameter> >`: pushes objects of the same type onto the stack
  - All parameters have to match the type given.
- `call(12) <Register: int> <Type: type> <Function: setter>`: calls a register object of a certain type along with the number of stack objects given.
  - If you want to call an object on the stack you have to use `callstack` or simply `regobj` to register an object to a register then you can follow up with a `calln` (which is faster if you are doing multiple calls).
- `callstack(13) <Type: obj> <Function: setter>`: calls a function of type given on the top object of the stack (doesn't pop it).
- `pop(14) <N: int>`: pops number of objects off the stack.
- `get(15) <Register: int> <Type: type> <Function: getter>` calls a function and places value onto stack.
- `getstack(16) <Type: type> <Function: getter>` sames as `get` but for the stack.
- `regobj(17) <Register: int>` registers the top object on the stack to the register value given, pops the object.
- `quickpush(20) <Type: typeID> < <Parameter: parameter> >`: Quick push doesn't push to the stack but rather stores it in a smaller 'register' like addressable state.
  - Note: normal calls won't work with quick pushes, only quick calls work.
- `quickcall(21) <Register: int> <Type: type> <Function: setter>`: Performs a call with quick push'd variables.
- `pcall(22) <Register: int> <Type: type> <Function: setter> < <Parameters> ... >`: performs a call with the parameters in the call
- `pnewobj(23) <Type: type> <Constructor: ctor> <Register: int> < <Parameters> >` Allows you to supply parameters in call.  Parameters have to be calculated at runtime so it is less efficient.
- `pget(24) <Register: int> <Type: type> <Function: getter> < <Parameters> ... >`: same as `pcall` but functions like `get`.
- `quickget(25) <Register: int> <Type: type> <Getter: get>` very similar in nature to `quickcall` but functions like `get`
- `pusharray(30) <Type: TypeID> <Len: int>`: pushes an array of length and type given onto the stack.
- `setarray(31) <Type: TypeID> <Indicies: int[]> <Obj: Type>`: indexes and sets an object.
- `getarray(32) <Indicies: int[]>`: indexes an object and pushes value onto stack.
- `arraycpy(33) <Type: TypeID> <Len: int> < <Obj> ... >`: basically builds the array through using a memcpy if it can as in the case of binary streams and in other cases also I'm sure.  Should be more efficient anyway as cuts down number of instructions.
- `compact(34) <Type: TypeID> <N Dimension: int>` compacts above arrays into dimensions given, i.e. if you give it two arrays will build a 2D array, the order is from top down not down up.
- `pushmap(40) <KeyType: TypeID> <ValueType: TypeID>` pushes a map onto the stack.
- `setmap(41) <KeyTypes: TypeID[]> <KeyType: TypeID> <ValueType: TypeID> < <Key: KeyTypes[0]>, ... > <Value: ValueType>` indexes and sets map for key and value given.
- `setmapstack(42) <KeyDepth: int>` runs setmap by using the stack.
- `quicksetmap(43) <KeyTypes: TypeID[]> <ValueType: TypeID> <N: int> < < <Key: KeyTypes[0]>, ... > <Value: ValueType>, ... >` just applies multiple set maps in a row, more efficient as less IR but doesn't have any benefits like memcpy.
- `zipmap(44) <Depth: int> <KeyTypes: TypeID[]> <ValueType: TypeID> <Count: int>` can be an efficient way to mass create a complex map, will zip maps together as per your depth, WON'T however allow you to have arrays for either keys or values.
  - When calling it, it will pop off items, you give it on the stack like the key is the top (or last pushed) and then next is the 'deeper' map i.e. if you have `[int : [str : flt]]` you would have the `[str : flt]` map at the bottom with the `int` key above each of its corresponding map and if you had `[int : [int : [str : int]]]` you would have the first int key at the top then the second int key with each of its corresponding map i.e. `map01 int1 map02 int2 ... top_int1 map1y inty ... top_int2`.
- `getmap(45) <KeyTypes: TypeID[]> <ValueType: TypeID> < <Key: KeyType[0]>, ... >` pushes value at key onto stack.  If value doesn't exist for key should push 'NULL' if in safe mode else if in strict mode raises panic.
- `getmapstack(46) <KeyTypes: TypeID[]> <ValueType: TypeID>` same as getmap but uses the stack.

## Interfacing with DOML

There are a few ways to interface with *DOML*, each one **should** exist in either a static or reflected binding (offering both could allow users to choose one that is more applicable to them).

### Static vs Reflection Bindings

Both are valid, and both have their advantages (static often requires code generation but is fast whereas reflection is dynamic, doesn't require code generation, requires little to no input from 'users' but is often quite a significant amount slower).  The best is compile time reflection as it requires very little from the user but provides as efficient code as static bindings, languages like Zig, D, and to a certain point Rust.

If your language doesn't support any introspection (i.e. C/C++) then either static analysis or maybe just a simple parser?  Or perhaps just requiring users to hook in to your program via something like macros.
