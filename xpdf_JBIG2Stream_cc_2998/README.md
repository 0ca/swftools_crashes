In `xpdf/JBIG2Stream.cc:2998`:

```c
  if (nRefSegs == 1) {
    seg = findSegment(refSegs[0]);
    if (seg->getType() != jbig2SegBitmap) {   <--- seg is NULL
```

Very similar to `../xpdf_JBIG2Stream_cc_1532/README.md`.

This is **NOT EXPLOITABLE**.
