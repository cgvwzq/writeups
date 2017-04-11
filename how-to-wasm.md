# 0x0 What is WebAssembly?

"WebAssembly or WASM is a new, portable, size- and load-time-efficient format suitable for compilation to the web."

Is currently being designed as an open [standard](https://github.com/WebAssembly/spec) by the [W3C group](https://www.w3.org/community/webassembly/) that includes representatives from all major browsers.

As of April 2017 both Chrome 57 and Firefox 52 (not tried other browsers) support version [0x1](https://github.com/WebAssembly/design/pull/1006), which implements most of the [MVP](https://github.com/WebAssembly/design/blob/master/MVP.md).

# 0x1 hello_world.c

WebAssembly is primarily a compiler target, hence it is not supposed to be coded directly.

AFAIK there are 2 main paths for C/C++ compilation (probably many more for other languages):

* Use [Emscripten](https://github.com/kripken/emscripten) to generate C/C++ -> LLVM -> asm.js, and [bynarien](https://github.com/WebAssembly/binaryen/)'s `asm2wasm` to generate the WebAssembly.
* Or use LLVM's experimental support for WebAssembly backend (need to build CLANG with `LLVM_EXPERIMENTAL_TARGETS_TO_BUILD=WebAssembly`) to compile C/C++ into LLVM's IR, and then bynarien's `s2wasm` into WebAssembly.

Let's write our first `hello.c`:
```c
int main(void) {
    printf("hello wasm\n");
    return 0;
}
```

Compile:

```
$ clang -emit-llvm --target=wasm32 -S hello.c
$ llc helo.ll -march=wasm32
$ s2wasm hello.s > hello.wast
```

This is how our output looks like:

```wasm
(module
 (type $FUNCSIG$ii (func (param i32) (result i32)))
 (type $FUNCSIG$iii (func (param i32 i32) (result i32)))
 (import "env" "printf" (func $printf (param i32 i32) (result i32)))
 (table 0 anyfunc)
 (memory $0 1)
 (data (i32.const 16) "hello wasm\n\00")
 (export "memory" (memory $0))
 (export "main" (func $main))
 (func $main (result i32)
  (local $0 i32)
  (i32.store offset=4
   (i32.const 0)
   (tee_local $0
    (i32.sub
     (i32.load offset=4
      (i32.const 0)
     )
     (i32.const 16)
    )
   )
  )
  (i32.store offset=12
   (get_local $0)
   (i32.const 0)
  )
  (drop
   (call $printf
    (i32.const 16)
    (i32.const 0)
   )
  )
  (i32.store offset=4
   (i32.const 0)
   (i32.add
    (get_local $0)
    (i32.const 16)
   )
  )
  (i32.const 0)
 )
)
```

Ok, we see a text format file with a bunch of `s-expressions`. This format is called WAST, which [seems](https://github.com/WebAssembly/design/blob/master/TextFormat.md) to be a superset of another text format named WAT, anyway, we can use yet-another-tool from the [WABT "WebAssembly Binary Toolkit"](https://github.com/WebAssembly/wabt) (pronunced "wabbit") to translate into a binary format:

```
wast2wasm hello.wast -o hello.wasm
```

This finally gives us a real WASM file. Unfortunately, this will not work since we still need to generate the `env` environment that emulates most libc features in JS. In this post we will not care about this, but it is important to realize that right now WebAssembly only allows an isolated computation environment and all interactions with the world are done through JS imports/exports and FFI (although this can change in the [future](https://github.com/WebAssembly/design/blob/master/GC.md)).

# 0x2 WebAssembly APIs

Let's take a quick look on the [WebAssembly API](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly) provided by web browsers:

```js
> Object.getOwnPropertyNames(WebAssembly)
["compile", "validate", "instantiate", "Module", "Instance", "Table", "Memory", "CompileError", "LinkError", "RuntimeError"]
```

The main WASM object is called `Module` and it is a stateless WebAssembly code. We can generate a module compiling a binary.

On the other hand, an `Instance` is a stateful executable instance of a module that will allow calling WASM functions from Javascript, or executing the `start` function (aka main or entrypoint).

As a rule of thumb, lower cases are asynchronous functions returning promises, and the constructors starting with a capital letter are synchronous. Check the MDN for more details.

# 0x3 The funny way

See the [rationale](https://github.com/WebAssembly/design/blob/master/Rationale.md#why-a-binary-encoding) on why using a binary encoding, it essentially provides size reduction (thanks to TLV and LEB128 for variable-length integer storage) and allows to perform real-time streamed compilation. Pretty cool.

You can see all the details of the binary encoding as well as the instruction set [here](https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md).

## Demos

That's enough theory, let's see a few minimal demos :)

### Minimal module

This is the shortest valid module:

```wasm
(module)
```

You can use `wast2wasm` to generate the binary (or write it by hand), and try this in your console:

```js
let bin = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d, // magic bytes "\0" "asm"
    0x01, 0x00, 0x00, 0x00, // version 0x1
)];

let m = WebAssembly.Module(bin); // let's use the sync version, though it's not recommended for large modules
console.log(m);
let i = WebAssembly.Instantiate(m); // the module does nothing but allocating its memory region
console.log(i);

```

### Functions

Now we are going to write a function that takes a parameter and returns it:

```wasm
(module
    (func $foo (param i32) (result i32)
        get_local 0
        return)
)
```

This module will need 3 dfferent sections: an entry in the Type section containing the function signature, another entry in the Function section with the function declaration, and a last one entry in the Code section with the actual bytecode:

```js
let bin = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00, // magic bytes + version
    0x01, // Type section
        0x06, // section length
        0x01, // number of entries
            0x60, 0x01, 0x7f, 0x01, 0x7f, // {form:func, param_count:1, param_types:[i32], return_count:1, return_type:i32}
    0x03, // Function section
        0x02, // section length
        0x01, // number of entries
            0x00, // signature index = 0 (ref to Type section)
    0x0a, // Code section
        0x07, // section length
        0x01, // number of entries
            0x05, // body size
            0x00, // numer of local variables
            0x20, 0x00, 0x0f, // get_local; return
            0x0b, // end
]);

WebAssembly.compile(bin).then(WebAssembly.instantiate).then(console.log); // async version
```

Note that we can also pass the buffer directly to `WebAssembly.instantiate` and that promise would return both the `Module` and the `Instance` generated, but sometimes we may want to compile only the module in order to instantiate it later or to transfer it into another environment, like a Web Worker.

### Start

Ok, the examples above are good, but despite these cute binary arrays we are NOT executing any Web Assembly code. In order to do that we need to mark a function (that has no parameters and returns no value) as entrypoint, and this is done in the Start section. Let's see:

```wasm
(module
    (func $main)
    (start $main)
)
```

Here we define an empty function and mark it as entrypoint:

```js
let bin = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00, // magic bytes + version
    0x01, // Type section
        0x04, // section length
        0x01, // number of entries
        0x60, 0x00, 0x00, // {form: func, param_count:0, return_count:0}
    0x03, // Function section
        0x02, // section length
        0x01, // number of entries
        0x00, // signature index = 0 (ref to Type section)
    0x08, // Start section
        0x01, // section length
        0x00, // function index (ref to Function space)
    0x0a, // Code section
        0x04, // section length
        0x01, // number of entries
            0x02, // body length
            0x00, // number of local variables
            0x0b, // end
]);

WebAssembly.instantiate(bin).then(console.log);
```

Now we have executed an empty function. Yeah... Not to impressive yet.

### Import

Now that we know how to define functions and execute stuff, let's try to interact with the real world (or the Javascript realm). In this example we are going to import the `console.log` function and call it with a parameter. Right now WASM only works with integers and floats (may change int the near future with `TypedObjects`), so it's not possible to do more fancy stuff directly.

```wasm
(module
    (import "console" "log" (func $log (param i32)))
    (func $main
        i32.const 1337
        call $log)
    (start $main)
)
```

The code above needs 4 sections:

```js
let bin = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00, // magic bytes + version
    0x01, 0x08, 0x02, 0x60, 0x01, 0x7f, 0x00, 0x60, 0x00, 0x00, // Type section (note that now we have 2 signatures ()->nil and (i32)->nil )
    0x02, // Import section
        0x0f, // section length
        0x01, // number of entries
        0x07, 0x63, 0x6f, 0x6e, 0x73, 0x6f, 0x6c, 0x65, 0x03, 0x6c, 0x6f, 0x67, 0x00, 0x00, // {
    0x03, 0x02, 0x01, 0x01, // Function section (ref to Type section, now $main's index=1)
    0x08, 0x01, 0x01, // Start section (ref to Function space (index=1), imported functions take lower indexes)
    0x0a, // Code section
        0x09, // section length
        0x01, // number of entries
            0x07, // body size
            0x00, // number of local variables
            0x41, 0xb9, 0x0a, 0x10, 0x00, // i32.const 1337; call 0
            0x0b, // end
]);

WebAssembly.instantiate(bin, {console});
```

Check out the console. Cool! Now we have a real proof that the code has been executed :D

### Export

In a similar way we can export WASM objects to be use from the JS realm. In the next example we define and export a function that calculates the factorial:

```wasm
(module
    (export "factorial" (func $fact))
    (func $fact (param $n i32) (result i32)
        (i32.eq (get_local $n) (i32.const 0))
        if i32
            (i32.const 1)
        else
            (i32.mul
                (get_local $n)
                (call $fact (i32.sub (get_local $n) (i32.const 1))))
        end
    )
)
```

The code above will produce the following binary:

```js
let bin = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00, // magic bytes + version
    0x01, 0x06, 0x01, 0x60, 0x01, 0x7f, 0x01, 0x7f, // Type section
    0x03, 0x02, 0x01, 0x00, // Function section
    0x07, // Export section
        0x0d, // section length
        0x01, // number of entries
            0x09, // field_lenght
            0x66, 0x61, 0x63, 0x74, 0x6f, 0x72, 0x69, 0x61, 0x6c, // field_str "factorial"
            0x00, // kind (0 is Func)
            0x00, // index = 0
    0x0a, // Code section
        0x19, // section length
        0x01, // number of entries
            0x17, // body length
            0x00, // number of locals
                0x20, 0x00, // get_local 0
                0x41, 0x00, // i32.const 0
                0x46,       // i32.eq
                0x04, 0x7f, // if
                0x41, 0x01, //   i32.const 1
                0x05,       // else
                0x20, 0x00, //   get_local 0
                0x20, 0x00, //   get_local 0
                0x41, 0x01, //   i32.const 1
                0x6b,       //   i32.sub
                0x10, 0x00, //   call 0
                0x6c,       //   i32.mul
                0x0b,       // end
                0x0b, // end
]);

WebAssembly.instantiate(bin).then( _ => {
    let wasmFactorial = _.instance.exports.factorial;
    console.log(wasmFactorial(7));
});
```

### Memory

WebAssembly instances have a linear memory section to interact with through the `load`/`store` operators. In the MVP there may be at most one linear memory, imported or defined. This memory is defined in the Memory section with an initial size (multiple of 64Kb pages) and, optionally, a maximum size. The memory size can be programatically queried or extended with the `current_memory` and `grow_memory` instructions, respectively.

This is a module declaration with an empty memory section:

```wasm
(module
    (memory 10 20)
)
```

That produces the following binary:

```js
let bin = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00, // magic bytes + version
    0x05, // Memory section
        0x04, // section length
        0x01, // number of entries
            0x01, 0x0a, 0x14, // {flag: 1, initial:10, max:20} (flag specifies if max field is present)
]);
```

We can import and export memories. In both cases we need/get a `WebAssembly.Memory` object containing a `buffer`. For example:

```
var mem = WebAssembly.Memory({initial: 10, max: 20});
var b = new Uint8Array(mem.buffer);
```

This will give us a buffer of 640Kb that can be passed into a WASM instance.

### Data

The initial contents of linear memory are zero. This section declares the intialized data that is loaded into the linear memory and it is analogous to the `.data` section in native binaries.

```wasm
(module
    (memory 1)
    (data (i32.const 0) "hi there! i'm in the data section\n")
)
```

If you remember the hello world example at the beginning, the strings were in the Data section. But let's see the binary output of this snippet:

```js
let bin = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00, // magic bytes + version
    0x05, // Memory section
        0x03, // section length
        0x01, // number of entries
            0x00, 0x01, // {flag:0, initial:1}
    0x0b, // Data section
        0x27, // section length
        0x01, // number of entries
            0x00, // memory index (in MVP only 1 memory, so index=0)
            0x41, 0x00, 0x0b, // offset (calculate dynamically, bytecode for i32.const 0)
            0x21, // size
            0x74, 0x68, 0x69, 0x73, 0x20, 0x69, 0x73, 0x20, 0x74, 0x65, 0x78, 0x74, 0x20, 0x69, 0x6e, 0x20, 0x74, 0x68, 0x65, 0x20, 0x64, 0x61, 0x74, 0x61, 0x20, 0x73, 0x65, 0x63, 0x74, 0x69, 0x6f, 0x6e, 0x0a // data = "hi there! ..."
]);
```

## Tools

In order to diswassembly the binary format we can use the WABT tools: `wasm2wast` will output something similar to the plaintext, and we can also use `wasmdump` to see more detailed information about sections or the diswassembly.

Most web browsers are already giving debugging support in their devtools, so we can work directly with the text format, set breakpoints, and so on.

Additionally, I've started adding some initial support for [radare2](https://github.com/radare/radare2/pull/7220).

# 0xb

Ok, I think this is enough for an introduction. There are a few more sections that we haven't seen like the Global, Table and Element, but you can get an idea with the current documentation.

We also have "custom sections" that are intended to be used for debugging information, future evolution, or third party extensions. Currently the MVP only defines the [Name section](https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md#name-section), which can contain the names of functions and variables to facilitate debugging.

In any case WASM has arrived to stay, and we'll probably start seeing it in many places apart from the web as soon as many new features are added.

Cheers! :D
