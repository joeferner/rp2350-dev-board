# TODO

Open work on this board. Roughly in the order it wants doing — the pin
layout gates the PCB, and the PCB gates everything else.

## Pin layout for J2 and J3

The current assignment exists only to satisfy ERC. Redoing it is the
largest remaining decision, because it determines how hard the fixture
HAT in `rpi-hal`'s hil-test bench is to route.

What is fixed by silicon:

- **ADC is GPIO40–47 only.** Anything analog has to land there.
- **`QMI CS1n` (the second QSPI chip select) is on GPIO0, 8, 19 and 47.**
  No other pin can do it.
- The console needs a **hardware UART pair** — 1.5 Mbaud with DMA is not a
  PIO job. UART0 TX is on GPIO 0, 2, 12, 14, 16, 18, 28, 32, 34, 44, 46;
  UART0 RX on 1, 3, 13, 15, 17, 19, 29, 33, 35, 45, 47.
- Everything else the fixture does — SPI slave, I2C slave, I2S receive, IR
  decode, logic capture, edge timestamping — is PIO, so those pins are
  electrically free. The one real constraint: a state machine addresses
  pins as a base plus a count, so **signals handled by one SM should be
  contiguous**.

### Structure: chip-order fanout

The headers follow the QFN's pin order — J2 takes the GPIO on the package's
first side, J3 the rest — so the fanout leaves the chip without crossings.
That is the right trade for the densest board in the stack, but it moves a
decision rather than removing one: the reordering to reach the Pi's header
order now happens on the HAT, and the mapping that determines it is no
longer implied by the pinout. See "Record the fixture ↔ Pi GPIO mapping"
below.

The stack is Pi on the bottom, fixture HAT above it, this board on top, so
**J2 goes on the edge sitting over the HAT's Pi-header socket** and J3 on
the inboard edge where the HAT's own devices are. **Pin 1 of J2 must sit
over the same corner as pin 1 of the Pi's own header** — both boards face
up so nothing is mirrored, but the wrong corner forces a whole-header
crossover in the HAT.

### Finish the wiring

ERC is at 39 errors, all of them this rework in progress:

- 33 unconnected header pins — J2 1–14, 16, 21–24, 37–40, and J3 1–5, 28,
  37–40.
- Five dangling labels: `BOOTSEL#`, `GPIO0`, `GPIO1`, `GPIO2`, `GPIO3`.
- One unconnected wire endpoint at (237.49, 148.59).

Still to place: GPIO0–3, `BOOTSEL#`, and every power and ground pin — the
headers currently carry no VSYS, 3V3 or GND at all. That leaves roughly 27
positions of power and ground across 32 free ones, which is ample;
interleave the grounds rather than banking them at one end.

### Decide the 28/20 split

The Pi exposes 28 GPIO and this board has 48, so 20 are housekeeping. The
clean arrangement puts all 28 shadow lines on one header, so the HAT reaches
the Pi's 40-pin header from a single socket.

Chip order does not give that for free. J2 holds GPIO4–20 today, reaching 21
once GPIO0–3 land there; J3 holds GPIO21–47. So as drawn the shadow set has
to straddle both headers and the HAT routes to two sockets to serve one Pi
header.

Moving **GPIO21–27 to J2** restores it — J2 becomes GPIO0–27, exactly the
shadow set, and J3 becomes GPIO28–47, exactly the housekeeping set with the
ADC block included. It costs little, because GPIO21–27 are package pins
21–28, sitting at the corner adjacent to the side J2 already serves. Worth
doing before layout, or worth deciding explicitly against.

### Record the fixture ↔ Pi GPIO mapping

This is the decision chip order defers, and it needs writing down rather
than emerging from layout. The mapping does **not** have to be
`GPn ↔ Pi GPIOn` — the HAT can wire any pin here to any Pi header pin — but
it does have to be deliberate, because two constraints must survive it:

- **Contiguity.** PIO addresses pins as a base plus a count, so each Pi bus
  has to land on a contiguous run on this board: SPI0 (Pi GPIO7–11) is five
  pins, PCM/I2S (Pi 18–21) four, aux SPI1 (Pi 16–21) six, I2C1 (Pi 2–3)
  two. A mapping that scatters any of those makes the corresponding slave
  role unimplementable without rewiring.
- **The console pair, crossed.** Whichever pins face Pi GPIO14/15 have to
  be a hardware UART pair — 1.5 Mbaud with DMA is not a PIO job — and
  crossed, this board's TX to the Pi's RXD0. Straight-through is silent on
  both sides and is the most expensive wiring mistake available on this
  bench.

