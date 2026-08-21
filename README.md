# leopard-speech has moved

**This repository is archived. Development and new releases are now in
[panthera-speech](https://github.com/tgeczy/panthera-speech).**

`leopardspeech` itself has not changed — same add-on, same name, same
settings. Only its home has.

## Where to get the add-on

New releases are published in panthera-speech, tagged `leopardspeech/vN.N.N`:

**https://github.com/tgeczy/panthera-speech/releases**

The six releases already published here stay downloadable, so existing links
keep working. They are not updated.

## Why

`leopardspeech` and `tigerspeech` were always one program wearing two hats:
**the same Mach-O loader**, pointed at a 10.5 tree or a 10.4 one. Across two
repositories that meant `build.sh` here reached into a sibling checkout to
find it, and the guard that refuses to ship a developer's paths only ever ran
over Tiger's half.

panthera-speech is the two of them under one loader, with Snow Leopard and
Lion to follow. It is named for the genus — tiger, leopard, snow leopard and
lion are all *Panthera* — and the cat that is not in it, the mountain lion, is
exactly where the work stops: 10.8 is the first release with no 32-bit slice.

## History

Every commit here was carried over with its authorship intact, including
[@serrebidev](https://github.com/serrebidev)'s. Issues moved too.

## Licence

MIT, unchanged. It covers the loader, the driver and the tools — never
Apple's engine, which was not distributed here and is not distributed there.
