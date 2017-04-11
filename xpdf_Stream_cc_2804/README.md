In `xpdf/Stream.cc:2804`:

```c
  // convert to 8-bit integers
  for (i = 0; i < 64; ++i) {
    dataOut[i] = dctClip[dctClipOffset + 128 + ((dataIn[i] + 8) >> 4)];
  }
```

The variable `dctClip` is defined at `xpdf/Stream.cc:1854` as:
```c
static Guchar dctClip[768];
```

But parsing the PoC the expression `dctClipOffset + 128 + ((dataIn[i] + 8) >> 4)` takes the value 780, accesing outside the buffer limits.

This is **NOT EXPLOITABLE**.

This bug doens't crash without ASAN. Only crashes when the program is launch without debugger or with ASAN.