The resulting table becomes the firmware pin map; there is nothing else to
derive it from once the pinout stops being self-describing.

**Never take VSYS from the Pi's 5V header pins.** They are the switched
rail, and this board has to stay alive while the Pi is dead.

### Housekeeping pins (provisional)

GPIO28–47, with the ADC group at one end. Assignment is a starting point,
not settled — the ADC row is forced by silicon, and GP28 and GP47 are
already committed to the status LED and the PSRAM chip select, so the HAT
must drive neither:

| Fixture | Use |
| --- | --- |
| GP28 | status LED — fitted, reserved, HAT must not drive it |
| GP29 | Pi load-switch EN |
| GP30 | Pi load-switch FAULT |
| GP31 | Pi `RUN` open-drain |
| GP32 / GP33 | INA226 I2C (I2C0 SDA/SCL) |
| GP34 | rail bleeder FET enable |
| GP35, GP39 | spare |
| GP36–GP38 | USB VBUS switch enables |
| GP40 (ADC0) | analog audio L |
| GP41 (ADC1) | analog audio R |
| GP42 (ADC2) | Pi 3V3 rail sense |
| GP43 (ADC3) | current shunt, if not using the INA226 |
| GP44–GP46 | USB VBUS switch faults |
| GP47 (ADC7) | **PSRAM chip select** — fitted; the only `QMI CS1n` pin left once GP0/8/19 are shadow lines |

The remaining header positions: VSYS ×4, 3V3 ×4, GND, VBUS ×2.

Silkscreen matters more than usual on a bench tool that gets probed —
label every position, and make VSYS and VBUS visually distinct so they are
never jumpered together.

## QSPI PSRAM

U5 is fitted and wired. Two things left:

- **Confirm the QMI supports PSRAM on `CS1n`** specifically, and check the
  RP2350 errata for anything touching it, before the footprint is committed
  to copper.
- **R11 (the CS pull-up) is 0805.** It sits on a QSPI net in the tightest
  part of the board; make it 0402 like the other signal-path passives.
- **Consider dropping GPIO47 from J3.** It is committed to the PSRAM chip
  select now, so exposing it on the header buys nothing and costs a
  header-length stub on a chip select, plus the chance of the HAT driving
  it. Leaving J3.6 unconnected is the safer default.

## Power path

- **Add ~1 µF at U4's VIN.** That node is the USB-C VBUS net and currently
  has no capacitance at all, since C3 and C4 both sit on the VOUT side. The
  part's absolute maximum on VIN is ±6 V, so an undecoupled VBUS trace
  ringing on hot-plug is a real threat and not just a datasheet
  formality.
- **Relabel the four +5V header pins as VSYS.** They are an input — the
  HAT's always-on rail feeding this board through the ideal diode — not an
  output. Optionally expose one raw VBUS pin, pre-diode, clearly named.
- **Route `ST` to a GPIO with a pull-up** instead of to GND. With `CE` on
  VOUT it becomes a real signal: Hi-Z while drawing from USB, low while
  running off the HAT rail, so the firmware can report its own power source.
- **Add an ERC exclusion for the `ST`-to-GND `pin_to_pin` error** if `ST`
  stays grounded. Grounding an unused `ST` is what the datasheet
  prescribes; the error is an artifact of the GND net carrying a
  `PWR_FLAG`.

Nothing else is needed here. Board power for the Pi under test belongs to
the HAT's load switch, which has soft-start, a programmable current limit
and a fault output — so no current limit, soft-start, fuse, bulk
capacitance or heavy copper is required on this board's 5V path. The 1.5 A
LM66100 is a wide margin for a fixture that draws under 100 mA.

## PCB

Nothing is routed yet: 48 footprints placed, no tracks, no zones, no board
outline.

- **Go 4 layers** (F.Cu / GND / PWR / B.Cu). Currently 2 layers at 1.6 mm.
  A close ground plane is what makes 90 Ω achievable on the USB pair,
  which 1.6 mm of FR4 to the bottom layer cannot do; it also frees the
  decoupling placement around an 0.4 mm-pitch QFN-80 and makes the
  RP2350's regulator layout reproducible. This is the one board in the rig
  that has to be more reliable than the things it measures.
