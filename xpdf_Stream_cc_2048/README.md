In `xpdf/Stream.cc:2048`:
```c
      x = 0;
      dy = 0;
    }
    c = rowBuf[comp][dy][x];    <--- NULL dereference
    if (++comp == numComps) {
```

The problem happens because the variable `rowBuf` is initiallized with NULL values.

`rowBuf` gets initiallized at `DCTStream::reset()`, `xpdf/Stream.cc:1969`:
```c
    // allocate a buffer for one row of MCUs
    bufWidth = ((width + mcuWidth - 1) / mcuWidth) * mcuWidth;
    for (i = 0; i < numComps; ++i) {
      for (j = 0; j < mcuHeight; ++j) {
        rowBuf[i][j] = (Guchar *)gmallocn(bufWidth, sizeof(Guchar));
      }
    }
```

Parsing the PoC pdf, `width` has the value `0`, the align variable `bufWidth` gets also the value `0`. And then the allocation is done for 0 elements, returning a NULL pointer. So the whole `rowBuf` is initiallized with NULL pointers and when the code try to dereference this values the program crashes.

This is **NOT EXPLOITABLE**.
