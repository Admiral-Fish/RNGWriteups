# Slot Advances
Up to 6 Pokemon can be transfered at a time from Dream Radar. Each one impacts the next one's generation. Each slot will have a type, gender, and gender ratio. Each slot will advance 13 IV frames and 5 PID frames if the slot is not genderless. If the type is a Genie legend, then an additional 13 IV frames and 5 PID frames will be advanced.

```
iv_advances = 0
pid_advances = 0

for i in range(len(slots)):
    slot = slots[i]

    if slot.type == Genie:
        iv_advances += 13
        pid_advances += 5

    # Only advance for slot if it isn't the last one
    if i != len(slots) - 1:
        iv_advances += 13
        pid_advances += 4 if slot.gender_ratio == Genderless else 5

```

# Generation
Dream Radar uses two different RNGs to for generation. The first is a LCRNG, used for non-IVs, which I will describe below. The other is Mersenne Twister, used for IVs, which you can read about [here](http://www.math.sci.hiroshima-u.ac.jp/~m-mat/MT/emt.html).

```
def next(prng):
    prng = (prng * 0x5d588b656c078965 + 0x269ec3) & 0xffffffffffffffff
    return prng

def force_non_shiny(pid, tid, sid):
    if (pid >> 16) ^ (pid & 0xffff) ^ tid ^ sid < 8:
        pid ^= 0x100000000
    return pid

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

Step 0: Advances

The advances that were desribed above are first advanced.

```
for i in range(iv_advances):
    iv_prng = iv_next()

for i in range(pid_advances):
    prng = next(prng)
```

Step 1: IVs

Each IV is generated from a call to the Mersenne Twister only keeping the top 5 bits.

```
for i in range(6):
    ivs[i] = iv_next() >> 27
```

Step 1.5: Blank

One blank frame is consumed

```
prng = next(prng)
```

Step 2: PID

The top 32 bits of the next prng call is given to the PID

```
prng = next(prng)
pid = prng >> 32
```

Step 3: Gender

Depending on the slot, the PID is potentially modified. It is important to note that even though the Gen 4 Legends are genderless, they are treated as 100% Male.

```
if slot.type == Genie or slot.type == Gen4Legend: # 100% Male
    prng = next(prng)
    pid = force_gender(pid, prng >> 32, 0, 0)
    gender = Male
elif slot.gender == Male or slot.gender == Female:
    prng = next(prng)
    pid = force_gender(pid, prng >> 32, slot.gender, slot.gender_ratio)
    gender = slot.gender
else:
    gender = Genderless
```

Step 4: Ability

While the Pokemon of Dream Radar are forced to their Hidden Ability, the typical Gen 5 Ability flip does happen.

```
pid ^= 0x10000
ability = Hidden
```

Step 5: Shiny

Pokemon from Dream Radar can never be shiny.

```
pid = force_non_shiny(pid, tid, sid)
shiny = False
```

Step 5.5: Blanks

Two blank frames are consumed

```
for i in range(2):
    prng = next(prng)
```

Step 6: Nature

One final call is used to generate the nature

```
prng = next(prng)
nature = ((prng >> 32) * 25) >> 32
```
