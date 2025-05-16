
## Introduction

Well the first thing I should do is ssh into the challenge to see what is going on exactly.
![[Pasted image 20250515182859.png]]
We are now in this ssh session.

![[Pasted image 20250515183123.png]]

We are greeted with a nice gui, and a note saying we can put files into /tmp.
Lets use the dir command to see what we have to work with.

![[Pasted image 20250515183244.png]]

We can see there are 3 files, flag, passcode and passcode.c.
Obviously flag is the file we want and it is placed there without any protection, lets try to cat it.

![[Pasted image 20250515183642.png]]

Those two files, passcode and passcode.c look interesting. But since this repo is focused on reverse engineering, unfortunately we don't have access to passcode.c. But we do have access to the binary. (At the end of **Analysis** we will see how close we were in mapping the functions)*

Lets use scp to copy the file into our vm so we can actually begin with the analysis.

![[Pasted image 20250515184151.png]]


---

## Analysis


Since we now have the file we can begin with gdb.

![[Pasted image 20250515184331.png]]

Well now I just realized I haven't even run the file yet.
But since I have already disassembled main we can continue from here.

Here are the things I notice:
- Up until main+26 is just function setup so we ignore it
- We see that we load an address and push it and call puts
- That address is a string
- Then we call welcome and login consecutively, and they have no arguments being pushed and their return value isn't used so they both are void(); functions
- Then we again print something and then just return.

Well here is my pseudo-code of the function based on the assembly:
main()
{
puts("somestring");
welcome();
login();
puts("somestring");
return;
}
Pretty short but that's the whole function. 

Well this function doesn't seem to do much except call the two functions **welcome** and **login** which are two void functions with no arguments.

So now we move onto the **welcome** function as that comes right after.

![[Pasted image 20250515185225.png]]

I notice that:
- At welcome+4 it allocates a significant amount of stack space
- welcome+39 it calls printf with a string, probably a welcome message
- At welcome+61 it calls scanf with a format and an address that's pretty high down, we will call this Buffer for now.
- Maybe this could be a vulnerability, depending on the format specifier.
- Then it calls printf with a global address and the same stack address, so it would be printf("formatspecifier", Buffer). This is just printing the value probably no other usages as we can't manipulate the formatspecifier, we just focus on our own inputs.
- Then we just return

This is my closest pseudo-code:
welcome()
{
printf(Welcome message);
Buffer\[SomeSize];
scanf(formatspecifier, Buffer);
printf(More message);

return;

}

Alright I can see one vulnerability in this, the scanf, depending on what the formatspecifier is.

Lets now dissect the login function, which is probably the most exciting, as the last two were not that great. Boo.

Well the login seems to be the biggest function, which makes me very excited.

![[Pasted image 20250515191323.png]]
![[Pasted image 20250515191344.png]]


Well the function is so big that I needed two pages on my terminal, but lets get back on track. These are the things I notice:
- Ignoring the prologue, we can see at login+29 a call to printf, likely a message saying something related to logging in.
- We see the next call at login+50 is a scanf call, but it seems it didn't allocate much space for the stack, this is another case for a vulnerability, we will be able see when we reveal the c code whether we are right or wrong.
- Now looking at the argument to scanf I see something very interesting, this could go unnoticed very easily both in the c code and in the assembly as well. We see that scanf, the first argument is the constant string, and the second argument, \[ebp-0x10] is rather interesting. We directly push the uninitialized \[ebp-0x10] as an argument. Lets continue
- Then after that is a call to fflush at login+70, nothing exploitable about it.
- Then it calls printf with a constant string at login+88, nothing weird here eaither.
- Then again, at login+99 to login+109 we see the same pattern or bug as before, it directly pushes the uninitialized local variable as an argument to scanf.
- Then it calls puts at login+127 with a constant argument, there is nothing exploitable here.
- From login+135 to login+151 it seems to be comparing the two bugged variables into two nested if statements or one if statement with && comparison
- If both conditions are true, from login+163 to login+206 it seems to print a message, likely a congratulation message based on the context as it uses setregid(getegid(), getegid()) and a system call probably going to give us the flag.
- If those any of those conditions are not satisfied, it puts a constant string and uses exit, so this is the failure point
- At the end is the graceful exit if the two conditions are satisfied.

