# copico-bonobo-midi

Turning the CoCoPi Bonobo cartridge into a class-compliant USB MIDI interface for the
Tandy Color Computer, with the MIDI stream exposed to the CoCo as hardware registers.

**Status: exploratory.** Nothing here is built or tested. This README records a plan
worked out on 2026-11-13 so it can be picked up later; it is not a commitment.

The Bonobo hardware is Henry Strickland's (github.com/strickyak/copico-bonobo), studied
read-only at `~/github/copico-bonobo` (`ac907dc`). This project would be new firmware for
that board, written from scratch. None of the existing Bonobo software features (the
NekotOS command protocol, DMA transfers, tether, NMI timer) are carried over.

`bonobo-memory-map.html` is an interactive map of where the card lives in the CoCo's
address space and what it can and cannot reach. Open it in any browser.

## What the hardware gives us

The card has no ROM, no RAM and no latches. Four 74HC138s decode four bus accesses into
four strobe wires to the Pico:

| CoCo access | Strobe | Pico pin |
|---|---|---|
| write $FF68 (or $FF69) | control write | GP22 |
| read $FF68 (or $FF69) | status read | GP26 |
| write $FF6A (or $FF6B) | data write | GP27 |
| read $FF6A (or $FF6B) | data read | GP28 |

A0 is not decoded. The Pico never sees an address. Each strobe lasts while E is high
(about 560 ns at 0.89 MHz, about 280 ns at 1.79 MHz), and whatever the Pico does in that
window is the register. Beyond the ports, the Pico can drive /HALT, /SLENB, /CART (FIRQ)
and /NMI (all together, via GP19) and the data bus (via GP16), which is what makes
spoonfeeding possible. It cannot drive /RESET, /CTS or /SCS, so it cannot be a ROM pak.

## Register model

6850-style, so existing 6850 polling code only needs its addresses changed:

| Address | Read | Write |
|---|---|---|
| $FF68 | status: bit 0 = byte waiting from host, bit 1 = ready to send | control: `$03` = empty the inbound buffer, others ignored (until channel select, below) |
| $FF6A | next byte from the host | byte to the host |

`$03` deliberately leaves the outbound queue alone, so bytes a program sent just before
resetting (for example a final all-notes-off) still reach the host.

## Firmware design

- **Pin setup first.** GP19 (SPOON) high so the card releases /HALT, /SLENB, /CART, /NMI;
  GP16 = 1 (data buffer toward the Pico); GP8-15 inputs.
- **Four PIO state machines**, one per strobe:
  - control write and data write: wait for strobe, sample D0-D7, push (as `write.pio` today);
  - status read: a *sticky* register. `pull noblock` then `mov x, osr`, so a read with
    nothing new queued repeats the last status. Today's `status.pio` returns 31 when empty,
    which has bit 0 set and would read as "byte waiting";
  - data read: drive the preloaded byte, then `push` a token so C knows it was taken.
- **Core 1** services the registers in a tight loop: preload the next inbound byte before
  setting bit 0, clear bit 0 when the last one is taken, forward written bytes to core 0.
  The CoCo needs at least about 5 µs between a data read and its next status read; core 1
  answers well inside that. USB stays off this core because `tud_task()` can stall.
- **Core 0** runs TinyUSB as a MIDI device (`CFG_TUD_MIDI`, `TUD_MIDI_DESCRIPTOR`, no
  `pico_stdio_usb`). It calls `tud_midi_stream_read` only while the inbound buffer has
  room, so the USB hardware holds the host back: flow control with no drops and no
  host-side pacing. Placeholder VID/PID until a real one is needed.

Rough size: 250 lines of C, 40 of PIO, 80 of descriptors and config.

## Roadmap

### 1. USB MIDI interface
The register model and firmware above. The CoCo boots its own ROM; the card only
provides the ports.

Test: a Color BASIC echo loop, driven from the Mac.

```basic
10 IF (PEEK(&HFF68) AND 1)=0 THEN 10
20 B=PEEK(&HFF6A):POKE &HFF6A,B:GOTO 10
```

Repeat at double speed (`POKE &HFFD9,0`) to check the 280 ns read window.

### 2. Spoonfed MIDI loader on RESET
On RESET (and only while a USB host has the device configured, so unplugging USB gives
plain BASIC), the Pico halts the CPU and spoonfeeds a few init writes plus a MIDI loader
at $7000, then jumps to it. The loader takes SysEx records (address, length, data as hex
nibbles; length 0 jumps), so any DECB binary can then be sent from the host.

