<img title="" src="img/MSU-1%20Logo.svg" alt="MSU-1 Logo.svg" width="449" data-align="center">

# The Media Streaming Unit (MSU-1)

Note: *The SD2SNES/FXPak Pro will be referred to as 'hardware', unless there is a need to be specific.*

The MSU-1 (Media Streaming Unit, revision 1) is an enhancement chip for the Super Nintendo Entertainment System (SNES), designed by Byuu (AKA Near). 

The MSU-1 is capable of the following:  

- 1 PCM audio playback channel, supports 44100hz 16 bit stereo unsigned PCM audio (some emulators also accept OGG) with a single loop point per track

- Up to 65,536 tracks can be indexed

- 4GB of general-purpose **read-only** data storage via a secondary data file

- Data can be streamed from the data file 1 byte at a time.

# MSU-1 Register Chart

The MSU-1 registers range from ``$2000`` to ``$2007``.  

Their functions change if they are being read from or written to.  

| Address   | Reading                                                        | Writing                                        |
| --------- | -------------------------------------------------------------- | ---------------------------------------------- |
| ``$2000`` | ``MSU_STATUS`` - Chip status port                              | ``MSU_SEEK`` - 32-bit Data seek port (4 bytes) |
| ``$2001`` | ``MSU_READ`` - 8-bit data read port                            | ``MSU_SEEK2``                                  |
| ``$2002`` | ``MSU_ID`` - Chip identification string (6 bytes); returns 'S' | ``MSU_SEEK3``                                  |
| ``$2003`` | ``MSU_ID2`` - returns '-'                                      | ``MSU_SEEK4``                                  |
| ``$2004`` | ``MSU_ID3`` - returns 'M'                                      | ``MSU_TRACK`` - Audio track index (2 bytes)    |
| ``$2005`` | ``MSU_ID4`` - returns 'S'                                      | ``MSU_TRACK2``                                 |
| ``$2006`` | ``MSU_ID5`` - returns 'U'                                      | ``MSU_VOLUME`` - Audio volume                  |
| ``$2007`` | ``MSU_ID6`` - returns '1'                                      | ``MSU_CONTROL`` - Audio state control          |

# Register Explanation

## Reading

## ``MSU_STATUS`` / `$2000` - Chip Status Port

This register is a bitfield indicating the current chip status.

| 7         | 6          | 5            | 4             | 3             | 2        | 1        | 0        |
| --------- | ---------- | ------------ | ------------- | ------------- | -------- | -------- | -------- |
| Data busy | Audio busy | Audio repeat | Audio playing | Track missing | Revision | Revision | Revision |

**Data/Audio busy** - these bits indicate when the MSU-1 is busy reading audio/data.

**Please note** that no emulators are known to correctly emulate this functionality. 

**However**, on hardware, you must wait for the Audio busy bit to clear before changing the current audio track, and for the Data busy bit to clear before reading out data. If you do not implement such checks, the MSU-1 will fail to stream any audio or data. It is strongly recommended you implement such checks regardless of your target platform.

**Track missing** - If the MSU-1 cannot find the track at the set index, this bit will be set. This bit can be used to handle playback errors.

**Revision bits** - Reports the chip's revision. Will likely return 2 if that particular emulator's standard is up to date. The only difference between revision 1 and revision 2 is audio resume support. Some emulators may report a revision of 1 and still support this feature. The emulators BSNES and SNES9x and the SD2SNES/FXPak Pro flashcarts are known to report the correct revision of 2.

## `MSU_READ` / `$2001` - Data Read Port

Performing a read operation on the data read port will return the next byte of data and automatically increment the seek port address by 1. Will do nothing if the data busy bit in the status port is set.

## `MSU_ID` / `$2002` - `$2007` - Chip Identification

Reading ``$2002`` - `$2007` in order will return the string ``S-MSU1`` in ASCII. This can be used to check if the chip is available and functioning for compatibility checks and the like. Realistically, you only need to read ``$2002``.

## Writing

## `MSU_SEEK` / `$2000` - ``$2003`` - Data Seek Port

These four registers combine to one 32-bit address into the data file, with ``$2000`` being the least significant byte.

Writing to ``$2003`` will trigger the seek, setting the Data busy bit until completed.

## ``MSU_TRACK`` / ``$2004`` - ``$2005`` - Audio Track Index

These two registers form one 16-bit index, with ``$2004`` being the least significant byte. Writing to ``$2005`` stops the current track and triggers the track change, setting the Audio busy bit until completed.

Tracks can be indexed from 0 to 65535.

## ``MSU_VOLUME`` / ``$2006`` - Audio Volume Control

Sets the playback volume. ``$00`` is off, ``$7F`` is medium, and ``$FF`` is the maximum.

## `MSU_CONTROL` / `$2007` - Audio State Control

This register is a bitfield that controls the audio playback state.

| 7      | 6      | 5      | 4      | 3      | 2      | 1      | 0    |
| ------ | ------ | ------ | ------ | ------ | ------ | ------ | ---- |
| Unused | Unused | Unused | Unused | Unused | Resume | Repeat | Play |

Writing to this address only has an effect if the Audio busy bit is clear.

If the play bit is not set, the MSU-1 audio is paused.

The repeat bit allows for the currently playing track to loop to the loop point set in its PCM file once the end of the track has been reached.

The resume bit was added in revision 2. To use:

If an audio track is playing and ``MSU_CONTROL`` is written to so that only the repeat bit is set, the current track number and offset into that track is saved.

To resume the saved track, load its track index into ``MSU_TRACK``. The track will be loaded and the MSU-1 will seek to the saved offset.

# Setup

## Audio Track Formats

