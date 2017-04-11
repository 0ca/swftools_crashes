In `xpdf/XRef.cc:183`:

```c
Object *ObjectStream::getObject(int objIdx, int objNum, Object *obj) {
  if (objIdx < 0 || objIdx >= nObjects || objNum != objNums[objIdx]) {  <--- objNums is NULL!
    return obj->initNull();
  }
  return objs[objIdx].copy(obj);
}
```

Parsing the PoC the variable `objNums` is NULL. So the program crashes because a NULL dereference.

The variable `objNums` should be initialized at the `ObjectStream` constructor at `xpdf/XRef.cc:109`:
```c
  objNums = (int *)gmallocn(nObjects, sizeof(int));
```

But under certain conditions the constructor could finish earlier and the variable doesn't get initiallized:
```c
  if (!objStr.streamGetDict()->lookup("First", &obj1)->isInt()) {
    obj1.free();
    goto err1;
  }
  first = obj1.getInt();
  obj1.free();
  if (first < 0) {
    goto err1;
  }

  objs = new Object[nObjects];
  objNums = (int *)gmallocn(nObjects, sizeof(int));
  offsets = (int *)gmallocn(nObjects, sizeof(int));
...
...
...
 err1:
  objStr.free();
  return;
}
```

Parsing the PoC the third `ObjectStream` created hasn't a dictionary with the `First` keyword and the constructor finish. Later when the method `getObject` is called, the program crashes.
```c
      objStr = new ObjectStream(this, e->offset);
    }
    objStr->getObject(e->gen, num, obj);   <--- Crash
```

The program should control when the constructor is returning an error and stop the execution.

This is **NOT EXPLOITABLE**, it is a NULL dereference.