- **Draw the board outline early — it gates the HAT's layout.** The HAT is
  65 × 56.5 mm and also has to carry 28 series resistors, a load switch,
  an INA226, an ID EEPROM, an I2C EEPROM, a temperature sensor, an SH1106
  footprint, an SPI flash or MCP3008, IR receive and transmit, a PCM5102,
  the analog audio path and four headers. This board sits on top and
  shadows part of it, and several of those parts cannot live underneath:
  the IR receiver and IR LED need line of sight, the SH1106 needs to be
  visible, the TRRS jack needs edge access, and test points need probe
  access. So the outline has to be settled before the HAT is placed, not
  discovered afterwards.
- **Being the top board makes the ergonomics easy — take the win.** Put
  BOOTSEL, RST, both LEDs, the USB-C and any SWD connector on the top side
  near an edge, and they are all reachable with the full stack assembled.
  The horizontal top-mount USB-C is the right choice for this position;
  check the cable boot clears the HAT and the Pi's own connectors.
- **Decide which board carries the pins and which the sockets.** Male
  headers pointing down on this board is the conventional and sturdier
  arrangement, and it is what makes standalone breadboard use possible.
- **Set the header spacing to a whole number of 0.1" increments** if
  breadboard use matters — the bench's smoke tier and early bring-up assume
  exactly that. A Pico's 0.7" straddles a breadboard channel but is too
  narrow to escape a 10 × 10 mm 0.4 mm-pitch QFN-80; 1.1" (27.94 mm) is the
  nearest spacing that plausibly clears the fanout.
- **Add mounting holes, with standoff positions coordinated with the HAT.**
  Two 2×20 headers carry the board mechanically, but they should not also
  absorb the force of somebody pressing BOOTSEL.
- Check the total stack height: Pi header, HAT, this board's headers, and
  this board's tallest parts — the 6 mm through-hole buttons.
- **Layout constraints carried from the RP2350 hardware design guidance:**
  L1's orientation and polarity marking (coil direction measurably affects
  the on-chip regulator); keep L2 away from L1 and from the RP2350's
  regulator output caps; `VREG_PGND` returning switching current directly
  to the pin without disturbing the rest of GND; QSPI short and direct;
  crystal traces short with minimal parasitic capacitance; the 27 Ω USB
  resistors close to the chip with the pair over solid ground.
- **Add test points** on 3V3, 1V1, VSYS and the `VREG_LX` node. Bring-up on
  a board with no debug connector is otherwise guesswork.

## Library and BOM hygiene

- **`fp-lib-table` still points at `${KIPRJMOD}/rp2350-dev-board.pretty`,
  which no longer exists.** KiCad errors on project open. Delete the row,
  or the file — it has no other entries.
- **No voltage rating or dielectric on any capacitor.** Specify X5R/X7R and
  a rating; the 10 µF on the 5 V rail wants ≥16 V, and the 0402 4.7 µF
  parts derate hard enough that the effective value is worth knowing.
- **L1 has no MPN**, and both inductors currently share the
  `L_Changjiang_FTC201612S` land. Verify each chosen part's recommended
  land against it — for the WIP201612S that is A = 1.6, B = 0.9, C = 2.0 —
  and draw an exact footprint if either disagrees.

## Additions worth considering

- **An SWD connector.** SWCLK/SWDIO reach J3 only, and once this board is
  socketed under a HAT under a Pi they are unreachable. A 3-pin JST
  SM03B-SRSS-TB takes a Raspberry Pi Debug Probe directly. For a board
  whose firmware is under active development, this is the cheapest
  debugging improvement available.
- **A DNF 10 kΩ pull-up on the primary flash's `QSPI_SS`.** Unnecessary for
  this Winbond part, useful insurance if the flash is ever substituted, and
  the question becomes live once a second QSPI device shares the bus.
- **Decide where the RP2350-E9 pull-downs live.** The erratum — GPIO inputs
  latching part-way high when relying on the internal pull-down — is close
  to worst case for a fixture whose job is high-impedance observation of
  lines that go floating. Per-line external pull-downs on the HAT are
  probably right, since the HAT knows what each line does, but the decision
  should be recorded rather than left to whoever lays out the HAT.
- **Expand `README.md`.** Three lines today. It should say what the board is
  for, why RP2350B rather than a Pico-class part, and what the two headers
  are, so the pinout above is discoverable without reading the schematic.

## Downstream

Consumers that need updating once the board exists. Listed so they are not
forgotten, not as work tracked here: `rpi-hal`'s hil-test fixture firmware
needs a board variant and identity, a pin map matching the layout above,
and its status-LED pin pointed at GP28.
