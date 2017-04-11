In `xpdf/JBIG2Stream.cc:1046`:

```c
JBIG2SymbolDict::~JBIG2SymbolDict() {
  Guint i;

  for (i = 0; i < size; ++i) {
    delete bitmaps[i];		<--- bitmaps is unitiallized
  }
  gfree(bitmaps);
```

`bitmaps` is an array of variables `JBIG2Bitmap`. This array is initialized at the constructor:
```c
JBIG2SymbolDict::JBIG2SymbolDict(Guint segNumA, Guint sizeA):
  JBIG2Segment(segNumA)
{
  size = sizeA;
  bitmaps = (JBIG2Bitmap **)gmallocn(size, sizeof(JBIG2Bitmap *));
  genericRegionStats = NULL;
  refinementRegionStats = NULL;
}
```

The variable `size` is controlled by the user. So the allocated size is controlled.

Parsing the PoC, the stream JBIG2Stream contain an invalid lenght, 0xFF, and after show this message:
```
Error (202): 69 extraneous bytes after segment
```

Then, the `JBIG2SymbolDict` destructor is called with the bitmaps array initialized.

The Destructor is executed following the vtable of the `JBIG2Bitmap` object being freed. The process followed is:
```
bitmaps:
->POINTER TO JBIG2Bitmap
POINTER TO JBIG2Bitmap
POINTER TO JBIG2Bitmap
...
...

JBIG2Bitmap:
->vtable JBIG2Bitmap
atributes

vtable JBIG2Bitmap:
Constructor
->Destructor
```

Since bitmaps is unitiallized it could be controlled by the user if he allocates, uses and releases the block that will be use for the bitmaps.

But it should be necesary to control three block and point one to the next one.

In the PoC rip gets corrupted. But it is not controlled because bitmaps contain a link to the next free block.
