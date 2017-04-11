In `xpdf/Gfx.cc:922`:
```
      funcs[0] = NULL;
      if (!obj2.dictLookup("TR", &obj3)->isNull()) {
        funcs[0] = Function::parse(&obj3);
        if (funcs[0]->getInputSize() != 1 ||	<--- funcs[0] is NULL, NULL dereference
            funcs[0]->getOutputSize() != 1) {
```

`funcs[0]` is assigned with the value returned by `Function::parse`. But this function can return a NULL value in some circumstances.
```
Function *Function::parse(Object *funcObj) {
  Function *func;
  Dict *dict;
  int funcType;
  Object obj1;
...
  } else if (funcType == 4) {
    func = new PostScriptFunction(funcObj, dict);
  } else {
    error(-1, "Unimplemented function type (%d)", funcType);
    return NULL;
  }
  if (!func->isOk()) {
    delete func;
    return NULL;
  }
```

The code should check the returned value before used it.`

This bug is **NOT EXPLOITABLE**. It is a NULL dereference.
