
# run_s0m3_w4r3z

run_s0m3_w4r3z was a task from WPI CTF 2019. Task description:

[![Screenshot_20190415_205919.png](https://wiki.armiaprezesa.pl/uploads/images/gallery/2019-04-Apr/scaled-840-0/naWK4btwDFm7RoDz-Screenshot_20190415_205919.png)](https://wiki.armiaprezesa.pl/uploads/images/gallery/2019-04-Apr/naWK4btwDFm7RoDz-Screenshot_20190415_205919.png)

***
I won't be going into much detail on how I reversed the functions, because it pretty much came down to renaming and retyping variables in ghidra - it decompiled everything pretty well.
***

So, there's a binary that asks for password:
```
./run_s0m3_w4r3z
Enter input:aaaaaaaaaaaaaaaa
Processing input...
Incorrect password!
```
After opening in ghidra and tweaking main function a bit I got:
```c
int main(int argc)

{
  time_t time1;
  time_t time2;
  time_t time3;
  long *local_FS_OFFSET_-1;
  char line [8];
  long canary;
  long canary_;

  canary_ = local_FS_OFFSET_-1[5];
  printf("Enter input:");
  time1 = time((time_t *)0x0);
  gets(line);
  _input = line;
  _canary = canary_;
  time2 = time((time_t *)0x0);
  time3 = time((time_t *)0x0);
  if (canary_ != local_FS_OFFSET_-1[5]) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return argc / (((int)time3 - (int)DAT_001070e0) - ((int)time2 - (int)time1));
}
```
That's definitely not what I expected. Where's the rest of program?
Turns out that it uses some form of obfuscation, whole program flow is not controlled as usual, but by signals and their handlers, and also there is code in functions like _FINI_1, _INIT_0, _INIT_1, etc.
So, I've got a bunch of functions and no information on how they interfere withc each other. Even better, most of data handling was trough global variables. Best of all, _input global variable and _canary global variable are overlapping. Sweet.
So I started looking trough functions, searching for intresting ones and reversing them.
To do that, the list of Strings is extremely useful. Turns out, string "Incorrect password!" is used only in one place (_FINI_1), let's reverse it a bit:
```c
void _FINI_1(void)

{
  long in_FS_OFFSET;
  undefined local_14 [4];
  long canary;

  canary = *(long *)(in_FS_OFFSET + 0x28);
  if (5 < DAT_001070bc) {
    wait(local_14);
    if (input_error_number == 2) {
      puts("Input too short!");
    }
    else {
      if (input_error_number == 3) {
        puts("Incorrect password!");
      }
      else {
        if (input_error_number != 0) {
          puts("Debugger detected!");
        }
      }
    }
  }
  if (canary == *(long *)(in_FS_OFFSET + 0x28)) {
    return;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
}
```
So, it uses `input_error_number` global variable to check if password entered is good, and values of 2, 3, 1 are bad, and value of 0 means good password. Let's check what functions can write 3 to this variable. There's again only one such function. Here it is reversed, but I'll explain a bit more on how to get it to such state:
```c
void check_password_is_valid(int param_1,siginfo_t *param_2,ucontext *param_3)

{
  byte bVar1;
  char cVar2;

  bVar1 = *(byte *)((param_3->uc_mcontext).gregs[0x10] + 4);
  cVar2 = *(char *)((param_3->uc_mcontext).gregs[0x10] + 5);
  (param_3->uc_mcontext).gregs[0x10] = (param_3->uc_mcontext).gregs[0x10] + 0xb;
  if ((uint)(byte)(&input)[(long)(int)(uint)bVar1] != (int)cVar2) {
    input_error_number = 3;
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  return;
}
```
Before setting correct types this func looks like garbage. How do we know that it takes a siginfo_t* and ucontext*? If we search for references to this func, there's one that looks intresting:

```c
void FUN_0010435b(void)

{
  long in_FS_OFFSET;
  int local_28c;
  undefined *func_input_too_short;
  undefined local_1e8 [160];
  undefined *func_input_check_valid;
  undefined local_a8 [152];
  ulong canary;

  canary = *(ulong *)(in_FS_OFFSET + 0x28);
  if (DAT_001070bc != DAT_001070cc) {
    wait(&local_28c);
                    /* WARNING: Subroutine does not return */
    exit(local_28c);
  }
  signal(6,exit);
  func_input_too_short = input_was_too_short;
  sigaction(8,(sigaction *)&func_input_too_short,(sigaction *)local_1e8);
  func_input_check_valid = check_password_is_valid;
  sigaction(0xb,(sigaction *)&func_input_check_valid,(sigaction *)local_a8);
  DAT_001070bc = 6;
  if (canary != *(ulong *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  DAT_001070bc = 6;
  return;
}
```
This function registers our check_password_is_valid function as handler for signal 11 (0xb, SIGSEGV). I checked in documentation what arguments must sigaction take and what parameters are passed to handler when process gets registered signal. As you saw in code before, those arguments are:
int - signal number
siginfo_t - some more informations about signal reveived. Unused in the task.
ucontext - it contains informations about program state, such as registers values.
Setting proper types in ghidra (and some renaming of course) gives us code I posted before. As you can see, it takes one of our input chars and compares it with some value. input index and that value are taken from ucontext struct, so program probably sets registers to certain values and then does something to make signal 11 happen. That's not really important. What's important is that we can now get the flag.
Battle plan:
1. Get check_password_is_valid offset in file and set breakpoint in gdb.
2. Check index of input and what value should it have.
3. Repeat as long as needed.

There was one problem left. As I said in the beggining, _input and _canary global variables are overlapping (_input is 16 bytes, last 8 ar overlapping with _canary, so if our input is more than 8 chars it will break the canary), and of course malformed _canary means program crash. If our input is too short, it is deemed as so, and app closes. Intrestingly, we can overwrite first byte or two of canary and it will work once every few tries. So, new battleplan:
1. Run the app with long enough input (9 chars, 8 is too short), works once every few tries.
2. Set 3 breakpoints: to check index, to check char, and to fix comparision by setting correct value in register
3. Breakpoints hit.
4. Check index of input and what value should it have, note id down.
5. Set value in register to correct value, so comparision works and program continues.
6. Go to step 3.

Script for gdb, you should be able to just ctrl+c ctrl+v it to gdb with binary open, and you should get output with indexes and chars:
```
handle all nostop noprint pass #program uses signals for flow control, we don't want to stop or care about them
del #delete other breakpoints, they would interfere with script
set follow-fork-mode parent #binary forks a few times, probably form of obfuscation/anti debugging
break *(0x555555556000+0x22BE+0x34)
break *(0x555555556000+0x22BE+0x49)
break *(0x555555556000+0x22BE+0x82)
r < <(echo 123456789) #don't do longer input, less chance of running successfully
while 1
p "index"
p $al
c

p "znak"
p $al
c

set $rdx=$rax # fix comparision
c
end
```
0x555555556000 is entry point, you may need to edit it.
This will only work once every few tries, so you just need to trie again if it says "Invalid password" before breaking anywhere.
After it works we get output:
```
$72 = "index"
$73 = 0

Breakpoint 2, 0x0000555555558307 in ?? ()
$74 = "znak"
$75 = 87

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$76 = "index"
$77 = 9

Breakpoint 2, 0x0000555555558307 in ?? ()
$78 = "znak"
$79 = 82

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$80 = "index"
$81 = 12

Breakpoint 2, 0x0000555555558307 in ?? ()
$82 = "znak"
$83 = 33

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$84 = "index"
$85 = 7

Breakpoint 2, 0x0000555555558307 in ?? ()
$86 = "znak"
$87 = 87

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$88 = "index"
$89 = 4

Breakpoint 2, 0x0000555555558307 in ?? ()
$90 = "znak"
$91 = 65

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$92 = "index"
$93 = 5

--Type <RET> for more, q to quit, c to continue without paging--c
Breakpoint 2, 0x0000555555558307 in ?? ()
$94 = "znak"
$95 = 83

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$96 = "index"
$97 = 8

Breakpoint 2, 0x0000555555558307 in ?? ()
$98 = "znak"
$99 = 79

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$100 = "index"
$101 = 11

Breakpoint 2, 0x0000555555558307 in ?? ()
$102 = "znak"
$103 = 33

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$104 = "index"
$105 = 3

Breakpoint 2, 0x0000555555558307 in ?? ()
$106 = "znak"
$107 = 80

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$108 = "index"
$109 = 2

Breakpoint 2, 0x0000555555558307 in ?? ()
$110 = "znak"
$111 = 87

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$112 = "index"
$113 = 1

Breakpoint 2, 0x0000555555558307 in ?? ()
$114 = "znak"
$115 = 79

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$116 = "index"
$117 = 6

Breakpoint 2, 0x0000555555558307 in ?? ()
$118 = "znak"
$119 = 83

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$120 = "index"
$121 = 13

Breakpoint 2, 0x0000555555558307 in ?? ()
$122 = "znak"
$123 = 33

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$124 = "index"
$125 = 14

Breakpoint 2, 0x0000555555558307 in ?? ()
$126 = "znak"
$127 = 49

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$128 = "index"
$129 = 15

Breakpoint 2, 0x0000555555558307 in ?? ()
$130 = "znak"
$131 = 33

Breakpoint 3, 0x0000555555558340 in ?? ()

Breakpoint 1, 0x00005555555582f2 in ?? ()
$132 = "index"
$133 = 10

Breakpoint 2, 0x0000555555558307 in ?? ()
$134 = "znak"
$135 = 68

Breakpoint 3, 0x0000555555558340 in ?? ()
Decrypting file...
File decrypted.
Here's your flag: flag{{�?z�X{�����Q��?}
[Inferior 1 (process 20873) exited normally]
$136 = "index"
```

So, input[0] = 87 (which is 'W'), input[9] = 82 etc.
Complete password:
WOWPASSWORD!!!1!
After feeding it to binary:
```
./run_s0m3_w4r3z
Enter input:WOWPASSWORD!!!1!
Processing input...
Decrypting file...
File decrypted.
Here's your flag: flag{s1gN4l_0r_n01S3?}
```
