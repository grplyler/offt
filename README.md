# FFTO - Fast Fourier Transform for Odin

A high-performance, in-place FFT library written in Odin, optimized for SDR (Software Defined Radio) applications and real-time signal processing.

## Features

- **In-place radix-2 DIT FFT** - Memory efficient Cooley-Tukey decimation-in-time algorithm
- **Optimized performance** - Pre-computed per-stage twiddle factors and unrolled early stages
- **SDR-friendly utilities** - Built-in support for CU8 IQ data conversion and spectrum analysis
- **Windowing support** - Integrated Hann window with efficient application
- **Spectrum analysis tools** - Magnitude, power (dBFS), peak detection, and FFT shift functions
- **Zero dependencies** - Uses only Odin's core math library

## Quick Start

**Add to your project:**
```bash
git submodule add  https://github.com/grpllyer/offt.git vendor/ffto
```

```odin
import fft "vendor/ffto"

// Create an FFT plan for 1024 samples at 2.4 MHz sample rate
plan := fft.fft_make_plan(1024, 2_400_000.0)
defer fft.fft_destroy_plan(&plan)

// Prepare your data (example with SDR IQ bytes)
iq_bytes := []u8{...} // Your CU8 IQ data
samples := fft.cu8_interleaved_to_cf32(iq_bytes)

// Apply windowing to reduce spectral leakage
fft.fft_apply_hann(plan.window, samples)

// Compute FFT in-place
fft.fft_inplace(&plan, samples)

// Analyze the spectrum
power_db := fft.fft_power_dbfs(samples)
peak_bin, peak_freq := fft.fft_peak_bin(power_db, plan.sample_rate)

// Center the spectrum for display (-Fs/2 to +Fs/2)
fft.fft_shift_inplace(samples)
```

## API Reference

### Core FFT Functions

#### `fft_make_plan(n: int, sample_rate: f32 = 0.0) -> FFTPlan`
Creates an FFT plan for size `n` (must be power of 2). Pre-computes twiddle factors and Hann window.

#### `fft_destroy_plan(plan: ^FFTPlan)`
Cleans up allocated memory for the FFT plan.

#### `fft_inplace(plan: ^FFTPlan, buf: []Complex32)`
Performs in-place FFT using optimized radix-2 algorithm with unrolled early stages.

### Windowing

#### `fft_apply_hann(window: []f32, buf: []Complex32)`
Applies Hann window to reduce spectral leakage. Use before FFT computation.

### Spectrum Analysis

#### `fft_magnitude(buf_freq: []Complex32) -> []f32`
Computes magnitude spectrum |X[k]| from frequency domain data.

#### `fft_power_dbfs(buf_freq: []Complex32, full_scale_ref: f32 = 1.0) -> []f32`
Converts to power spectrum in dBFS (decibels relative to full scale).

#### `fft_peak_bin(power_db: []f32, sample_rate: f32) -> (int, f32)`
Finds the bin index and frequency of the peak in the positive frequency range [0..N/2).

#### `fft_shift_inplace(buf_freq: []Complex32)`
Shifts zero-frequency component to center for spectrum display (-Fs/2 to +Fs/2).

### SDR Utilities

#### `cu8_interleaved_to_cf32(iq_bytes: []u8) -> []Complex32`
Converts interleaved CU8 IQ samples (common SDR format) to Complex32 in range [-1,1).

#### `bin_to_hz_centered(k: int, n: int, sample_rate: f32) -> f32`
Maps FFT bin index to frequency in Hz after FFT shift.

## Performance Optimizations

The library includes several optimizations for real-time performance:

1. **Per-stage twiddle pre-computation** - Eliminates trigonometric calculations during FFT
2. **Early stage unrolling** - Stages 0 and 1 are unrolled for better performance  
3. **Inlined complex arithmetic** - Reduces function call overhead in inner loops
4. **Optional bounds checking disable** - Use `#no_bounds_check` for maximum speed

## Algorithm Notes

Current implementation uses iterative in-place radix-2 Cooley-Tukey DIT. The code includes extensive comments about potential future optimizations:

- Split-radix FFT for ~20% fewer operations
- Mixed radix (radix-4/8) for better vectorization
- SIMD optimizations for modern CPUs
- Real-input FFT specialization
- Cache-friendly memory access patterns

## Memory Usage

- **FFT Plan**: O(N) for twiddle factors + O(N) for window = ~2N complex numbers
- **Computation**: In-place, no additional allocation during FFT
- **Utilities**: Output arrays allocated as needed (caller responsibility to free)

## Requirements

- Odin compiler
- Power-of-2 FFT sizes only
- Input data as `[]Complex32` or convertible formats

## Examples

### Basic Signal Analysis
```odin
// Analyze a 1 kHz sine wave
plan := fft.fft_make_plan(512, 48000.0)
defer fft.fft_destroy_plan(&plan)

// Generate test signal (1 kHz sine wave)
samples := make([]fft.Complex32, 512)
for i in 0..<512 {
    t := f32(i) / 48000.0
    samples[i] = fft.Complex32{math.sin(2*math.PI*1000*t), 0}
}

fft.fft_apply_hann(plan.window, samples)
fft.fft_inplace(&plan, samples)
power := fft.fft_power_dbfs(samples)
peak_bin, peak_freq := fft.fft_peak_bin(power, plan.sample_rate)
```

### SDR Spectrum Display
```odin
// Process RTL-SDR data for waterfall display
plan := fft.fft_make_plan(2048, 2_400_000.0)
defer fft.fft_destroy_plan(&plan)

// Convert raw SDR samples
samples := fft.cu8_interleaved_to_cf32(sdr_iq_data)
fft.fft_apply_hann(plan.window, samples)
fft.fft_inplace(&plan, samples)
fft.fft_shift_inplace(samples) // Center for display

// samples now contains centered spectrum ready for waterfall
```

## License

[Specify your license here]

## Contributing

[Add contribution guidelines if desired]