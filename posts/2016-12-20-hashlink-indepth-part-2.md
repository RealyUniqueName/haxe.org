title: HashLink In Depth - Part 2
author: nicolas
description: Digging into the runtime
background: NVIDIA-Tegra-K1-Chip.jpg
published: true
tags: tech
---

*This is the second part of my in-depth presentation of the new Haxe target: HashLink, please read the [Part 1](https://haxe.org/blog/hashlink-indepth/)  beforehand.*

## HashLink Runtime

As we saw in the bytecode part, the "Natives table" section contains a list of C functions that are loaded from either the HL runtime or extra C libraries.

The HashLink Runtime consists of:

 - A set of C functions to manipulate Objects, Bytes, Functions, etc.
 - A complete UCS2 String API with conversions from/to UTF8
 - A custom Garbage Collector which will perform allocation and regular memory collection
 - Support for exceptions and stack traces
 - Full system API (Haxe `Sys` class and `sys` package) with file-system I/O, networking, etc.
 - Math functions, etc.

The HL run-time is compiled as a native dynamic library, named `libhl` (using the platform native extension - `.dll` on Windows, `.so` on Linux, etc.).

At the moment, the JIT compiler and byte-code reader are not part of the HL but of the virtual machine (`hl` executable).

## HL/C

The separation between the VM and the run-time is necessary in order to allow the translation of HL byte-code to C.

Because our byte-code is quite low level and strictly typed, it is easy to translate each *opcode* to the corresponding C statement with the exact same semantics. This can be done with theHaxe compiler by compiling to a C file instead of a `.hl` byte-code file:

`haxe -hl main.c -main Main`

This will output the following C file:

```haxe
// main.c
// Generated by HLC 3.3.0 (HL v1)
#define HLC_BOOT
#include <hlc.h>

// Types definitions
typedef struct _hl__types__BaseType *hl__types__BaseType;
typedef struct _hl__types__Class *hl__types__Class;
typedef struct _String *String;
...

// Types implementation
struct _hl__types__BaseType {
	hl_type *$type;
	hl_type* __type__;
	vdynamic* __meta__;
	varray* __implementedBy__;
};
....

// Globals
static hl__types__$BaseType global$0 = 0;
static $String global$1 = 0;
static hl__types__Class global$2 = 0;
...

// Natives functions
HL_API varray* hl_alloc_array(hl_type*,int);
HL_API bool hl_sys_utf8_path();
HL_API void hl_bytes_blit(vbyte*,int,vbyte*,int,int);
...

// Functions declaration
static void Main_main();
static String Std_string(vdynamic*);
static vdynamic* Std___add__(vdynamic*,vdynamic*);
static String String_fromCharCode(int);
....

// Strings
static vbyte string$0[] = {0,0} /*  */;
static vbyte string$1[] = {104,0,108,0,46,0,116,...} /* hl.types.Class */;
static vbyte string$2[] = {104,0,108,0,46,0,116,0,121,0,112,0,101,0,115,...} /* hl.types.BaseType */;
....

// Functions code
static void Main_main() {
	String r2;
	int r3;
	vbyte *r1;
	r2 = (String)hl_alloc_obj(String__val);
	r1 = string$33;
	r2->bytes = r1;
	r3 = 11;
	r2->length = r3;
	Sys_println(((vdynamic*)r2));
	return;
}
....
```

I have cut some parts that are related to type signatures and closures but, as you can see, if you have read the **HashLink Bytecode** section, this is very similar to what the byte-code does. The only difference is that instead of being loaded by the HL VM and run using JIT, the "byte-code" will be compiled by a fully optimizing C compiler, increasing the speed even more! And because both HL/JIT and HL/C share the same run-time (same garbage collector, same native functions, etc.), they will run exactly the same without any difference in terms of semantics.

Once you have your C code, you simply have to compile it with the `hlc.h` `hl.h` and `hlc_main.c` files which are all present in the `src` directory of [HashLink repository](http://github.com/HaxeFoundation/hashlink) and link it to the HL-run-time. One way of doing this is using GCC:

```
gcc -o myApp -O3 -std=c11 -I hl/src main.c hl/src/hlc_main.c -lhl -lm
```

Important Notes:

 - if you get an error `pasting "u" and ""Null access"" does not give a valid preprocessing token` it means you forgot to add `-std=c11`. We only use C11 for unicode string literals support but it is necessary to compile HL run-time and HL/C code.

- at the moment, the HL/JIT only supports x86 and not x86-64, which would require building a 32 bit version of the HL run-time to use it. This is done by using `make ARCH=32` in HL sources. But if you want to link to this version of the run-time, you need to add `-m32` to your gcc compilation options. Another option would be to have two compiled versions of the run-time: a 32 bit version for HL/JIT and a 64 bit for HL/C. I hope to add 64 bit JIT support in the upcoming months.

## HL Type System

HashLink has its own low level types that are used for registers and function parameters. Haxe types are represented using the following low level types:

### HashLink basic types and their memory representation:

  * `void` not really a value, used for typing purpose
  * `ui8` an unsigned 8 bits integer (0-255)
  * `ui16` and unsigned 16 bits integer (0-65535)
  * `i32` a signed 32 bits integer (-2147483648-2147483647)
  * `bool` a boolean (true or false)
  * `f32` 32 bits single precision IEEE floating point
  * `f64` 64 bits double precision IEEE floating point

All the following values are memory addresse pointers and takes either 4 bytes in x86 mode or 8 bytes in x86-64 mode:

  * `bytes` raw bytes (similar to C `char*`) can be freely read/written to without any kind of bounds check
  * `dyn` a dynamic value, can contain any other value, its runtime type can be known with `gettype` opcode
  * `ret fun(args)` functions are strictly typed in HL. Can be a closure or a direct function
  * `array` an array of values. Low level arrays are not strictly typed and not bounds checked, and they have a fixed length
  * `#object` a fixed object type with single inheritance, see below
  * `dynobj` a dynamic object which we can add or remove fields at will (see below)
  * `virtual(fields...)` a virtual interface to an underlying `#object` or `dynobj` (see below)
  * `enum(name)` an enum value, can have different constructor with optional parameters (see below)
  * `ref(T)` a memory reference of type T, which can be read or written
  * `null(T)` a value of type T that can be `null` (T being a basic type)
  * `type` a value which represent an HL type, any type in HL is also a value
  * `abstract(name)` an abstract value, usually made accessible from a C interface

### Memory storage

Memory consumption in HL is identical to C, and might depend on the C compiler you are using:

 * `ui8` will take 1 byte, `ui16` 2 bytes,  `i32` and `f32` 4 bytes and `f64` 8 bytes
 * depending on the C compiler `bool` might take 1 byte or 4 bytes in memory
 * object data will be aligned on each member bytes size, so for instance `f64` will always be aligned on 8 bytes. 

### Boxing

Any Function, Object, Virtual, Array, Null and DynObj can be assigned to `dyn` without requiring any allocation since they carry their runtime type in their first memory address. All other types (basic types as well as bytes, type, ref, abstract and enum) requires boxing allocation when cast to dynamic (using `todyn` opcode)

### Matches between Haxe and HashLink:

 * Haxe Void is HL `void`
 * Haxe Int is HL `i32`
 * Haxe Bool is HL `bool`
 * Haxe String is HL `#String` (as with all other objects)
 * Haxe Float is HL `f64`
 * Haxe Single is HL `f32`
 * Haxe Dynamic is HL `dyn`
 * Haxe anonymous structure or interface instances are HL `virtual(...)` (see bellow)

## Objects

There are two kind of objects in HL: static objects and dynamic objects.

Static objects are similar to classes in Haxe. Let's look at the following Haxe class:

```haxe
class Point extends Geometry {
     public var x : Float;
     public var y : Float;
     public function new() {}
     function add( p2 : Point ) { ... }
     static function alloc(x,y) { ... }
}
```

Here's the dump for the HL definition of the class as a static object:

```
Point @658
        extends Geometry
	2 fields
	  @0 x f64
	  @1 y f64
	2 methods
	  @0 add fun@456
	  @1 toString fun@82[0]
```

This data consists of:

  * a specific name: `Point`
  * an optional super-class which from which we will inherit both fields and methods - `Geometry` in this example
  * a list of strictly typed fields: `x` and `y` both declared as being `f64`
  * a list of methods and their function indexes `add` and `toString` - some methods will override others or will be overridden in sub-classes so they need an override slot: here `toString` overrides slot 0
  * an optional global table index which can be retrieved and is used to store the static class value - here `@658`. It will give us access to an object of type `#$Point` which is the the class point value which extends `hl.types.Class` and will have a field `alloc`

Given that we know all the fields of the static objects and their order, compiling a field access can be done very efficiently by simply reading at the correct memory offset.

### DynObj
 
While static objects are very fast, they lack the ability to create more fields or delete them. Instead, the `dynobj` type can be used to perform dynamic field allocation. Dynamic objects are a lot more flexible than static objects but they are also a lot slower so they are not always suitable for the implementation of Haxe features such as [anonymous structures](https://haxe.org/manual/types-anonymous-structure.html).

Dynamic objects are implemented as sorted arrays so searching a field have a run-time of O(log n) using the hashed representation of the field.

### Virtuals

Virtuals are used to implement both [anonymous structures](https://haxe.org/manual/types-anonymous-structure.html) and interfaces in Haxe.

A virtual consists of a strictly typed list of references to an underlying object field. This underlying object can be a static object, a dynamic object, or some space allocated into the virtual itself.

#### Virtuals of Static Object

This creates a virtual from static object point:

```haxe
var o : { x : Float, y : Float } = new Point();
```

The virtual type will be `virtual(x:f64,y:f64)`, which measn the virtual will store:
 
 * a reference to the `Point` object, so if cast to it is made, its value can be retrieved 
 * references to the addresses of the `x` and `y` values inside the `Point` value, so we can read and write them with an extra indirection

In that case (static object to virtual), a virtual is allocated every time this kind of operation occurs. this means that it is a  bit of an expensive operation but these cases are rarer.

In the special cases of **interfaces** which are also virtuals of static objects, each class has an extra field per interface used to store the allocated virtual interface when it is required, so allocation will only occur once - when requested.

#### Virtuals of Dynamic Objects

Let's look at the following example:

```haxe
var o : { x : Float, y : Float } = haxe.Json.parse('{ "x" : 5.3 }');
```

The JSON parser will allocate a dynamic object with only a single field `x`. Casting to virtual, we will allocate a virtual with:

 * a reference to the underlying `dynobj`
 * a reference to the `x` value inside the `dynobj` data
 * a `NULL` reference for the `y` field

Since there is a `NULL` reference to `y`, every access to it - either to read or write - will be performed dynamically. In case of writing, this will create the field in the `dynobj` and update the reference in the virtual table.

Given that the form of the `dynobj` can change, each `dynobj` stores the list of virtuals it has been casted to and will update their field references as fields get added / removed or have their type changed. Thus, only the first dynobj-to-virtual cast will perform an allocation.

#### Compact Virtuals

When we already know the list of fields but they *might* increase if accessed through reflection, we allocate a virtual with some extra space to store the data for these fields.  Look at the following example:

```haxe
var o = { x : 1.5, y : -1.5 };
```

This will allocate a virtual with:

 * no reference to any underlying object, but extra data to store the two `f64` (16 bytes)
 * references to the addresses of the `x` and `y` value inside our own virtual

This is not as fast as a static object since it requires one extra indirection for reading/writing the fields and takes a bit more memory (one memory address per field) but it still gives very good performance.

We cannot use a static object in that case, because later in the code on might do:

```haxe
Reflect.setField(o,"y","BAD!");
```

In that case, we will:

 * allocate a new `dynobj` which copies our current virtual data
 * store it as a reference in our virtual
 *.update the dynobj by changing the `y` field from being an `f64` to being a `#String`

 As a result, this will set our reference to `y` in our virtual to `NULL`

 * the next read of `y` on the virtual will then be performed dynamically, which will cast the String to f64
 * the next write to `y` will change back the dynobj `y` field to being an `f64` and update the virtual reference to it
 * When a "compact virtual" has mutated into a virtual of dynobj, it continues to occupy the now unused extra data space that was allocated for it (16 bytes in our example).

## Enums

Enums are represented by a list of constructors with per-constructor fields, for example:

```haxe
enum MyEnum {
    A;
    B;
    C( e : String );
    D( x : Int, y : Int );
    E( v : MyEnum );
}
```

This defines 5 constructors for `enum(MyEnum)`:
 
 * constructor `A` with index 0 and no extra field
 * constructor `B` with index 1 and no extra field
 * constructor `C` with index 2 and one extra field of type `#String`
 * constructor `D` with index 3 and two extra fields of type `i32`
 * constructor `E` with index 4 and one extra field of type `enum(MyEnum)`

When allocating an enum value we specify the constructor index an the field values. This wazy,  HL knows exactly how much memory space to allocate to store these values.

The constructor index can be read using `getenumindex` opcode and fields can be accessed by specifying both the constructor and field indexes. The correctness of this access is guaranteed by the Haxe Compiler's **switch** expression typing.

As a result, an enum only takes 4 bytes plus its extra field data in terms of memory. Constructors with no extra fields are constant so they are only allocated once.

## Bytecode Reference

As a complementary to previous part, here's a list of HashLink opcodes:

`mov [dst], [src]` dst := src (copy src register value to dst register)

**Constant loading**

`int [dst], @[index]` store int value at specified index in Int table into dst register

`float [dst], @[index]` store float value at specified index in Int table into dst register

`string [dst], @[index]` store the UCS2 string at specified index in String table into dst register

`bytes [dst], @[index]` store the raw UTF8/Binary bytes at specified index in String table into dst register

`true [dst]` dst := true (store `true` boolean into dst register)

`false [dst]` dst := false (store `false` boolean into dst register)

`null [dst]` dst := null (store `null` value into dst register)

**Numerical Operations**

`add [dst], [a], [b]` dst := a + b (numeric types only)

`sub [dst], [a], [b]` dst := a - b (numeric types only)

`mul [dst], [a], [b]` dst := a * b (numeric types only)

`sdiv [dst], [a], [b]` dst := a / b (numeric types only) signed mode

`udiv [dst], [a], [b]` dst := a / b (integer types only) unsigned mode

`smod [dst], [a], [b]` dst := a % b (numeric types only) signed mode

`umod [dst], [a], [b]` dst := a % b (integer types only) unsigned mode

**Bitwise Operations**

`shl [dst], [a], [b]` dst := a << b (integer types only)

`sshr [dst], [a], [b]` dst := a >> b (integer types only) signed mode

`ushr [dst], [a], [b]` dst := a >>> b (integer types only) unsigned mode

`and [dst], [a], [b]` dst := a & b (integer types only)

`or [dst], [a], [b]` dst := a OR b (integer types only)

`xor [dst], [a], [b]` dst := a ^ b (integer types only)

** Other Operations**

`neg [dst], [a]` dst := -a

`not [dst], [a]` dst := !a (bool only)

`incr [dst]` dst := dst + 1

`decr [dst]` dst := dst - 1

** Calls **

`call [dst], FunctionName([args])` call the static function with registers `args` and store the result in `dst`

`callmethod [dst], [obj][field]([args])` call the `obj` prototype method stored at index `field` with registers `args` and store the result in `dst`

`callclosure [dst], [func]([args])` call the function stored in register `func` with registers `args` and store the result in `dst`

`callthis [dst], [field]([args])` same as callmethod, but assume obj = register 0

**Functions**

`staticclosure [dst], FunctionName` creates a closure from the given function and store it in `dst`

`instanceclosure [dst], FunctionName([obj])` creatures a closure by applying the first argument of the function to `obj` and store the function with the remaining arguments in `dst`

`virtualclosure [dst], [obj][field]` same as instance closure, but fetch the method from the object prototype instead of being a static function

`setmethod [obj][field], FunctionName` initialize a static method into an class

** Globals **

`getglobal [dst], [index]` dst := GLOBALS[index] (load the global value at `index` into `dst`)

`setglobal [index], [r]` GLOBALS[index] := r (store `r` into the global value at `index`)

** Control Flow**

Offsets in jumps are always expressed in number of opcodes, starting from the opcode just after the jump.

`ret [r]` return with the register `r` value

`jtrue [r], [offset]` jump by `offset` opcodes if register `r` is `true`

`jfalse [r], [offset]` jump by `offset` opcodes if register `r` is `false`

`jnull [r], [offset]` jump by `offset` opcodes if register `r` is `null`

`jnotnull [r], [offset]` jump by `offset` opcodes if register `r` is not `null`

`jslt [a], [b], [offset]` jump by `offset` opcodes if register values comparison `a < b` (numerical types only) signed mode

`jsgt [a], [b], [offset]` jump by `offset` opcodes if register values comparison `a > b` (numerical types only) signed mode
 
`jslte [a], [b], [offset]` jump by `offset` opcodes if register values comparison `a <= b` (numerical types only) signed mode

`jsgte [a], [b], [offset]` jump by `offset` opcodes if register values comparison `a >= b` (numerical types only) signed mode

`jult [a], [b], [offset]` jump by `offset` opcodes if register values comparison `a < b` (integer types only) unsigned mode

`jugte [a], [b], [offset]` jump by `offset` opcodes if register values comparison `a >= b` (integer types only) unsigned mode

`jeq [a], [b], [offset]` jump by `offset` opcodes if register values comparison `a == b`  (numerical types only)

`jnoteq [a], [b], [offset]` jump by `offset` opcodes if register values comparison `a != b` (numerical types only)

`jalways [offset]` always jump by `offset`

`label` necessary to indicate a loop starting point. jumps with negative offsets must always target a label

**Conversion**

`todyn [dst], [r]` dst := Dynamic(r) - boxing of a basic type

`tosfloat [dst], [r]` dst := Float(r) (signed mode)

`toufloat [dst], [r]` dst := Foat(r) (unsigned mode)

`toint [dst], [r]` dst := Int(r) (signed mode)

**Object access**

Object field accesses are not null checked in HL but the Haxe compiler will insert all the necessary `nullcheck`s. As a result, if a variable is not modified and accessed for several fields, only the first access will perform a null-check.

`new [dst]` dst := new object (allocate a new object uninitialized object into `dst`) 

`nullcheck [r]` throw an exception if `r` is `null`

`field [dst], [obj][field]` read the field of the register `obj` at the specified index and store it into the `dst` register 

`setfield [obj][field], [r]` write the field of the register `obj` at the specified index with the register `r`'s current value

`getthis [dst], [field]` same as getfield with register 0 as obj

`setthis [field], [r]` same as setfield with register 0 as obj

`dynget [dst], [obj][f]` read the field of the register `obj` the name of which is stored in the register `f` into `dst` register

`dynset [obj][f], [r]` write the field of the register `obj` the name of which is stored in the register `f` with the register `r`'s current value

**Exceptions**

`throw [r]` throw the exception value in register `r`

`rethrow [r]` same as throw but keep the previous exception stack trace

`trap [r], [offset]` setup a try/catch, will store the exception in register `r` and jump by the offset if an exception occurs

`endtrap` end the latest trap section

**Bytes Access**

All byte accesses in HL are low level which means that there are no bound checks or null checks being performed. This will give the same speed as doing a pointer access in C. All accesses are performed in native CPU endianness.

`getui8 [dst], [bytes][pos]` read 8 bits unsigned integer in register `bytes` at index register `pos` and store it into `dst`

`getui16 [dst], [bytes][pos]` read 16 bits unsigned integer in register `bytes` at index register `pos` and store it into `dst`

`geti32 [dst], [bytes][pos]` read 32 bits integer in register `bytes` at index register `pos` and store it into `dst`

`getf32 [dst], [bytes][pos]` read 32 bits float in register `bytes` at index register `pos` and store it into `dst`

`getf64 [dst], [bytes][pos]` read 64 bits double in register `bytes` at index register `pos` and store it into `dst`

`setui8 [bytes][pos], [r]` stores 8 bits unsigned integer in register `r` into register `bytes` at index register `pos`

`setui16 [bytes][pos], [r]` stores 16 bits unsigned integer in register `r` into register `bytes` at index register `pos`
 
`seti32 [bytes][pos], [r]` stores 32 bits integer in register `r` into register `bytes` at index register `pos`

`setf32 [bytes][pos], [r]` stores 32 bits float in register `r` into register `bytes` at index register `pos`

`setf64 [bytes][pos], [r]` stores 64 bits double in register `r` into register `bytes` at index register `pos`

**Array Access**

Array accesses are unsafe in HL in the sense that there is a single `array` type but you can read any type of value from it. No bound checks are performed either.

`getarray [dst], [array][pos]` read the value at register `pos` into register `array` and store it into register `dst`

`setarray [array][pos], [r]` store the value of register `r` into register `array` at register `pos`

`arraysize [dst], [array]` read the `array` size into register `dst`

**Coercions**

`safecast [dst], [r]` cast register `r` into register `dst`, throw an exception if there is no way to perform such operation 

`unsafecast [dst], [r]` store register `r` into register `dst` of a different type without any runtime cast (might crash the program at a later point in case the type was not valid)

`tovirtual [dst], [r]` converts object or virtual value in register `r` to the `dst` virtual  

**Types**

`type [dst], T` dst := T, store the type T into register `dst` 

`gettype [dst], [r]` dst := typeof(r), extract the type from current value `r` into `dst`

`gettid [dst], [r]` store the type kind integer identifier of type value `r` into `dst`

**References**

`ref [dst], [r]` dst := &r, store a reference to register `r` into `dst` 

`unref [dst], [r]` dst := *r, read reference at address `r` into `dst` 

`setref [r], [v]` *r := v, store value in register `v` into reference at address `r`

**Enums**

`makenum [dst], CID, [args...]` create an enum with constructor CID and value in registers `args` and store it into `dst`

`enumalloc [dst], CID` create an enum with constructor CID with all values as default and store it into `dst`

`enumindex [dst], [r]` store into register `dst` the enum constructor index of register `r`

`enumfield [dst], [r], CID, FID` read the field at index `FID` of constructor `CID` of enum in register `r` into `dst`

`setenumfield [e], FID, [r]` store the value in register `r` into the field at index `FID` (of constructor index 0) of enum value in register `e` 

**Misc**

`switch [r] [offsets] [end]` jump by an offset depending on the int value of register `r` or continue if `r` is outside the range `end` mark the end of the last case of the `switch`

`nop`  do nothing, used by optimized to tag some removed opcodes