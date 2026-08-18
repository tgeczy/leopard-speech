# Leopard-speech

**Not working yet.** This is where Mac OS X 10.5 Leopard's speech engine will
live if it can be made to run — and, with it, **Alex**, which is the voice
people actually want.

Its sibling [tiger-speech](https://github.com/tgeczy/tiger-speech) does the
same job for 10.4 and works: all twenty-three Tiger voices, natively, about
twelve milliseconds an utterance. The loader there is the starting point here,
and most of it transfers unchanged.

## Why Leopard at all, when Tiger works

Because Alex only exists on Leopard, and **Tiger's engine cannot read him.**

That was measured, not assumed. Alex's sample bank is a `meow` container like
Vicki's, but a later version of it:

| voice | magic | version | header |
|---|---|---|---|
| Vicki, Tiger | `meow` | 1.0.4 | 0x28 |
| Vicki, Leopard | `meow` | 1.0.4 | 0x28 — byte-identical to Tiger's |
| **Alex** | `meow` | **1.0.6** | **0x30** |

Eight more bytes of header. Tiger's engine reads the 1.0.4 layout, so its
cursor into Alex's demi table is eight bytes out and it walks into nothing.
`SEUseVoice` and `SESpeakBuffer` both return `noErr` first, which is the engine
accepting the voice and then misreading it — an encouraging-looking dead end.

## What already works

Carried over from tiger-speech, all of it measured against Leopard's own
binaries:

- **The images load.** Leopard's `MacinTalk` and `SpeechDictionary` map,
  relocate and bind.
- **`__IMPORT,__jump_table` binding.** Leopard's engine is not PIC: it has
  five-byte stubs that arrive as `0xf4` padding for dyld to overwrite with
  `jmp rel32`. Binding only the pointer sections leaves them, and the first
  call executes a privileged instruction. 284 stubs now bind.
- **`$UNIX2003` symbols.** Leopard's libSystem publishes conformance variants
  (`_open$UNIX2003` and twelve more); the shim lookup falls back to the name
  before the `$`.
- **`operator new`.** A thunk returns 0, the engine writes through it, and it
  dies in `SEOpenSpeechChannel`. Written out along with `_List_node_base`.
- **All five initializers run**, and `SEOpenSpeechChannel` is reached.
- **The dictionary opens**, resolving all seven of its resources — Leopard has
  seven where Tiger had four, and no `StdDictionary` at all.
- **AudioConverter**, which is how Alex decodes where Vicki uses the Sound
  Manager. Implemented and flushed for the Windows 7 decoder quirk.

## Where it stops

One bug, and it is **not** about Alex — it happens with Fred just the same,
inside `SEOpenSpeechChannel`, before any voice is chosen:

```
_SEOpenSpeechChannel + 0xe
  SpeechChannelManager::ISpeechChannelManager + 0x2ba
    SpeechChannelManager::UseVoice + 0x6fe
      SLCartDict::SLCartDict + 0x7a
        SLCartDict::SymtabRead        <- access violation
```

The manager's constructor picks a default voice and builds its dictionaries.
It maps `PrefixDictionary` — the first of the seven, and **the only one ever
opened** — and constructs an `SLCartDict` over it.

That is the wrong class for that file. `SLCartDict`'s constructor reads a
header word and sets its cursor to `data + 20 + SmartSwapInt32Value(data[0x10])
+ 1`. `SLDictionary`'s constructor leaves the swap fields at 2 and 1, so the
swap always happens; `PrefixDictionary` is little-endian, so the value comes
back as `0x00EA0601` = 15,336,961 and the cursor lands 15,336,982 bytes past a
1,638,242-byte file. The faulting address matches that arithmetic **to the
byte**.

`SLPrefixDict` exists in the same binary and is presumably what should have
been constructed. So the open question is:

> Why does `SpeechChannelManager::UseVoice` build an `SLCartDict` over
> `PrefixDictionary`, rather than over `CartLite`?

Ruled out already, each by experiment rather than by reasoning:

- **Not the CoreFoundation string lifetime.** Making the object graveyard never
  free changes nothing.
- **Not a wrong resource path.** All six URLs resolve to their own correct
  files.
- **Not CoreFoundation collections.** No stubbed symbol is reached at all
  before the crash, so no plist or dictionary parsing is involved.
- **Not the external relocations.** All 49 unresolved ones are the three C++
  ABI RTTI vtables, which only `dynamic_cast` and exception matching ever
  dereference.

The next step is to read `UseVoice` around `MacinTalk + 0xac7a` and see which
URL it hands to `SLMMapCache::Map`.

## What is still unshimmed

Leopard's binaries import a great deal more than Tiger's, but the *linked*
surface badly overstates the *executed* one — Tiger's engine linked 44
undefined symbols and called six. Still outstanding if the engine reaches them:
a CoreFoundation collection subset, **sqlite3** (eight calls, and SQLite is
public domain so compiling it in is easy), vDSP and `sgesvd_` from Accelerate,
AudioFile, Mach messaging, and thirteen **libstdc++** symbols. Those last must
come from Leopard's own `/usr/lib/libstdc++.6.dylib` rather than be
reimplemented — GCC 4.0.1's copy-on-write `basic_string` layout has to match
exactly, and the engine inlines code that touches it.

## Getting the engine

The same rule as tiger-speech, and it is not a formality: **no part of Apple's
software will ever be distributed here.** You supply your own Leopard install
and a tool extracts the engine from it.

The install DVD hides its filesystem behind a small ISO9660 boot partition, so
a plain listing shows only the Boot Camp documentation. The real one is an APM
partition map:

```
7z l -tapm "Mac OS X Leopard Install DVD.iso"
  Apple.Apple_partition_map            30,720
  Macintosh.Apple_Driver_ATAPI    421,261,312
  Mac_OS_X.hfs                  7,634,907,136   <- everything is in here
```

## Licence

MIT, once there is anything here to license — as with tiger-speech and
outspoken-nvda. It covers the loader and the shims: the work of making Apple's
engine run somewhere it was never built to run, not the engine itself.
