In `xpdf/Stream.cc:475`:

```c
  // read the raw line, apply PNG (byte) predictor
  memset(upLeftBuf, 0, pixBytes + 1);    <--- Stack buffer overflow
  for (i = pixBytes; i < rowBytes; ++i) {
    for (j = pixBytes; j > 0; --j) {
      upLeftBuf[j] = upLeftBuf[j-1];   <--- The values are overwritten here too
    }
    upLeftBuf[0] = predLine[i];    <--- The values are overwritten here too 

```

The variable `pixBytes` can takes values higher than the size of `upLeftBuf` causing a buffer overflow overwriting the stack frame and RET address with NULL values.

`upLeftBuf` is a function variable defined as:
```c
GBool StreamPredictor::getNextLine() {
  int curPred;
  Guchar upLeftBuf[gfxColorMaxComps * 2 + 1];
```

`gfxColorMaxComps` is defined as 32. So `upLeftBuf` is an array of 64 `unsigned char`.

The problem comes because the variable `pixBytes` is not contrained and can have any value. This variable is set in:
```c
StreamPredictor::StreamPredictor(Stream *strA, int predictorA,
                                 int widthA, int nCompsA, int nBitsA) {
  str = strA;
  predictor = predictorA;
  width = widthA;
  nComps = nCompsA;
  nBits = nBitsA;
  predLine = NULL;
  ok = gFalse;

  nVals = width * nComps;
  if (width <= 0 || nComps <= 0 || nBits <= 0 ||
      nComps >= INT_MAX / nBits ||
      width >= INT_MAX / nComps / nBits ||
      nVals * nBits + 7 < 0) {
    return;
  }
  pixBytes = (nComps * nBits + 7) >> 3;   

```

`nBits` comes directly from the PDF document it is the bits defined in the decode parameters:
```c
/Filter /FlateDecode
/DecodeParms 
<< 
	/BitsPerComponent 1000    <--- This is nBits
	/Predictor 15 
>>
``` 

The result of the operation shown before `pixBytes = (nComps * nBits + 7) >> 3;` is 125 greater than the size of `upLeftBuf`, 64.

It looks the user can't control the values written. Since it is not possible to control the current predLine value:
```c
	upLeftBuf[0] = predLine[i];   <--- upLeftBuf is assigned with predLine[i]
```

The user controls the `predLine[i]` value at the end of the loop, but not the `preLine[i+1]`, that is the value used at the beggining to set `upLeftBuf`:

```c
    case 10:                    // PNG none
    default:                    // no predictor or TIFF predictor
      predLine[i] = (Guchar)c;    <--- The user controls c
```

This is **NOT EXPLOITABLE** since it is not possible to control the bytes written.
