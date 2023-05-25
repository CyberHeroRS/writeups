# Prompt

>See if you can find the flag!

Attachment: gettingBetter

# Solution
<p> As usual, the first thing to do is see if there's anything intersting using the 'strings' command and the only interesting thing seemed to be the following string </p>

>Xmj%yzwsji%rj%nsyt%f%sj|y

<p>I decided to open the program in ghidra to see what's going on. I found the following code in main: </p>

```c
    local_108 = 0x6e806b79687a7e67;
    local_100 = 0x4c64;
    uStack_fe = 0x796a38647935;
    uStack_f8 = 0x6a59;
    local_f6 = 0x823a3c3e36642657;
    decrypt_passphrase(&local_108,local_178,5);
    print_flag(local_178);]
```

<p> It seems like a string we could use as it's passed to a function decrypt_passphrase as an argument. After taking a look at the function and changing some variable names, we come to see the following code: </p>

```c
    void decrypt_passphrase(long in_str,long out_str,char int_5)

    {
    int i;
    
    for (i = 0; *(char *)(in_str + i) != '\0'; i = i + 1) {
        *(char *)(out_str + i) = *(char *)(in_str + i) - int_5;
    }
    *(undefined *)(out_str + i) = 0;
    return;
    }
```

<p> What this does is simply subtract 5 from the ascii value of each character. We can easily do this ourselves with some simple python code. Note that chars are read right-to-left. The following is the code I used to get the flag: </p>

```py
a = [0]*5 #the following are just strings that i found in ghidra
a[0] = '6e806b79687a7e67' 
a[1] = '4c64'
a[2] = '796a38647935'
a[3] = '6a59'
a[4] = '823a3c3e36642657'
s = "".join([(bytes.fromhex(substr).decode('latin-1'))[-1::-1] for substr in a])
print("".join([chr(ord(c)-5) for c in s]))
```

<p> The flag is the following </p>

> byuctf{i_G0t_3etTeR!_1975}