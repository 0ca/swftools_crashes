In `xpdf/JBIG2Stream.cc:768`:

```c
void JBIG2Bitmap::clearToZero() {
  memset(data, 0, h * line); 	<--- data is NULL
}
```

The variable data is not initialized so the program crashes writing to the address 0x0.

This happens when the constructor `JBIG2Bitmap::JBIG2Bitmap` is called with a negative width `wA` (as the PoC does):
```c
JBIG2Bitmap::JBIG2Bitmap(Guint segNumA, int wA, int hA):
  JBIG2Segment(segNumA)
{
  w = wA;
  h = hA;
  line = (wA + 7) >> 3;
  if (w <= 0 || h <= 0 || line <= 0 || h >= (INT_MAX - 1) / line) {
    data = NULL;
    return;
  }
  // need to allocate one extra guard byte for use in combine()
  data = (Guchar *)gmalloc(h * line + 1);
  data[h * line] = 0;
}
```

Then data is set to NULL and it isn't allocated. After that `clearToZero` is called and the bug happens:
```c
  bitmap = new JBIG2Bitmap(0, w, h);
  bitmap->clearToZero();
```

This is **NOT EXPLOITABLE**.
