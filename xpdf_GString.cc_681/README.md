In `xpdf/GString.cc:681`:

```c
int GString::cmp(const char *sA) {
  int n1, i, x;
  const char *p1, *p2;

  n1 = length;   <--- The code is accesing to GString->n1, but GString has not been assigned.
```

`GString::cmp` is called from `xpdf/Annot.cc:300`:
```c
  // must be a Widget annotation
  if (type->cmp("Widget")) {
    return;
  }
``` 

The problem is `type` is NULL so when the static method cmp is called but the object GString has the value NULL.

The variable `type` is allocated at the `Annot` Constructor:
```c
Annot::Annot(XRef *xrefA, Dict *acroForm, Dict *dict, Ref *refA) {
  Object apObj, asObj, obj1, obj2, obj3;
  AnnotBorderType borderType;
  double borderWidth;
  double *borderDash;
  int borderDashLength;
  double borderR, borderG, borderB;
  double t;
  int i;

  ok = gTrue;
  xref = xrefA;
  ref = *refA;
  type = NULL;
  appearBuf = NULL;
  borderStyle = NULL;

  //----- parse the type

  if (dict->lookup("Subtype", &obj1)->isName()) {
    type = new GString(obj1.getName());  <--- type allocated
  }
```

But if the `Subtype` field is not found in the pdf document, then the type is not set and when in the function `Annot::generateFieldAppearance` is executed the crash happens.

This is **NOT EXPLOITABLE**. 