## MSU-1 .pcm Format

Hardware and all emulators support the ``.pcm`` file format.

[MSUPCM++](https://github.com/qwertymodo/msupcmplusplus) can be used to generate ``.pcm`` files in the correct format.

``.pcm`` files are in 44.1khz, 16-bit stereo, uncompressed, unsigned PCM, little endian, left channel first format.

``.pcm`` files begin with an 8-byte header. The first 4 bytes are ``MSU1`` in ASCII. The next 4 bytes are a 32-bit unsigned integer that define the loop point in samples. Each sample is 4 bytes.

## OGG

Some emulators also support the OGG format for encoding audio tracks as a space saving measure. a ``LOOPSTART`` comment in each track's OGG file is supported that defines the loop point in samples.

## Common Naming Scheme

Hardware and most emulators accept this common file structure:

- ``[NAME].sfc`` for the program ROM,

- ``[NAME].msu`` for the MSU-1 data file, and

- ``[NAME]-#.pcm`` for each audio track 0-65535 (if any). OGG can be used instead if the target emulator supports it.

The MSU-1 data file is required for emulators to recognize that the MSU-1 should be used. If no data is needed, the data file can be left empty or filled with some placeholder data.

## .msu1 Container Format

This is a ZIP-based container format that at least SNES9x is known to support. It allows for a MSU-1 software to be bundled into a single file. If stored as uncompressed, the ``.msu1`` container can be used to create a patch.

Its filesystem is as follows:

- ``program.rom`` - The program ROM

- ``data.msu`` - The MSU-1 data file

- ``track-#.(pcm/ogg)`` - Each audio track 0-65535 (if any). OGG can be used instead of ``.pcm``.

## Miscellaneous File Format Notes

- Ares requires that the MSU-1 data file be named ``msu1.data.rom``. Ares is supposed to be able to detect the data file by the common name, but as of writing this does not appear to work.

# Library

## Register Defines

```asm
; READING
msu_status    equ    $2000 ; Status port
msu_read    equ    $2001 ; Data read port - when read, returns the next data byte, then increments the seek port address by 1
msu_id        equ    $2002 ; Identification bytes
msu_id2     equ $2003 ; contains ASCII string "S-MSU1"
msu_id3     equ $2004
msu_id4     equ $2005
msu_id5     equ $2006
msu_id6     equ $2007

; WRITING
msu_seek    equ    $2000 ; Data seek port bytes, forms a 32 bit address
msu_seek2    equ    $2001
msu_seek3    equ    $2002
msu_seek4    equ    $2003
msu_track    equ    $2004 ; Current audio track
msu_track2    equ    $2005
msu_volume    equ    $2006 ; track volume from 0 - 255 ($FF)
msu_control    equ    $2007 ; stop/play/repeat track

; CONSTANTS

; ## MSU_STATUS register ($2000) ##
status_databusy         equ 10000000b ; 128
status_audiobusy        equ 01000000b ; 64
status_audiorepeat      equ 00100000b ; 32
status_audioplaying     equ 00010000b ; 16
status_trackmissing     equ 00001000b ; 8
; Chip revision, will either report 1 or 2 depending on the emulator/hardware
status_revision1        equ 00000100b ; 4
status_revision2        equ 00000010b ; 2
status_revision3        equ 00000001b ; 1

; ## MSU_CONTROL register ($2007) ##
; Bit 0: Play track flag
; Bit 1: Repeat track flag
; Bit 2: Resume track flag
; Bits 3 - 7: unused
audiostate_none        equ    00000000b ; 0
audiostate_norepeat    equ    00000001b ; 1
audiostate_repeat    equ    00000011b ; 3
audiostate_resume    equ    00000100b ; 4
```

## Example Code

Below are some basic assembly routines dealing with the MSU-1:

## Checking for chip presence

```asm
CheckForMSU
    a8    ; Set A to 8-bit mode if necessary
    lda MSU_ID
    cmp #$53    ; 'S'
    bne NoMSU    ; Stop checking if it's wrong
MSUFound
    ; Do something with this fact here.
    rts
NoMSU
    ; Do something with this fact here.
    rts
```

## Track Playback

```asm
    a8i16    ; A 8-bit, X/Y 16-bit
    stz MSU_VOLUME    ;Stop chip to prevent any buzzing on hardware
    stz MSU_CONTROL
    ldx #$0001    ; Writing a 16-bit value will automatically
    stx MSU_TRACK    ; set $2005 as well, so this is easy.
.audiobusyloop
    bit    msu_status ; Is the audio busy bit set?
    bvs    .audiobusyloop ; Loop until it isn't
    lda #$FF    ; Set volume to max
    sta MSU_VOLUME
    lda #audiostate_norepeat    ; Set audio state to play, no repeat.
    sta MSU_CONTROL
    ; The MSU-1 will now start playing.
    ; Use lda #audiostate_repeat to play a song repeatedly.
```

## Data Seeking

```asm
    ; Seeks to $0000:0410, just as an example.
    i16    ; 16-bit X/Y
    ldx #$0410
    stx MSU_SEEK
    ldx #$0000
    stx MSU_SEEK+2
.databusyloop
    bit MSU_STATUS    ; Wait for the Data Busy bit to clear.
    bmi .databusyloop
```

# See Also

- [Programming the MSU1](https://helmet.kafuka.org/msu1.htm) by Kawa - The page this document was heavily based upon. It's quite outdated (13 years old as of writing) and only covers revision 1 of the chip, but a good amount of the information within is still relevant.

- [SNES-MSU-1-Driver](https://github.com/Sunlitspace542/SNES-MSU-1-Driver/tree/main) by Sunlit - Lots of example code in here.
