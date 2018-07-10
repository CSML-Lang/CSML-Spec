# Changelog

This changelog will contain all the various changes to the grammar and syntax of DOML from v0.1 onwards.

## Version 0.3.2

- Type system overhaul, it is much easier to understand and should be more efficient as we have cut out a lot of the commands.
  - `vec` and `map` are type extenders that is they are to be followed with more types to formulate their types; i.e. `vec int` defines an integer array and `map int str` defines a integer to string map.
  - This means that push now accepts arrays and maps, and every call previously that took a type can now take vectors/maps, however you still can't put a vector or a map where it expects 'values'.
- Thus due to above collections are simplified, there is no specific arary/map instructions anymore as they take types to determine what the instruction is.
  - Thus; `pusharray`, `pushmap`, `setarray`, `setmap`, `arraycpy`, `getarray`, `pushcollection`, `setcollection`, `quicksetmap`, `zipmap`, `getmap`, `getcollection` are all gone.  Replaced with `setindex`, `setindexstack`, `quicksetindex`, `getindex`, `quickgetindex`, `quickcpy`, `compact` (which has been changed to take a more extensive array of types).
- `push` now allows maps and vecs with vectors requiring a length as the parameter and maps requiring 'void' (or nothing).
- `vec` and `map` are now typeID 6 and 7 instead of collectionTypeID 0 and 1.  Furthermore collectionTypeIDs are gone.  Furthermore as a consequence #IR_type now requires a number > 99 as the as 0-99 are reserved by the system for future collections (reserving an extreme amount as to not ever run into problems).  As this only effects non simple IR (as simple IR uses string identifiers) I feel that reserving isn't going to cause any issues.

Probably could have been 0.4 worthy in terms of what was done, but I don't feel its ready for a release yet.

## Version 0.3.1

- `dumbget` is gone
- `regobj`, `pnewobj`, `pget`, `deinit` added
- `getn`, `calln` have been renamed to `get`, and `call` respectively
- All functions that took a 'n' parameter except for array functions (i.e. `push`, `call`, `get`, ...) all don't take an 'n' parameter anymore, it was superfluous.
- Types have been removed from `get` as they weren't needed removing the need for `dumbget`
- Definition support has been proposed and is experimental in the C# compiler but isn't official.
- New attributes added;
  - `IR_obj`, `IR_set`, `IR_get`, `IR_ctor` all to allow you to define objects and use them, this is mainly so by default you can't use objects by their name and for non simple mode you define them by a number which is faster to parse (from benchmarking C#).
    - Possibility of a rollback if deemed not necessary.
  - `createType` is gone and is replaced with attribute `IR_type` functionality is the same however with the exception that in simple mode you can name the type rather than using an ID.
- Syntax change to IR when giving a list of arguments that are variadic, must be separated by a comma i.e. `push int 1, 2, 5, 9` this is to increase readability of the IR.

## Version 0.3

- New syntax which is cleaner but still resembles the old one (no need for the `;` at the start nor the `@`)
- New IR style and commands which are more efficient
- Arrays of top level objects and dictionaries of top level objects (purely syntatical sugar however)
- Allows indexing of arrays as well

### Note:

Everything below this was in an old version of DOML that no compiler supports, I'm keeping this history for the sake of it to show how far we have come however.

## Version 0.21

Rollback on the dictionaries!
- Dictionaries have been removed, it complicated the pushing code due to the requirement for two values, meaning it resulted in a general slowdown of the code across multiple versions (GO/C# being the ones tested) furthermore it was decided that a map went against the real idea of DOML and needs to be rethinked (was only ever really added due to arrays being added).

## Version 0.2

Version 0.2 didn't bring any massive changes but it did bring arrays and dictionaries.

- Arrays
  - `X.Y = [1, 2, 3]`
  - `18/pushvec (int)` command added
- Dictionaries
  - `X.Y = ["K": 1, "B": 2, "C": 3]`
  - `19/pushmap (int)` command added
- Short Function calls
  - `@ X->Y = 2` which translates to `@ _ = X.X ... .Y = 2`
    - i.e. `@ System->CallWithInt = 9` is equivalent to `@ _ = System.System ... .CallWithInt = 9`
    - The double X may be removed in the future but really it exists just to make sure you don't pollute the namespace and that you specify these calls can only exist for certain things.
      - The `_` object will only be created once despite how many calls you perform to save memory (this should be fine since its only meant to register things and no state should really be kept)
- A standidized syntax (really existed in like 0.15 or whatever)

## Pre-Version (leading up to and including) 0.1

There were many changes made here, I'm highlighting a few just to demonstrate the various ways the language has evolved, many of these changes lasted less than a week and were effectively 'litmus' tests.

- Syntax changes
  - Originally you had a `;` at the beginning of each 'set' statement (i.e. `; X.Y = 2`),
  this was done to make parsing easier and I felt it made the language simpler, it was only once I started writing out DOML I found how annoying it was to maintain the `;` and forgetting one was infuriating, so I removed this; you can still place one at the end to put multiple on the same line.
    - This is actually why the implementation of C# still treats the `;` as the start, and just allows you to also use a newline to denote the start just a side effect of this shift that hasn't been changed (due to not needing to be changed).
  - Removal of the shortcut; `; .{x = 4; y = 2; z = 9}`
    - The current way which is `.x = 4; .y = 2; .z = 9` is only more work when you can't just use `.` which is rare, and not worth introducing braces for, furthermore embedding IR uses `{ ... }` so basically it just became useless
- IR
  - So many commands have been removed, previously there were `CheckIsInt` and so on for each of the types (with ones for 8, 16, 32, and 64 bit integers along with all the sizes for the other types), there was also panic ones to produce an error, effectively they just checked you had the right number of parameters this was later done by the compiler so there was no need for commands.
- Multiple type sizes
  - From byte sized integers to 128 bit sized integers (8, 16, 32, 64, and 128), floats went from half to quad, there was characters (8 and 16), strings came in narrow and wide, decimals also had 64 and 128.
  - Removed because of the complexity, the newer system keeps the flexibility but removes all the complexity for just a small cast cost (which is relatively minor and the impact is more the increased memory usage).
- Arrays and Dictionaries were removed and explicity stated to become never a thing
  - This changed once I saw the evidence that it was very clear that arrays should exist (and same for dictionaries), I came across this evidence myself when I built the C# parser (and it was even more clear with the GO one)
  - Mainly I had declared them explicity gone to maintain simplicity, I still doubt that dictionaries really 'need' to exist, however since arrays will exist I might as well do both at the same time (they can always be removed later).
    - The reason they need to exist however is that to maintain compile time reliability we require the number of parameters (it also helps with parsing), later on most parsers will also store the types of the parameters to further help with producing compile time safety (i.e. if the code compiles the IR will 100% run with no errors unless the user's code has specific range requirements like a float between 0 and 1).  This means that you need a specific number of arguments (you can declare a negative number/null parameter list to obtain any amount but this basically just disables checking except for type), so arrays came in as a solution.
