# Hfs-mbr
#### Author: Paweł Wieczorek aka cerber-os
In this challange we were given a disk image containing Bootloader + FAT32 filesystem. Our goal was to retrieve the password required to boot to the FreeDos system. Due to the lack of remote debugging support in IDA Free (as far as we know it is only available in IDA Pro), we were forced to use hybrid method of analyzing bootloader.

###### Stage 1.
First of all, we've attached gdb and set breakpoint at the start of bootloader (the address was given in README file). Next we've dumped next few kilobytes of program:
```
dump binary memory dump.bin 0x7c00 0x7fff
```
and analyzed it with IDA. It was stage 1 of bootloader which performs basic initialization of system and loads the next part.

###### Stage 2.
After setting next breakpoint and executing similar command in gdb (`dump binary memory dump.bin 0x7e00 0x7fff`), we successfully extracted the most interesting part of bootloader - the password checker. After performing an analyze of the assembly code, we translated it to the following C code:
```c
int verifyPassword(void) {
  char correctness = 0;
  char letterCnt = 0;
  char dl = 0;
  char i = 0;
  while(1) {
    char chr = getChar();
    dl = chr;
    i++;
    if(chr < 'a' || chr > 'z') {
        std::cout << "Invalid character in string" << std::endl;
        return 0;
    }
    switch(chr) {
        case 'e':
            dl ^= letterCnt;
            dl -= 0x60;
            if(dl == 2) goto stepCloserToWin;
            else        goto addCharacter;
            break;
        case 'j':
            dl ^= letterCnt;
            dl -= 0x30;
            if(dl == 0x38) goto stepCloserToWin;
            else           goto addCharacter;
            break;
        case 'n':
            dl ^= letterCnt;
            if(dl == 0x68) goto stepCloserToWin;
            else           goto addCharacter;
            break;
        case 'o':
            dl ^= letterCnt;
            dl += 0x20;
            if(dl == 0x8e) goto stepCloserToWin;
            else           goto addCharacter;
            break;
        case 'p':
            dl ^= letterCnt;
            dl += 20;
            if(dl == 0x88) goto stepCloserToWin;
            else           goto addCharacter;
            break;
        case 'r':
            dl ^= letterCnt;
            dl += 0x32;
            if(dl == 0xac) goto stepCloserToWin;
            else           goto addCharacter;
            break;
        case 's':
            dl ^= letterCnt;
            if(dl == 0x73) goto stepCloserToWin;
            else           goto addCharacter;
            break;
        case 'u':
            dl ^= letterCnt;
            dl -= 0x30;
            if(dl == 0x46) goto stepCloserToWin;
            else           goto addCharacter;
            break;
        case 'w':
            dl ^= letterCnt;
            dl += 0x10;
            if(dl == 0x82) goto stepCloserToWin;
            else           goto addCharacter;
            break;
        default:
            goto addCharacter;
            break;
    }
    stepCloserToWin:
        correctness++;
        if(correctness == 9)
            return 1;
    addCharacter:
        letterCnt++;
        if(letterCnt == 9)
            return 0;
  }
}
```
From the code above, it's clear that the password must be 9 character long and consists only of letters "ejnoprsuw". The entered character is being xored with its position in input and compared with some constant value.

```text
For example reversing letter "e": 
e in ascii: 1100101
0x60 + 2:   1100010
xor:		0000111 == 7
```

So letter "e" should be 7th letter in password (counting from 0).
After performing the same operation on all 9 letters we quickly reversed the correct password: `sojupwner`.
When bootloader accepts password, it loads the 3. stage, which prints the flag stored in file `FLAG1` and opens shell (which we have pwned as a part of the next challenge - **Hfs-dos**).
