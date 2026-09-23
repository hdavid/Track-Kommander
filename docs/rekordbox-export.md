# Exporting to a USB stick

Track Kommander writes a Pioneer/AlphaTheta device export straight from your
Traktor collection — a stick a CDJ reads on its own, with beat grids, hotcues
and waveforms. No Rekordbox needed to produce it.

The same stick also gets a `rekordbox.xml`, so it can be imported into
Rekordbox if you want to.

> **The player side has not been confirmed on real hardware yet.** Every byte
> matches what Rekordbox writes wherever that could be checked, and Rekordbox
> itself accepts the analysis — necessary, but not the same as a CDJ loading
> it. Use a spare stick until you have tried one.

## Before the first export

Nothing. The database template is generated if there isn't one
(see [The blank `export.pdb`](#the-blank-exportpdb)).

## Exporting

1. **Library → Rekordbox tab.**
2. Tick the playlists to export. Ticking nothing exports the whole
   collection. Smart playlists are evaluated at export time, so they carry
   whatever matches then.
3. Choose the options:
   - **On the key** *(Add by default)* —
     - **Add to what's there** — merge. Tracks and playlists already on the
       stick are kept; a track you export again is refreshed, and a playlist
       of the same name is replaced.
     - **Replace what's there** — the stick ends up holding this selection
       and nothing else. The library is rebuilt from the blank template and
       the audio that is no longer listed is **deleted**. The confirmation
       says how many tracks and how many GB that is before anything is
       written.
   - **Re-analyse the audio** *(off by default)* — decode every track again,
     ignoring anything analysed before. Only needed if a file was replaced
     with different audio of the same size and modification time.
   - **Timing offset** — shifts every cue and beat grid. See
     [MP3 timing](#mp3-timing).
4. **Export to USB…**, pick the drive, confirm.
5. **Stop** ends the run after the track being written. What is already on the
   stick stays there and stays playable; running the export again adds the
   rest. Waveforms computed before you stopped are kept, so nothing is
   decoded twice.

Eject the drive properly before pulling it out — the database is written last
and needs to be flushed.

## What lands on the stick

```
Contents/<artist>/<album>/<audio files>
PIONEER/rekordbox/export.pdb                  the track and playlist database
PIONEER/rekordbox/track-kommander.json        our own record of what was written
PIONEER/USBANLZ/Pxxx/xxxxxxxx/ANLZ0000.DAT    beat grid, cues A–C, waveform
                             /ANLZ0000.EXT    cues D–H, colour waveform
PLAYLISTS/<name>.m3u8                         plain playlists, for everything else
rekordbox.xml                                 for importing into Rekordbox
```

`track-kommander.json` is how a later merge knows what changed: it fingerprints
each track's audio and its metadata separately, so a track whose cues moved is
rewritten without decoding its audio again.

The `.m3u8` files are plain UTF-8 text with paths relative to the playlist, so
the same stick works in a car stereo, a media player or another DJ's software
— none of which can read `export.pdb`. They are a convenience on top of the
real export: if writing them fails, the stick is still a valid device export.

## Importing into Rekordbox (optional)

The stick is already playable without this. To get the tracks into a Rekordbox
library instead:

1. In Rekordbox's preferences, find the **rekordbox.xml / Imported Library**
   setting (under *Advanced → Database* in recent versions) and point it at
   the `rekordbox.xml` at the root of the stick.
2. The imported library then appears in the tree as its own source; drag the
   playlists from it into your collection.

The XML points at the copies on the stick, not at your originals, so import it
with the stick plugged in.

## Caveats

### The blank `export.pdb`

The database is built by extending a blank one, because the writer library can
add to a file but not create it. Track Kommander generates that blank itself
(`core/pdb_seed.py`) and caches it, so no
Rekordbox-prepared stick is needed. If one happens to be plugged in, its blank
is preferred — a template Rekordbox wrote is by definition the shape Rekordbox
expects.

A generated blank leaves out the `COLORS` and `COLUMNS` rows a Rekordbox one
has (its eight colour names and the browser's column labels). Nothing in the
export reads them, but no player has confirmed it does not want them.

### Hotcues

Traktor stores its beat-grid anchor twice — once as the grid, once as a cue on
the same millisecond sitting in hotcue slot 1. That second one is not a cue you
placed, so it is dropped rather than exported; otherwise every track would
arrive with hotcue A pinned to the first beat. A cue you put anywhere else,
including slot 1, is exported normally.

Players read eight hotcues. Cues beyond that, and cues with no hotcue slot,
become memory cues in the XML and are not written to the device database.

### MP3 timing

MP3 encoder delay can put Traktor and Rekordbox a few milliseconds apart, which
shows up as cues landing slightly off the beat. Export one track, check it, and
set **Timing offset** if you need to. It shifts every cue and grid by the same
amount.

### Not written

- **`exportLibrary.db`** (Device Library Plus / OneLibrary) — encrypted, so it
  cannot be produced. Newer players prefer it but still read `export.pdb`;
  older players only read `export.pdb`.
- **`.2EX` / `.3EX`** — newer analysis variants players do not require.
- **`PSSI` phrase data** — the CDJ-3000 phrase view will be empty.

### Rebuilding leaves orphans

Rebuilding the database from the blank template leaves the audio and analysis
of earlier exports on the drive with nothing pointing at them — invisible to a
player, and still filling the stick.

**Replace** sweeps them, which is what makes it a replace rather than a way to
hide tracks: leaving the files behind is a state nobody wants. The sweep only
ever touches files inside `Contents/` and `PIONEER/` that the database *on that
key* does not reference, and it does nothing at all if that database cannot be
read — an unreadable `export.pdb` would otherwise make every file on the stick
look unreferenced.

Filenames are compared in one Unicode normal form. macOS writes decomposed
names to exFAT while the database rows hold the composed form, so `Maōh` is one
code point in the row and two on the drive; comparing them as written would
make a track the key still lists look unreferenced.

**Add** never sweeps: everything the key already had is still listed, so
nothing is an orphan.

### Damaged files

A file with corrupt MPEG frames still exports — a waveform is produced from
whatever decodes — but the damage is still in the audio a player has to get
through. If a track misbehaves on the CDJ, check whether it decodes cleanly
before suspecting the export.

## Where things live

| What | Where |
|------|-------|
| Export logic | `core/device_export.py` |
| Analysis file format | `core/anlz.py` |
| Copying and free space | `core/usb_export.py` |
| Rekordbox XML | `core/rekordbox.py` |
| Waveform cache | `core/waveform_cache.py` |
| The worker thread | `workers/device_export.py` |
| Cached blank template | `<app data>/export-seed.pdb` |
