In `plashFTFontFile.cc:41`:
```c
SplashFontFile *SplashFTFontFile::loadType1Font(SplashFTFontEngine *engineA,
                                                SplashFontFileID *idA,
                                                char *fileNameA,
                                                GBool deleteFileA,
                                                char **encA) {
  FT_Face faceA;
  Gushort *codeToGIDA;
  char *name;
  int i;

  if (FT_New_Face(engineA->lib, fileNameA, 0, &faceA)) {
    return NULL;
  }
  codeToGIDA = (Gushort *)gmallocn(256, sizeof(int));
  for (i = 0; i < 256; ++i) {
    codeToGIDA[i] = 0;
    if ((name = encA[i])) {
      codeToGIDA[i] = (Gushort)FT_Get_Name_Index(faceA, name);   <--- when i = 3 name pointing to an invalid memory
    }
  }
```

`FT_Get_Name_Index` is a freetype library function. And it crashes inside the function because name is pointing to an unallocated area.

This is **NOT EXPLOITABLE**, since the crash happens reading from an unmap memory region.

The problem comes because `encA` has only 3 elements and in the loop 256 items are being accessed.

`encA` comes from `xpdf/SplashFTFontEngine.cc:77`:
```c
SplashFontFile *SplashFTFontEngine::loadType1CFont(SplashFontFileID *idA,
                                                   char *fileName,
                                                   GBool deleteFile,
                                                   char **enc) {
  return SplashFTFontFile::loadType1Font(this, idA, fileName, deleteFile, enc);
}
```

`enc` comes from: `xpdf/SplashFontEngine.cc:152`:
```c
SplashFontFile *SplashFontEngine::loadType1CFont(SplashFontFileID *idA,
                                                 char *fileName,
                                                 GBool deleteFile,
                                                 char **enc) {
  SplashFontFile *fontFile;

  fontFile = NULL;
#if HAVE_T1LIB_H
  if (!fontFile && t1Engine) {
    fontFile = t1Engine->loadType1CFont(idA, fileName, deleteFile, enc);
  }
#endif
#if HAVE_FREETYPE_FREETYPE_H || HAVE_FREETYPE_H
  if (!fontFile && ftEngine) {
    fontFile = ftEngine->loadType1CFont(idA, fileName, deleteFile, enc);
  }
```

And again `enc` comes from `xpdf/SplashOutputDev.cc:1097`:
```c
      if (!(fontFile = fontEngine->loadType1CFont(
                           id,
                           fileName->getCString(),
                           fileName == tmpFileName,
                           ((Gfx8BitFont *)gfxFont)->getEncoding()))) {
```

`getEncoding` return the `encoding` variable, and 256 array with a set of strings set in the function `void FoFiType1C::buildEncoding()` at `xpdf/FoFiType1C.cc:2258`.
```c
encoding[c] = copyString(getString(charset[i], buf, &parsedOk));
```

It looks like the array is not completelly filled, only the first three positions are assigned, and that is the reason the program crashes when values greater than 3 are accessed.
