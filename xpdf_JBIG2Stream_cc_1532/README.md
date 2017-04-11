In `xpdf/JBIG2Stream.cc:1532`:

```c
  for (i = 0; i < nRefSegs; ++i) {
    seg = findSegment(refSegs[i]);
    if (seg->getType() == jbig2SegSymbolDict) {		<--- seg is NULL
```

The program crashes dereferencing the variable `seg`, because it is value is NULL.

`seg` is assigned in the line before with the function `findSegment`. The function `findSegment` in some scenarios can return a NULL value, and the program is not controlling that:
```c
JBIG2Segment *JBIG2Stream::findSegment(Guint segNum) {
  JBIG2Segment *seg;
  int i;

  for (i = 0; i < globalSegments->getLength(); ++i) {
    seg = (JBIG2Segment *)globalSegments->get(i);
    if (seg->getSegNum() == segNum) {
      return seg;
    }
  }
  for (i = 0; i < segments->getLength(); ++i) {
    seg = (JBIG2Segment *)segments->get(i);
    if (seg->getSegNum() == segNum) {
      return seg;
    }
  }
  return NULL;
}
```

This is not **EXPLOITABLE**.
