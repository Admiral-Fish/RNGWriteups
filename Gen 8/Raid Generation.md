# Background Info

This writeup will be split into two sections. One section will be about how the generation of the den type and why this isn't RNGable. The second section will detail how a raid is generated.

# Den generation
Den generation occurs at sub_7100E91740. The RNG for den/raid generation is [xoroshiro128+](http://prng.di.unimi.it/xoroshiro128plus.c).

Like [egg generation](https://github.com/Admiral-Fish/RNGWriteups/blob/master/Gen%208/Egg%20Generation.md) it uses nn::crypto::GenerateCryptographicallyRandomBytes which makes the den type unable to be RNGed. This function is responsible for setting the den seed, star count, species, rarity, etc.

# Raid Generation

Raid generation occurs at sub_710076FAD0. An important note is that the call to the RNG is a slightly modified version of xoroshiro128+. A TLDR is that it will modulo the result by the next power of two until the result is less then the desired max value. It is as follows:

```        
nextState():
    # Refer to the link above to see how xoroshiro128+ generates the next state and rand value

nextPowerTwo(num):
    if num & (num - 1) == 0:
        return num - 1

    result = 1
    while result < num:
        result <<= 1
    return result - 1

next(maxValue):
    mask = nextPowerTwo(maxValue)

    result = nextState() & mask
    while result >= maxValue:
        result = nextState() & mask
    return result
```

Step 1. EC

The encryption constant is set a value of next(0xFFFFFFFF).

```
ec = next(0xFFFFFFFF)
```

Step 2. TID/SID

A temporary TID/SID is set to a value of next(0xFFFFFFFF).

```
otid = next(0xFFFFFFFF)
```

Step 3. PID

The PID is set a value of next(0xFFFFFFFF).

```
pid = next(0xFFFFFFFF)
```

Step 4. Shiny

Shininess for the raid is determined by the temporary TID/SID. Later on any PID modifications will be done using the real TID/SID

```
otsv = ((otid >> 16) ^ (otid & 0xFFFF)) >> 4
psv = ((pid >> 16) ^ (pid & 0xFFFF)) >> 4

if otsv == psv: # Shiny
    shiny = True
    
    if (otid >> 16) ^ (otid & 0xffff) ^ (pid >> 16) ^ (pid & 0xffff):
        shinyType = 2
    else:
        shinyType = 1
    
    if psv != realTSV: # Force PID to be shiny from the real TID/SID
        high = (pid & 0xFFFF) ^ realTID ^ realSID ^ (shinyType == 1)
        pid = (high << 16) | (pid & 0xFFFF)
else: // Not shiny
    shiny = False
    if psv == realTSV: # Force PID to be not shiny from the real TID/SID
        pid ^= 0x10000000
```

Step 5. IVs

The amount of 31 IVs that there will be is dependent on the star count of the den type using next(6) calls. After the 31 IVs are assigned the rest of the IVs are filled using next(32)

```
ivs = [-1]*6
i = 0
while i < ivCount:
    stat = next(6)
    if ivs[stat] == -1:
        ivs[stat] = 31
        i += 1

for i in range(6):
    if ivs[i] == -1:
        ivs[i] = next(32)
```

Step 6. Ability

There are two ability groupings a raid can have. A raid can either pull from Ability0/Ability1 (next(2)) or Ability0/Ability1/Ability2 (next(3)).

```
if abilityType == 4: # Allow hidden ability
    ability = next(3)
elif abilityType == 3: # Don't allow hidden ability
    ability = next(2)
```

Step 7. Gender

There are 4 different gender types a raid can have. It can either be random, female, male, or genderless. If it's random it uses a rand(253) call and compares to the gender ratio for female/male.

```
if genderType == 0: # Random gender
    if genderRatio == 255: # Locked genderless
        gender = 2
    elif genderRatio == 254: # Locked female
        gender = 1
    elif genderRatio == 0: # Locked male
        gender = 0
    else: # Random gender
        gender = next(253) + 1 < genderRatio
elif genderType == 1: # Male
    gender = 0
elif genderType == 2: # Female
    gender = 1
elif genderType == 3: # Genderless
    gender = 2
```

Step 8. Nature

Nature is set to a value of next(25). If the raid pokemon is Toxtricity then there is some special handling. For raids Toxtricity appears to be only allowed Amped Form.

```
if species != Toxtricity:
    nature = next(25)
else:
    natures = [3, 4, 2, 8, 9, 19, 22, 11, 13, 14, 0, 6, 24]
    nature = natures[next(13)]
```

Step 9. Height/Weight

Height and weight are set to a value of next(129) + next(128).

```
height = next(129) + next(128)
weight = next(129) + next(128)
```