- Base it on jamz's `tools/loader.asm` (89 bytes): change the two port equates to $FF68 and
  $FF6A, and add `LDS #$7000` at the top, since after a reset S is garbage.
- On a CoCo 1/2, set the SAM memory size (`$FFDD` for 64K or `$FFDB` for 16K) before the
  loader, or $7000 is not RAM.
- BASIC never runs, so loaded programs must do their own setup. The host can send machine
  init writes as ordinary loader records ahead of the program.
- When the Pico sees the loader's `$03` control write, it sends a ready SysEx to the host
  (for example `F0 7D 42 01 F7`; `7D` is the non-commercial ID). Host tools wait for it.
- Power-on may need one press of RESET, since the Pico can't assert /RESET and may boot
  after the CoCo's own power-on reset.

### 3. Adapting jamz
jamz (`~/github/jamz`) polls a 6850 at $FF6E/$FF6F with the same status bits. The changes
belong to that project: make the two addresses constants (about a dozen uses in
`lib/midi-pak.fs`, `lib/smf.fs`, `src/player.fs`, `src/clocktest.fs`, `tools/loader.asm`),
add the Bonobo's USB name to `board.py`'s port match, and drop the 3,125 bytes/s pacing in
`send()`. `src/midiplay.fs` is the natural first target, since it needs no disk. The player's
wire time should mostly disappear (inferred: roughly 700 messages/s instead of about 425).

jamz's automated tests depend on the CoCo Zero's serial console and would not carry over.

### 4. Flash storage
The board has no storage of its own; the Pico has 2 MB of flash (4 MB on a Pico 2), and
the firmware needs well under 100 KB. Uses: built-in 6809 programs fed through the loader
without a host, a store of `.mid` files the CoCo fetches through the registers, and a
TinyUSB mass-storage interface so files can be dragged on from the Mac. Code that answers
register reads must run from RAM while flash is being erased or written.

### 5. DriveWire
DLOAD is not reachable: it uses the bit-banger port on the CoCo's serial DIN, not the
cartridge connector. DriveWire is, and gives `DIR`, `LOADM`, `RUN` from BASIC plus NitrOS-9.

- **Channel select:** a control byte picks which stream the data register carries
  (for example `$D0` DriveWire, `$D1` MIDI). Each driver selects before each burst.
- **Server:** first a relay to pyDriveWire or DW4 over an added USB CDC interface; later a
  minimal server on the Pico serving `.dsk` images from flash (about 11 on a 2 MB Pico).
- **CoCo side:** a Bonobo variant of the `dwread`/`dwwrite` routines that HDB-DOS and
  NitrOS-9 are built with (about 30 lines). The Becker port at $FF41/$FF42 is the closest
  existing variant but sits in /SCS space, which the card cannot decode.
- **Order:** NitrOS-9 first (it boots through its own loader, which step 2 can deliver),
  then HDB-DOS.

### 6. HDB-DOS without a ROM pak
HDB-DOS normally lives in ROM at $C000, which the card cannot provide. Options: a ROM pak
in a Multi-Pak; spoonfeed it into RAM at reset and enter BASIC part way through its startup
(the CoCo 3 copies its ROM to RAM on startup, which would overwrite it); inject it after
BASIC has booted (risky, interrupts are live); or a short typed-in BASIC bootstrap that
pulls it over DriveWire. Read the BASIC startup code in a ROM disassembly before choosing.

## Open questions and risks

- **Power-up halt.** RP2040 pins come up pulled low, which probably enables U10 and holds
  the CoCo halted until the firmware drives GP19 high. An unflashed Pico or one in BOOTSEL
  mode would leave the CoCo halted.
- **Power back-feed.** With J4 jumpered and the CoCo off, USB power back-feeds the CoCo's 5 V
  rail. Plug USB in after the CoCo is on, or fit a Schottky diode in place of the jumper.
- **Double speed.** The 280 ns read window at 1.79 MHz is unverified on the Bonobo.
- **Multi-Pak clash.** A Deluxe RS-232 Pak is commonly listed at $FF68-$FF6B (unverified).
  JP1 can move the block to $FF78-$FF7B, where Orchestra-90 is said to sit at $FF7A-$FF7B.
- **Contention.** Driving /CART or /NMI enables all four U10 drivers push-pull; in a
  Multi-Pak that can fight another card. Polling avoids it.
- **USB IDs.** A real VID/PID is needed before anything is shared.

## References

- `~/github/copico-bonobo`: upstream hardware and firmware (read-only).
- `~/github/jamz`: MIDI file player for the CoCo 3, `tools/loader.asm`, `lib/midi-pak.fs`.
- `bonobo-memory-map.html`: the interactive memory map in this directory.
