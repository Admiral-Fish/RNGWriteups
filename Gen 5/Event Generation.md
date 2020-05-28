This is the one of the first main times where the wondercard file greatly impacts the generation of the resulting pokemon.

# Wondercard Structure
The full wondercard structure can be seen [here](https://projectpokemon.org/docs/gen-5/5th-generation-wondercard-map-r2/) but I will highlight the important parts for RNG.

| Offset    | Description | Value notes                                                   |
|-----------|-------------|---------------------------------------------------------------|
| 0x00-0x01 | TID         |                                                               |
| 0x02-0x03 | SID         |                                                               |
| 0x1A-0x1B | Species     |                                                               |
| 0x34      | Nature      | 0xff is random nature                                         |
| 0x35      | Gender      | 0: male, 1: female, 2: random                                 |
| 0x36      | Ability     | 0: ability 1, 1: ability 2, 2: hidden ability, 3: ability 1/2 |
| 0x37      | Shiny       | 0: not shiny, 1: random, 2: always shiny                      |
| 0x43-0x48 | IVs         | 0xff is random IV                                             |
| 0x5C      | Egg         | 0: not egg, 1: is egg                                         |

# Wondercard Loading
Before the generation of the wondercard can occur the game must first setup and load the wondercard. This process can advance a number of frames between 8 and 24. First a base number of 8 frames are advanced. Then for each random IV 2 frames will be advanced, if the gender is male or female 2 frames are advance, and if the nature is random 2 frames are advanced.

```
advances = 8

# IV Advances
for i in range(0x43, 0x49):
    if wondercard[i] == 0xff:
        advances += 2

# Gender advance
if wondercard[0x35] == 0 || wondercard[0x35] == 1:
    advances += 2

# Nature advance
if wondercard[0x34] == 0xff:
    advances += 2
```

# Wondercard Generation
Once the wondercard has been loaded and those initial frames have been consumed generation can begin. I will first begin by explaining the helper functions that contribute to the generation including the RNG generation, gender forcing, and shiny forcing.

```
def next(prng):
    prng = (prng * 0x5d588b656c078965 + 0x269ec3) & 0xffffffffffffffff
    return prng, prng >> 32

def next(prng, max):
    prng, rand = next(prng)
    rand = (rand * max) >> 32
    return prng, rand

def force_non_shiny(pid, tid, sid):
    if (pid >> 16) ^ (pid & 0xffff) ^ tid ^ sid < 8:
        pid ^= 0x100000000
    return pid

def force_shiny(pid, tid, sid):
    low = pid & 0xff
    pid = ((low ^ tid ^ sid) << 16) | low

def force_gender(pid, rand, gender, gender_ratio):
    pid &= 0xffffff00
    val = 0

    if gender_ratio == 0: # Male only
        val = ((rand * 0xf6) >> 32) + 8
    elif gender_ratio == 254: # Female only
        val = ((rand * 0x08) >> 32) + 1
    else: # Random gender
        if gender == 0: # Male
            val = ((rand * (0xFE - gender_ratio)) >> 32) + gender_ratio
        elif gender == 1: # Female
            val = ((rand * (gender_ratio - 1)) >> 32) + 1

    return pid | val
```

Before generation can occur we must first determine what TID/SID to use, our own or from the event. It is easy to determine this from whether the event is an egg or not.

| Egg | TID/SID       |
|-----|---------------|
| Yes | Our TID/SID   |
| No  | Event TID/SID |

Step 0: Wondercard Loading

The frames described above must be advanced through.

```
for i in range(advances):
    prng, _ = next(prng)
```

Step 1: IVs

For each IV that is random we generate an IV otherwise just copy over from the wondercard file.

```
for i in range(6):
    template_iv = wondercard[0x43 + i]
    if template_iv == 0xff:
        prng, rand = next(prng)
        ivs[i] = rand >> 27
    else:
        ivs[i] = template_iv
```

Step 1.5: Blanks

2 blank frames get consumed in between the IVs and PID.

```
for i in range(2):
    prng, _ = next(prng)
```

Step 2: PID

The upper 32bits of the next prng state is assigned to the PID.

```
prng, rand = next(prng)
pid = rand
```

Step 3: Gender

Depending on the wondercard file the PID will potentially get modified to satisify the gender requirements. If modification is required a prng call is required.

```
if wondercard[0x35] == 0 || wondercard[0x35] == 1:
    prng, rand = next(prng)

    # Note that the gender ratio doesn't get stored in the wondercard structure
    # However, it is not hard to know given the species
    pid = force_gender(pid, rand, wondercard[0x35], gender_ratio)
    gender = wondercard[0x35]
else:
    if genderRatio == 255: # locked genderless
        gender = 2
    elif genderRatio == 254: # locked female
        gender = 1
    elif genderRatio == 0: #locked male
        gender = 0
    else:
        gender = (pid & 0xff) < genderRatio # Less than gives Male, greater than gives Female
```

Step 4: Shiny

Depending on the wondercard file the PID will potentially get modified to satisify the shiny requirement. No prng calls are required by this.

```
if wondercard[0x37] == 0:
    pid = force_non_shiny(pid, tid, sid)
    shiny = False
elif wondercard[0x37] == 1:
    shiny = (pid >> 16) ^ (pid & 0xffff) ^ tid ^ sid < 8
elif wondercard[0x38] == 2:
    pid = force_shiny(pid, tid, sid)
    shiny = True
```

Step 5: Ability

Depending on the wondercard file the PID will potentially get modified to satisify the ability requirement. Although this happens after the shiny modifications, the ability modifications do not modify the PID enough to make a shiny into non-shiny or a non-shiny into shiny.

```
if wondercard[0x36] == 0: # Force ability 1
    pid &= ~0x10000
    ability = 1
elif wondercard[0x36] == 1: # Force ability 2
    pid |= 0x10000
    ability = 2
elif wondercard[0x36] == 2: # Force hidden ability
    pid &= ~0x10000
    ability = Hidden
elif wondercard[0x36] == 3: # Ability 1/2
    pid ^= 0x10000
    ability = ((pid >> 16) & 1) + 1
```

Step 5.5: Blank

1 blank frame gets consumed in between the Ability and Nature.

```
prng, _ = next(prng)
```

Step 6: Nature

If the nature is random we will generate one otherwise just copy over from the wondercard file.

```
if wondercard[0x34] == 0xff:
    prng, rand = next(prng, 25)
    nature = rand
else:
    nature = wondercard[0x34]
```