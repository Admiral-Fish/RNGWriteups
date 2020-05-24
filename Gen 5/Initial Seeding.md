# Seed Generation (non C-Gear)
This generation probably has some of the most unique seeding of all mainstream Pokemon games. It utilizes [SHA-1](https://en.wikipedia.org/wiki/SHA-1) with a number of parameters to generate the initial seed whenever the game is turned on. These parameters include the language of the game, version of the game, DS type, Timer0, VCount, MAC address of the DS, whether or not the game is being soft reset, VFrame, GxStat, date/time, and keys that are pressed during boot. Let's go over how each parameter impacts the SHA-1 seed generation.

For the purpose of describing how SHA-1 interacts with these parameters I will be talking in terms of an array of size 16 which SHA-1 uses to compute the initial seed.

Language, Version, and DS Type:
Each combination of language (English, Japanese, German, Spanish, French, Italian, Korean), version (Black, White, Black2, White2), and DS type(DS Phat/Lite, DSi, 3DS) have 5 unique values that are called Nazos. These Nazos take indexes 0-4 of the array and the values for each combo can be found [here](https://github.com/Admiral-Fish/PokeFinder/blob/master/Core/Gen5/Nazos.cpp#L24-L85).

VCount and Timer0:

These two values contribute to index 5 of the message. VCount is left shifted by 16 and is bitwise ORed with the Timer0. The final step is to change the endianness of the result.

MAC Address and Soft Reset:

These two values contribute to index 6 of the message. The lowest 16 bits of the MAC address is taken. If the game was soft reset then bitwise XOR this value with 0x01000000

MAC Address, VFrame, and GxStat:

These three values contribute to index 7 of the message. VFrame is left shifted by 24 and gets bitwise XORed with the GxStat. This value then gets bitwise XORed with the MAC address after bitshifting it right by 16.

Date:

The date contributes to index 8 of the message. The three main components of the date are year, month, and day. These three values get converted into binary-coded decimal (BCD). The BCD of the year gets left bitshifted by 24, the BCD of the month gets left bitshifted by 16, and the BCD of the day gets left bitshifted by 8. These three values get bitwise ORed together. Finally the day of the week is bitwise ORed into this value.

```
1: Monday
2: Tuesday
...
7: Sunday
```

Time and DS Type:
These two values contribute to index 9 of the message. The three main components of the time are hour, minute, and second. These three values also get converted into BCD. The special part is the hour. If the hour is greater than 12 and the DS type is not a 3DS then add 0x40 to this value. The adjusted BCD of hour gets left bitshifted by 24, the BCD of the minute gets left bitshifted by 16, and the BCD of the second gets left bitshifted by 8. These three values get bitwise ORed together.

Keypresses:
The keypresses contribute to index 12 of the message. The base value starts from 0xff2f0000 and for each key that is pressed a specific number will be subtracted from this base value.

```
R: 0x10000
L: 0x20000
X: 0x40000
Y: 0x80000
A: 0x1000000
B: 0x2000000
Select: 0x4000000
Start: 0x8000000
Right: 0x10000000
Left: 0x20000000
Up: 0x40000000
Down: 0x80000000
```

Overall:
```
changeEndian(value):
    val = ((val << 8) & 0xFF00FF00) | ((val >> 8) & 0xFF00FF)
    return (val << 16) | (val >> 16)
```

```
bcd(value):
    tens = value / 10
    ones = value % 10

    return (tens << 4) | ones
```

```
message[0] = nazo[0]
message[1] = nazo[1]
message[2] = nazo[2]
message[3] = nazo[3]
message[4] = nazo[4]
message[5] = changeEndian((vcount << 16) | timer0)
message[6] = (mac & 0xffff) ^ (softReset ? 0x01000000 : 0)
message[7] = (mac >> 16) ^ (vframe << 24) ^ gxstat
message[8] = (bcd(year - 2000) << 24) | (bcd(month) << 16) | (bcd(day) << 8) | dayOfWeek
message[9] = ((bcd(hour) + (hour >= 12 and dsType != 3DS ? 0x40 : 0)) << 24) | (bcd(minute) << 16) | (bcd(second) << 8)
message[10] = 0
message[11] = 0
message[12] = keypresses

The rest of the message is per the standard of SHA-1, marking the end, padding with 0s, and putting the size of our message at the end
message[13] = 0x80000000
message[14] = 0
message[15] = 0x1a0
```

Running this through SHA-1 we will look at the final values of h0, h1, h2, h3, and h4. The values of h0 and h1 are used to compute the initial seed. Both h0 and h1 will have their endianness changed. Then b will be left bitshifted by 32 and then bitwise ORed with a. Finally this value will be multipled by 0x5d588b656c078965 and then have 0x269ec3 added to it.

Example:
For this example the default parameters of Pokemon White, English using Desmume will be used. The date/time will be Jan 1, 2000 at 00:00:00.

```
Language = English
Version = White
DS Type = DS Phat/Lite
Timer0 = 0x621
VCount = 0x2f
MAC = 0x9BF123456
Soft Reset = false
VFrame = 0x5
GxStat = 0x6
Date = 1/1/2000
Time = 00:00:00
Keys = None
```

```
message[0] =  0xd0602102
message[1] =  0xcc612102
message[2] =  0xcc612102
message[3] =  0x18622102
message[4] =  0x18622102
message[5] =  0x21062f00
message[6] =  0x00003456
message[7] =  0x0509bf14
message[8] =  0x00010106
message[9] =  0x00000000
message[10] = 0x00000000
message[11] = 0x00000000
message[12] = 0xff2f0000
message[13] = 0x80000000
message[14] = 0x00000000
message[15] = 0x000001a0
```

The final h0 and h1 values are 0x16a4a8f6 and 0x9a2bb383. After changing the endianness we get 0xf6a8a416 and 0x83b32b9a. After the bitwise operations, multiplication, and adding the final initial seed is 0xb082b4a755192171.

# Seed Generation (C-Gear)
Compared to non C-Gear seed generation, this is a much simplier process that uses less parameters and doesn't rely on SHA-1. The parameters include date/time, delay, and the MAC address. For those that are familiar with Generation 4 this will look very familiar.

First we will break our parameters into the necessary sub parts.

```
ab = ((month * day) + minute + second) & 0xff
cd = hour
efgh = delay + (year - 2000)
partial_mac = mac & 0xffffff
```

With these sub parts we can compute the initial seed
```
seed = (ab << 24) + (cd << 16) + efgh + partial_mac
```