Here is my pseudo-code:
login()
{
buffer1\[SomeSize]; likely 3 bytes
buffer2\[SomeSize]; likely 3 bytes
printf(Message);
scanf(SomeSpecifier, \*(Buffer1));
fflush();
scanf(SomeSpecifier, \*(Buffer2));

if( not compare buffer1 and buffer2)
{
Faliure();
}

Success();

}



Well so that was a more exciting function, well now what I wonder is how to get those two conditions to be true, because its pretty much impossible to get that to be true. What I do notice about the scanf is that if we can somehow control the uninitialized variable at \[ebp-0x10] or \[ebp-0xc] then we win because we can change those addresses to anything, I am also not sure if this is a buffer overflow as the previous welcome call would be the buffer overflow, there is no need for two functions.

Well since we can't figure out anything from this, lets look at the c code.

Here's the c code:
![[Pasted image 20250515194025.png]]

Opening it up using nano, lets first compare my pseudo-code, also from the length of the tab you can probably tell I am a beginner.

<details>
<summary>main pseudo-code</summary>
main()
{
puts("somestring");
welcome();
login();
puts("somestring");
return;
}

</details> 

It seems pretty accurate except the puts and printf call, and the return 0, but those are minor details that don't matter. I guess the compiler sometimes replaces printf with puts if no format specifier is provided.

<details>
<summary>welcome pseudo-code</summary>
welcome()
{
printf(Welcome message);
Buffer\[SomeSize];
scanf(formatspecifier, Buffer);
printf(More message);

return;

}
</details>

This one is pretty accurate, the only thing missing was the constant format specifier obviously I didn't look for it so its fine.

<details>
<summary>login pseudo-code</summary>
login()
{
buffer1\[SomeSize]; likely 3 bytes
buffer2\[SomeSize]; likely 3 bytes
printf(Message);
scanf(SomeSpecifier, \*(Buffer1));
fflush();
scanf(SomeSpecifier, \*(Buffer2));

if( not compare buffer1 and buffer2)
{
Faliure();
}

Success();

}
</details>

This one had the most mess ups, the first is that it's an integer not a character buffer, but I guess they are both the same at the lowest level. Alright so it seems to actually write to the address of the two integers. 

I think I have understood how to solve this challenge, as I have seen this pattern before. We can indirectly control the two integers, because functions called from the same parent function will have the same base pointer, as when a function returns it clears up and sets stack and base to its previous state, that means that if a variable in the welcome function is written, and not cleared, then that means if another function is called before that variable is cleared, It will be at the same address, and will have the same value.

So this can be seen, in the welcome disassembly, it asks for input of exactly 100 characters, no vulnerability here, it starts from \[ebp-0x70] then we stop writing at \[ebp - 0xC], exactly a 100 bytes, but don't focus on these in the context of the welcome function but the context of the login function. In the login function, it uses an uninitialized variable at \[ebp - 0x10] at exactly 4 bytes away from our allocation, this isn't a coincidence, that means we can control that value as the last 4 bytes of the name variable in the welcome function.

Directly after the scanf is the fflush function, and if we disassemble the fflush function:

![[Pasted image 20250515200104.png]]

Well how nice, its a got thing, as I am a beginner, I just know that as jump tables, I guess its the same thing, but anyways, here we see it jumps to the address pointed by 0x804c014. That means if we overwrite the value inside 0x804c014 we can control the execution flow, but we should just set it to after we pass from the variable comparison, right here:
![[Pasted image 20250515200433.png]]


Alright lets move on to the Exploit.

---

## Exploit

Here is the python I used and a detailed analysis of it.
import sys;
sys.stdout.buffer.write(b'A' * 96 + b'\x14\xc0\x04\x08' + b'\n' + b'124517409' + b'\n');

That's, just one line and we have the flag.

Here is what it does.
Well first thing we do is just fill the bytes using this python code, worth 96 bytes. Then the 4 bytes in little endian is the input to scanf, the address that stores where fflush will jump to. Then we have a newline to signal to scanf we are done. Then we add the decimal form of the hexadecimal as thats what scanf takes and it writes to that address and another newline to signal stop.

Well that was it.



---

## Reflections

Overall this challenge was mild, It wasn't that hard as the vulnerability was pretty clear. I just reinforced my previous understandings of things on this challenge. I know that the future challenges will get more and more difficult but I will push through.