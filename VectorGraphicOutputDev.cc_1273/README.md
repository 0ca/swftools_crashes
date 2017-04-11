In `VectorGraphicOutputDev.cc:1273`:
```c
  if(colorMap->getNumPixelComps()!=1 || str->getKind()==strDCT) {
      gfxcolor_t*pic=new gfxcolor_t[width*height];   <--- Pic allocated here, width*height integer overflow!
      for (y = 0; y < height; ++y) {
        for (x = 0; x < width; ++x) {
          imgStr->getPixel(pixBuf);
          colorMap->getRGB(pixBuf, &rgb);
          pic[width*y+x].r = (unsigned char)(colToByte(rgb.r));   <---buffer overflow
          pic[width*y+x].g = (unsigned char)(colToByte(rgb.g));
```

There is an integer overflow when the size of the allocation for pic is calculated:
```c
widht * height
```

The user controls `width` and `height`, both of them are signed int, if the value of width * height overflows the max value of an integer then the allocated memory is smaller than expected and the instructions inside the for loops overwrite memory outside the reversed memory.

For example in the PoC the width is 0x100 and the height is 0x1000001, then when the calculation is done:
```c
width * height = 0x100000100 bigger than an INT (4 bytes)
```
Due the overflow the allocation is made for 0x100 bytes.

And then the program crashes inside the for loop with the values:
```c
 width = 0x100 
y = 1  
x = 0 
```
A buffer overflow happens:

```c
width * y+x = 0x100
pic[0x100] <- for an allocation of 0x100 the last value is 0xFF
```

So pic[width * y+x] is writing outside the allocated memory. This is a heap overflow.

During the interger overflow it is possible to control the size if allocated. In the example 0x100 bytes, but the program crashes always inside the loop. There is no way to overflow only X bytes after the buffer and continue the execution. So this bug is **NOT EXPLOITABLE**.
