# Performance

## Intro

Find here some information about the performace of the library such us CPU
footprint, GPU usage (future) and maybe memory consuption.

## CPU footprint

We built different video transformers where selfie segmentation was
performed through different third party libraries. The goal was to compare the
CPU footprint in a sample app (the same) running on a given platform (macOS).

We measured TFLite library (third party library) provides the lowest
footprint so as of today it is the library being used by the ML transformer
library for all supported platforms.

## CPU footprint data(*) (no GPU being used)

| Transformer | Third party library | % CPU Footprint (avg) |
|-------------|---------------------|-----------------------|
| Noop        | None                | 11                    |
| TFLite      | TFLite              | 22                    |
| MediaPipe   | MediaPipe           | 100                   |

Notes: *  TFLite and MediaPipe transformers just obtaing the selfie segmentation
          mask from each video frame.
