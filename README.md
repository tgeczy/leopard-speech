# Leopard-speech

**It does not speak yet — but the engine loads, opens a speech channel,
accepts a voice and runs its own worker threads.** This is where Mac OS X 10.5
Leopard's speech engine will live once it makes a sound, and with it **Alex**,
which is the voice people actually want.

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
- **All five initializers run.**
- **`SEOpenSpeechChannel` returns `noErr`.** All four dictionaries the channel
  manager wants -- `PrefixDictionary`, `CartNames`, `CartLite` and
  `SymbolDictionary` -- map to their own files. See below for what stood in the
  way of that for a day.
- **`SEUseVoice` and `SESpeakBuffer` return `noErr`**, and the engine reads the
  voice's `VoiceDescription`.
- **The engine runs.** It builds an `AUGraph`, negotiates a stream format of
  22050 Hz mono float, starts it, and spins up its own worker threads, which
  tick through `Parse`, `Audio?`, `Samples` and `Ping`.
- **AudioConverter**, which is how Alex decodes where Vicki uses the Sound
  Manager. Implemented and flushed for the Windows 7 decoder quirk.

## The bug that looked like Apple's and was ours

Worth writing down, because it wasted a day and every hypothesis it generated
was wrong in the same way.

`SEOpenSpeechChannel` crashed inside `SLCartDict::SLCartDict`, reading
15,336,982 bytes past a 1,638,242-byte mapping of `PrefixDictionary`. The
arithmetic matched the faulting address to the byte, which made the conclusion
irresistible: the engine had built the wrong class over the wrong file, and the
question was why Apple's own code would do that.

It doesn't. Disassembling `SpeechChannelManager::ISpeechChannelManager` shows
six `CFBundleCopyResourceURL` calls, and the two dictionaries it wraps in
`SLCartDict` are `CartNames` and `CartLite` -- never `PrefixDictionary`. Those
two are then merged into an `SLSplitCartDict`. The engine was right all along.

The real cause was in `SLMMapCache::Map(const char *)`, which nothing had
looked at because it appeared to be working. It stats the path and then walks
its cache list comparing exactly the first eight bytes of the stat buffer --
`st_dev` and `st_ino` -- and nothing else. The loader's `stat` shim zeroed the
whole buffer and filled in only `st_size`, so **every file on disk answered to
the same key**. `PrefixDictionary` mapped correctly; the six after it were
served its bytes straight from the cache, without ever reaching `open()`.

The clue was there the whole time and read backwards: one `open()` for seven
resources looks like six lookups failing, when it was six cache hits
succeeding. Every one of the four hypotheses ruled out below was ruled out
correctly. The cause was somewhere nobody had thought to suspect, because it
was the part that was *not* failing.

Two smaller traps came with it:

- **An anonymous local function inherits the previous global symbol's name.**
  The crash frame read `SpeechChannelManager::UseVoice + 0x6fe`, and `UseVoice`
  really is the nearest preceding symbol -- but the function at that address
  starts at `+0x6c8`, after a complete epilogue, and is a static the linker
  never named. A day of reading the wrong function.
- **A spilled PIC base looks exactly like a return address**, which is already
  written down in the loader's notes and caught nobody by surprise this time.

## Where it is now: Alex speaks

Confirmed by ear on 2026-08-18. He counts to seven, and he is unmistakably
Alex. What remains is a crackle laid over the speech and a doubled initial
consonant here and there — "thtree" for "three".

Getting there needed four things, and three of them were the loader's fault
rather than the engine's.

**Accelerate.** Alex is concatenative, so changing his rate means time-scaling
recorded speech rather than re-running a model. `MTMBModRateWsola` does it with
WSOLA — waveform similarity overlap-add — and WSOLA is a search. Apple sent the
search to vDSP, and with those stubbed Alex ran to completion and produced one
frame of nothing. Nothing failed; the only evidence was a counter reading
58,186,903 calls into an empty function. The signatures were counted off the
call sites rather than remembered, which mattered: `vDSP_vmsb` takes nine
arguments and computes `A*B − C`, `vDSP_vmma` takes eleven and computes
`A*B + C*D`, and `vmul` is vecLib's older seven-argument spelling.

**Alex is AAC**, like Vicki — but fetched completely differently. The engine
maps only the first 77,114,248 bytes of his 701 MB bank, which is the index and
is exactly the value at `+0x28` of the `meow` header, then `pread`s each
waveform grain out of the remaining 624 MB. That is Apple loading it
chunk-by-chunk, visible in the log.

**So two things written for Vicki are wrong for him.** `aac_flush_delay`
re-feeds the last packet to shake loose the frame Windows 7's decoder holds
back; on one long stream the duplicates land past the end, but on Alex the
packet *is* the payload and it arrived three times over. And the priming must
not be trimmed — Apple sets `kAudioConverterPrimeMethod` to None, so taking
Vicki's 2112 samples off a 1024-sample refill deletes it outright.

**And the decoder has to stay open.** `aac_begin()` sends `COMMAND_FLUSH`, and
AAC frames overlap: a frame is not finished until the next one's window is
added to it. Flushing between packets left a seam at every 1024-sample
boundary, which is audibly a stutter — one gap per frame. One packet in also
yields nothing out, so a refill has to keep pulling until the decoder gives
something back.

### What the remaining crackle is not

Measured, so that the next attempt does not start from the same four guesses:

- **Not a framing seam.** The largest sample-to-sample jumps are not periodic
  at 229 (our slice size), 256, 512, 1024 (an AAC frame) or 2048 — no offset
  holds more than 6% of them.
- **Not clipping.** Peak 13638 of 32767.
- **Not the output format.** The engine asks for `'lpcm'` flags 0x0c — signed,
  packed, little-endian, 16-bit — which is exactly what the decoder produces.
- **Not a `pread` race.** Made atomic anyway; the render is byte-identical.
- **Not nondeterminism.** Two runs byte-identical.

It looks like broadband noise laid over the speech: around each large jump the
samples alternate sign at high amplitude, and Alex measures three times rougher
than Fred through the same output path. The next two suspects are the WSOLA
semantics — the argument *shapes* are confirmed from the binary, the *meanings*
only inferred — and the engine's own scheduling clock, since
`MTBEAudioUnitSoundOutput::QueueSamples` turns timestamps into sample positions
and the host completes slices faster than real time.

## The stack alignment nobody can skip

Worth its own heading, because it is invisible until it is not, and it applies
to any Darwin i386 code run on Windows.

**Darwin's i386 ABI requires ESP to be 16-byte aligned at every call
instruction. Windows requires four.** Apple's compiler used the guarantee:
`MTBEWorker::Timestamp` stores a pair of doubles into its own frame with
`movapd`, which faults outright when the address is not 16-byte aligned.

Inside the engine it can never break, because every frame preserves the
alignment it was given. It breaks wherever the host hands control over, and a
thread entry point is the worst case -- the alignment a Windows thread starts
with is neither ours to choose nor the same on every run. Leopard's engine
starts two MP tasks and died at the first `movapd` one of them reached, with
`ebp-0x18` sitting at 8 mod 16.

Tiger never showed this. That is not evidence it was safe; it is one compiler
declining to vectorise one function.

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
