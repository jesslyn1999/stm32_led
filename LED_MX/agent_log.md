# Agent Log

## WS2812B Bring-up Summary

- Replaced plain GPIO blinking approach with WS2812-compatible strategy: `TIM1 PWM + DMA`.
- CubeMX changes required:
  - Enable `TIM1 CH1` as PWM (data pin on `PA8`).
  - Set prescaler to `0`.
  - Set auto-reload (`ARR`) to `209`.
  - Add DMA request for `TIM1_CH1` in memory-to-peripheral normal mode.
  - Regenerate code so TIM/DMA init functions and HAL TIM driver are included.
- Added WS2812 user-code logic in `Core/Src/main.c`:
  - Frame buffer for encoded duty values (`24 bits per LED + reset slots`).
  - `ws2812_set_color()` encodes GRB bits into compare values.
  - `ws2812_show()` starts `HAL_TIM_PWM_Start_DMA(...)`.
  - PWM pulse-finished callback stops DMA and clears busy flag.
  - Main loop demo animates RGB pattern across LEDs.

## Timing Rationale

- WS2812 bit rate is `800 kHz`, so one bit period is `1.25 us`.
- TIM1 clock is `168 MHz` (APB2 timer clock), so:
  - ticks per bit = `168e6 * 1.25e-6 = 210`.
  - timer period uses `ARR + 1`, therefore `ARR = 210 - 1 = 209`.
- Duty ticks used:
  - logic `0`: about `0.4 us` high -> `~67` ticks.
  - logic `1`: about `0.8 us` high -> `~134` ticks.

## DMA Width Note

- `Word/Word` was selected because the code buffer is `uint32_t[]`, and HAL API uses `uint32_t*`.
- `Half-word` can also work if buffer type and DMA memory/peripheral widths are changed consistently (typically `uint16_t[]` + half-word transfer).
- Critical rule: DMA memory width must match buffer element size; mismatching widths corrupt waveform timing.

## Hardware Notes

- Common ground between STM32 and LED strip is mandatory.
- Typical data line path: MCU pin -> ~330 ohm series resistor -> WS2812 DIN.
- If 3.3 V data is unreliable at 5 V strip supply, use a level shifter (e.g. 74HCT-family) or lower strip supply slightly.
