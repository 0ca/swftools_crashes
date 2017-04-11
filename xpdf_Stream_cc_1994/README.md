In `xpdf/Stream.cc:1994`:

```c
    // allocate a buffer for one row of MCUs
    bufWidth = ((width + mcuWidth - 1) / mcuWidth) * mcuWidth;
    for (i = 0; i < numComps; ++i) {
      for (j = 0; j < mcuHeight; ++j) {
        rowBuf[i][j] = (Guchar *)gmallocn(bufWidth, sizeof(Guchar));  <--- Writing outside the buffer limits
      }
    }
```

The variable rowBuf is defined at `xpdf/Stream.h:612` as:
```c
  Guchar *rowBuf[4][32];        // buffer for one MCU (non-progressive mode
```

But when the PoC is parsed the mcuHeight takes the value `88`, more than the `32` elements expected and the buffer is overflown writting new allocated pointers outside the memory space of the rowBNuf variable.

This is **NOT EXPLOITABLE**, since it is not possible to control tha values written, pointers to allocated memory.
