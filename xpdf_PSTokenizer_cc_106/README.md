In `xpdf/PSTokenizer.cc:106`:

```c
  } else if (c != '[' && c != ']') {
    while ((c = lookChar()) != EOF && !specialChars[c]) {  <--- Using a negative index to read a value
      getChar();
      if (i < size - 1) {
        buf[i++] = c;
      }
    }
  }

```

the variable `c` is defined as an `int` (`signed int`). The array `specialChars` is a 256 bytes array. 

But parsing the PoC the variable c gets a negative number -120. And that number is used as an index, underflowing the array and reading below the variable.

This doesn't make the program inmediatelly to crash. In the same PoC there is another bug that makes the program to crash: https://github.com/illera88/Oceanic/tree/master/swftools/crashes/xpdf_GString.h_80

