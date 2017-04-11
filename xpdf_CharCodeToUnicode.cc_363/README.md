In `xpdf/CharCodeToUnicode.cc:363`:
```c
void CharCodeToUnicode::addMapping(CharCode code, char *uStr, int n,
                                   int offset) {
...
      }
    }
    sMap[sMapLen].u[sMap[sMapLen].len - 1] += offset;    <-- Buffer overflow
    ++sMapLen;

```

sMap is a variable of type: `CharCodeToUnicodeString *`:
```c
#define maxUnicodeString 8

struct CharCodeToUnicodeString {
  CharCode c;
  Unicode u[maxUnicodeString];
  int len;
};
```

So the u varibale has 8 bytes with the unicode representation. But in the PoC the 
```c
sMap[sMapLen].u[sMap[sMapLen].len - 1]
```

The `sMap[sMapLen].len` is calculated a few lines before:
```c
    map[code] = 0;
    sMap[sMapLen].c = code;
    sMap[sMapLen].len = n / 4;
```
`n` is one of the argument of the current functio: `CharCodeToUnicode::addMapping`. pdf2swf crashes when n is bigger than 32 (8 * 4). And this happens in the PoC where n has the value 198.

This value comes from the lines, 305:
```c
          tok3[n3 - 1] = '\0';
          for (i = 0; code1 <= code2; ++code1, ++i) {
            addMapping(code1, tok3 + 1, n3 - 2, i);
          }

```
`n3 -2` and n3 is assigned some lines before while parsing the pdf document:
```c
        if (!pst->getToken(tok2, sizeof(tok2), &n2) ||
            !strcmp(tok2, "endbfrange") ||
            !pst->getToken(tok3, sizeof(tok3), &n3) ||
            !strcmp(tok3, "endbfrange")) {
```


The program is not validating that the len of an unicode encoded character is bigger than 32 bytes, so the buffer storing every unicode character is overflown.

The value written during the overflow is not controlled by the user, it is an incremental number. So this is **NOT EXPLOITABLE**, since the user doesn't control the data written during the overflow,
