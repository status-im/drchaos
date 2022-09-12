# drchaos

Fuzzing is an automated bug finding technique, where randomized inputs are fed to a target
program in order to get it to crash. With fuzzing, you can increase your test coverage to
find edge cases and trigger bugs more effectively.

drchaos extends the Nim interface to LLVM/Clang libFuzzer, an in-process,
coverage-guided, evolutionary fuzzing engine. And adds support for
[structured fuzzing](https://github.com/google/fuzzing/blob/master/docs/structure-aware-fuzzing.md).
The user should define the input type, as a parameter to the target function and the
fuzzer is responsible for providing valid inputs. Behind the scenes it uses value profiling
to guide the fuzzer past these comparisons much more efficiently than simply hoping to
stumble on the exact sequence of bytes by chance.

## Usage

For most cases, it is fairly trivial to define a data type and a target function that
performs some operations and checks if the invariants expressed as assert conditions still
hold. See [What makes a good fuzz target](https://github.com/google/fuzzing/blob/master/docs/good-fuzz-target.md)
for more information. Then call `defaultMutator` with that function as parameter. That fuzz target can be as basic as
defining a fixed-size type and ensuring the software under test doesn't crash like:

```nim
import drchaos

proc fuzzMe(s: string, a, b, c: int32) =
  # function under test
  if a == 0xdeadc0de'i32 and b == 0x11111111'i32 and c == 0x22222222'i32:
    if s.len == 100: doAssert false

func fuzzTarget(data: (string, int32, int32, int32)) =
  let (s, a, b, c) = data
  fuzzMe(s, a, b, c)

defaultMutator(fuzzTarget)
```

> **WARNING**: Fuzz targets must not modify the input variable. This can be ensured by using `.noSideEffect`
> and {.experimental: "strictFuncs".}

Or complex as shown bellow:

```nim
import drchaos

type
  ContentNodeKind = enum
    P, Br, Text
  ContentNode = object
    case kind: ContentNodeKind
    of P: pChildren: seq[ContentNode]
    of Br: discard
    of Text: textStr: string

func `==`(a, b: ContentNode): bool =
  if a.kind != b.kind: return false
  case a.kind
  of P: return a.pChildren == b.pChildren
  of Br: return true
  of Text: return a.textStr == b.textStr

func fuzzTarget(x: ContentNode) =
  # Convert or translate `x` to any format (JSON, HMTL, binary, etc...)
  # and feed it to the API you are testing.

defaultMutator(fuzzTarget)
```

drchaos will generate millions of inputs and run `fuzzTarget` under a few seconds.
More articulate examples, such as fuzzing a graph library are in the `examples/` directory.

Defining a `==` proc for the input type is necessary. `proc default(_: typedesc[T]): T` can also
be overloaded. Which is especially useful when `nil` for `ref` is not an acceptable value.

### Needed config

Add these flags to either `<filename>.nims`, `config.nims`, `nim.cfg` or directly pass to the compiler:

```nim
--cc: clang
--define: useMalloc
--noMain: on
--define: noSignalHandler
--passC: "-fsanitize=fuzzer,address,undefined"
--passL: "-fsanitize=fuzzer,address,undefined"
#--define: release
--debugger: native
```

Alternatively, drchaos provides structured input for fuzzing with [nim-testutils](https://github.com/status-im/nim-testutils)
Which includes a convenient [testrunner](https://github.com/status-im/nim-testutils/blob/master/testutils/readme.md)

### Post-processors

Sometimes it is necessary to adjust the random input in order to add magic values or
dependencies between some fields. This is supported with a post-processing step, which for
performance and clarity reasons only runs on compound types such as
object/tuple/ref/seq/string/array/set and by exception distinct types.

```nim
proc postProcess(x: var ContentNode; r: var Rand) =
  if x.kind == Text:
    x.textStr = "The man the professor the student has studies Rome."
```

### Custom mutator

Besides `defaultMutator` there is also `customMutator` which allows more fine-grained
control of the mutation procedure, like uncompressing a `seq[byte]` then calling
`runMutator` on the raw data and compressing the output again.

```nim
func myTarget(x: seq[byte]) =
  var data = uncompress(x)
  ...

proc myMutator(x: var seq[byte]; sizeIncreaseHint: Natural; r: var Rand) =
  var data = uncompress(x)
  runMutator(data, sizeIncreaseHint, r)
  x = compress(data)

customMutator(myTarget, myMutator)
```

### User-defined mutate procs

It's possible to use distinct types to provide a mutate overload for fields that have
interesting values, like file signatures or to limit the search space.

```nim
# Fuzzed library
when defined(runFuzzTests):
  type
    ClientId = distinct int

  proc `==`(a, b: ClientId): bool {.borrow.}
else:
  type
    ClientId = int

# In a test file
import drchaos/mutator

const
  idA = 0.ClientId
  idB = 2.ClientId
  idC = 4.ClientId

proc mutate(value: var ClientId; sizeIncreaseHint: int; enforceChanges: bool; r: var Rand) =
  # use `rand()` to return a new value.
  repeatMutate(r.sample([idA, idB, idC]))
```

For aiding the creation of mutate functions, mutators for every supported type are
exported by `drchaos/mutator`.

### User-defined serializers

User overloads must use the following proc signatures:

```nim
proc fromData(data: openArray[byte]; pos: var int; output: var T)
proc toData(data: var openArray[byte]; pos: var int; input: T)
proc byteSize(x: T): int {.inline.} ## The size that will be consumed by the serialized type in bytes.
```

This is only necessary for objects that contain raw pointers. `mutate`, `default` and `==`
must also be defined. `drchaos/common` exports read/write procs that assist with this task.

### Dos and don'ts

- Don't `echo`  in a fuzz target as it slows down execution speed.
- Prefer `-d:danger` for maximum performance.
- Once you have a crash you can recompile with `-d:debug` and pass the crashing test case as parameter.
- Use `debugEcho(x)` in a target to print the crashing input.
- You could compile without sanitizers, AddressSanitizer slows down programs by ~2x, but it's not recommended.

### What's not supported

- Polymorphic types, missing serialization support.
- References with cycles. A `.noFuzz` custom pragma will be added soon for cursors.
- Object variants work only with the lastest memory management model `--mm:arc/orc`.

## Why choose drchaos

drchaos has several advantages over frameworks derived from
[FuzzDataProvider](https://github.com/google/fuzzing/blob/master/docs/split-inputs.md)
which struggle with dynamic types that in particular are nested. For a better explanation
read an article written by the author of
[Fuzzcheck](https://github.com/loiclec/fuzzcheck-rs/blob/main/articles/why_not_bytes.md).

## Bugs found with the help of drchaos

### Nim reference implementation

* [use-after-free bugs in object variants](https://github.com/nim-lang/Nim/issues/20305)
* [openArray on empty seq triggers UB](https://github.com/nim-lang/Nim/issues/20294)

## License

Licensed and distributed under either of

* MIT license: [LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT

or

* Apache License, Version 2.0, ([LICENSE-APACHEv2](LICENSE-APACHEv2) or http://www.apache.org/licenses/LICENSE-2.0)

at your option. These files may not be copied, modified, or distributed except according to those terms.
