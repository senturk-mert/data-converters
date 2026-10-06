<!--
This file is used to generate the project datasheet.
-->

## How it works

TinyQuant puts three data converters on one 1x2 tile, behind a single differential analog
input (ua[0] = VINP, ua[1] = VINN):

* **SAR ADC (10 bit).** Asynchronous successive approximation with Vcm-based (merged
  capacitor) switching. Two MOM capacitor DACs (512 units of 2 fF per side, calibrated by
  extraction), a StrongARM comparator and self-timed bit cycling: the clock only sets the
  sampling instant (falling edge of `clk`), the ten decisions run on internal timing
  (49 ns typical after layout).
* **Noise-shaping SAR (NS_EN = 1).** The same core with a passive first-order noise shaper:
  after the last decision a half-LSB step centres the residue, which is then charge-shared onto
  an integration capacitor (about 4x the DAC capacitance). A second comparator input pair with 4x
  weight adds the integrated residue to every comparison, giving NTF = 1 - 0.8 z^-1 without any
  amplifier. With the same chip you can compare plain and noise-shaped operation.
* **Delta-sigma modulator.** 2nd-order, 1-bit, fully differential switched-capacitor modulator
  (CIFB, coefficients 0.15 / 0.5 / 0.15), two folded-cascode OTAs with switched-capacitor common
  mode feedback, references at the supply rails, clocked at the `clk` rate (phase 1 = clock high).

`ui_in[0]` selects the active converter. The inactive one is held idle (SAR in reset, or the
modulator clock gated and its bias switched off), and the outputs are multiplexed onto the same
pins.

| pin | SAR mode (ui_in[0] = 0) | DSM mode (ui_in[0] = 1) |
|---|---|---|
| uo_out[7:0] | D9..D2 (D9 = MSB) | uo_out[0] = bitstream, uo_out[1] = phase-1 monitor, others 0 |
| uio_out[1:0] | D1..D0 | 0 |
| uio_out[2] | CONV (high while converting) | 0 |
| ui_in[1] | NS_EN (1 = noise shaping on) | not used |
| ui_in[3:2] | not used | bias trim (00 = nominal, 01 = 1.5x, 10 = 0.6x, 11 = 1.1x) |

## How to test

Apply a differential input around a 1.65 V common mode to ua[0]/ua[1]
(SAR full scale about +-3.0 V differential, keep the modulator within +-2.3 V).

**SAR / NS-SAR.** Set `ui_in = 0x00` (or `0x02` for noise shaping), pulse `rst_n`, then clock the
project. Every falling edge of `clk` samples the input and starts a conversion; the result
appears on the outputs when the conversion ends, so it is stable for reading at the next falling
edge. Code = (uo_out << 2) | uio_out[1:0]. The conversion needs up to about 63 ns
in the slow corner, so keep the clock low time longer than that (up to 7 MHz with a
50 % duty cycle). In noise-shaping mode, oversample and low-pass filter (decimate) the codes, and
discard the first 32 codes after reset or after switching NS_EN on (the integration capacitors
charge to their operating point).

**Delta-sigma.** Set `ui_in[0] = 1`, pulse `rst_n` and clock up to 40 MHz. Capture uo_out[0] on
the rising edge of `clk` (it changes on the falling edge), then decimate (for example sinc3 by
64 to 128) or take an FFT of the bitstream.

`test/tinyquant_demo.py` runs both converters from the demo board by single-stepping the clock
(no fast capture needed), and `test/analyze.py` computes SNDR/ENOB from captured data.

## External hardware

A differential source with 1.65 V common mode (two DAC or bench-supply outputs for DC tests, a
signal generator with a single-ended to differential driver for dynamic tests). For full-speed
tests, a logic analyzer or the RP2040/RP2350 PIO capturing the outputs synchronously to `clk`.
