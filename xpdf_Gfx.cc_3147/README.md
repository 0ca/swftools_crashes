In `xpdf/Gfx.cc:3147`:

```c
void Gfx::doShowText(GString *s) {
  GfxFont *font;
  int wMode;
  double riseX, riseY;
  CharCode code;
  Unicode u[8];
  double x, y, dx, dy, dx2, dy2, curX, curY, tdx, tdy, lineX, lineY;
  double originX, originY, tOriginX, tOriginY;
  double oldCTM[6], newCTM[6];
  double *mat;
  Object charProc;
  Dict *resDict;
  Parser *oldParser;
  char *p;
  int len, n, uLen, nChars, nSpaces, i;

  font = state->getFont();
  wMode = font->getWMode();   <-- font has the value NULL
```

This is **NOT EXPLOITABLE**. It is NULL dereference.

ToDo: Identify exactly why:
* Where is it set the font attribute for the state instance?
