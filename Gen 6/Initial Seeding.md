# Background Info

Most of the information in this document only applies to the emulator Citra since it provides a higher level of control then the 3DS console provides.

All of the offsets that will be discussed are from ORAS v1.2, however the mechanics work the same in XY.

# Seed generation
The seed generation is actually very simple. The game uses two variables that I will refer to as the time variable and save variable.

The time variable is the milliseconds since epoch. It is composed of a call to get the system time and the user time offset(this value is always 0 on Citra). The call to get the system time is sub_10d8f0 and the call to get the time offset is sub_10d918.

The save variable is a value that is changed everytime the game is saved. It grabs the prng state from the MT 16 frames after pressing A to save. This value is obtained from sub_1e78fc by reading the value at memory address 0x8c71db8 (or 0x8c6a6a4 in XY).

The initial seed is just a simple addition between these two values while only keeping the low 32bits.

```
initial_seed = (time_variable + save_variable) & 0xffffffff
```

# Abusing seed generation

I will dump my initial notes from my research then provide an example of how the process work.

```
1. Choose a base frame and base RTC
2. Determine the save parameter
3. Grab the seed that is generated
4. Base time variable = (generated seed - save parameter) & 0xffffffff
5. Even when save parameter changes the time variable will be constant
6. Add to the base time variable with the pattern
```

Thie process revolves around taking constant actions. This involves starting from the same base frame and the same RTC clock.

Step 1:

The base frame is the frame that an "A" press will lead to the game generating the seed(For XY this will be the first "A" press and for ORAS this will be the second "A" press). For me my base frame is 300.

The base RTC can be set in Citra's configuration. For me this is January 1, 2001 00:00:00.

What values you choose are not important. The important part is being consistent after choosing them.

Step 2:

This invovles reading from Citra's memory. I wrote a simple script I will include that can be used. My save variable is 0x3cd3267b.

```
from citra import Citra

def byte_to_hex(byteString):
    hexString = []
    for i in range(len(byteString)-1, -1, -1):
        hexString.append(format(byteString[i], "02x"))
    return ''.join(hexString)

def main():
    citra = Citra()

    go = 1
    while go == 1:
        address = int(input("Enter address to read from: 0x"), 16)

        data = citra.read_memory(address, 4)
        data = byte_to_hex(data)

        print(data)
        print("")

        go = int(input("Search again? (1/0)"))
        print("")

if __name__ == "__main__":
    main()
```

Step 3:

After pressing "A" to generate the seed you must find out what that seed is. This can be done using <CitraRNG>(https://github.com/Admiral-Fish/CitraRNG). From base frame 300 my initial seed is 0x99aad9fa.

Step 4:

There is actually two ways to determine the time variable. The harder way includes needing to dump the elf file from the game and debugging the game, which I will not cover. The simplier way involves a little bit of math. The formula to calculate time variable is as follows:

```
time_variable = (initial_seed - save_variable) & 0xffffffff
```

When I substitute in my values I get the following:

```
time_variable = (0x99aad9fa - 0x3cd3267b) 0xffffffff
```

This gives me a time_variable of 0x5cd7bc7f. You can verify this by using the initial_seed formula from above.

```
initial_seed = (0x3cd3267b + 0x5cd7bc7f) & 0xffffffff
```

This gives me a initial_seed of 0x99aad9fa which matches.

Step 5:

Even as the save_variable changes the time_variable will remain constant. This relies on two things, the first being that your base frame remains the same and second being that you always use the same RTC.

But on the bright side if either of those two things change it is still easy to calculate the time_variable for different base frames and RTC values.

Step 6:

This is where the fun part begins, actually manipulating the initial seed. This part involves knowing the "pattern" of your game. Determining the pattern is actually fairly simple. It just requires grabbing 4 different initial seeds starting from the base frame.

Citra with frame advances allows the game to advance 1 frame at a time when most mechanics don't actually change until 2 frames pass. So I will grab 3 more seeds with offsets of 2 frames from my base frame.

```
Frame 300: 0x99aad9fa
Frame 302: 0x99aada1c
Frame 304: 0x99aada3d
Frame 306: 0x99aada5e
```

The difference between frame 302 and 300 is 0x22. The difference between frame 304 and 302 is 0x21. The difference between 306 and 304 is 0x21.

This pattern of 0x22, 0x21, 0x21 will always be the same(note the order of your pattern might be different). From this pattern it is possible to calculate the initial seeds that are obtainable.

I wrote this script to do it(note it will require some slight modification if your pattern order is different):

```
time_variable = 0x5cd7b37f
save_variable = 0x3cd3267b

base = time_variable + save_variable

frames = int(int(input("How many frames: ")) / 2)

flag = 0
for i in range(frames):
    if flag == 0:
        base += 0x22
    elif flag == 1:
        base += 0x21
    elif flag == 2:
        base += 0x21

    flag = (flag + 1) % 3

print(hex(base))
```

So according to this script my initial_seed will be 0x99aae07d on frame 400. Verifying with Citra provides this as the initial_seed on frame 400.

# Next steps

This example relies on consistency, however the process is the same no matter what the variables are. This will allow for greating control of the initial_seed by changing the save_variable and time_variable.
