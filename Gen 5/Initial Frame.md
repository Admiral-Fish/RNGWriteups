Whenever the game is booted in BW/BW2 a number of frames are advanced depending on the initial seed and a pre-defined table the game uses. I will quickly define how the RNG is used by this table:

```
def next(seed):
    return (seed * 0x5d588b656c078965 + 0x269ec3) & 0xffffffffffffffff
```

# Advancement table
The game uses a much more complex table then what I will describe below, but are functionally equivalent. The original is described [here](https://www.smogon.com/forums/threads/past-gen-rng-research.61090/page-30#post-3649544).

```
table = [[50, 100, 100, 100], [50, 50, 100, 100], [30, 50, 100, 100],
         [25, 30, 50, 100], [20, 25, 33, 50]]
```

The game uses this modified table and the RNG described above to advance the initial frames. For each array in the table, a RNG call is compared to the non-100s to see if the loop over the array should break early.

```
def advance_table(prng)
    count = 0

    for i in range(5):
        for j in range(4):
            if table[i][j] == 100:
                break

            count += 1
            prng = next(prng)
            rand = ((prng >> 32) * 101) >> 32
            if rand <= table[i][j]:
                break

    return prng, count
```

# Black/White
In Black/White this table gets advanced over by a set number and that determines the initial frame. In normal cases this number will be 5, but when a new save is being started (typically for TID/SID RNG) this number will be 2 if a save file already exists otherwise it will be 3.

```
def initial_frame_bw(prng, rounds):
    count = 1

    for i in range(rounds):
        prng, num = advance_table(prng)
        count += num

    return count
```

# Black/White 2
For the case where a new save is not starting, Black/White 2 has some additional advances over Black/White. After the first round there are some additional advances that depend on whether or not memory link has been used. After the final round 3 prng states are grabbed and checking for duplicates up to 100 times.

```
def initial_frame_bw2(prng, memory):
    count = 1
    
    for i in range(5):
        prng, num = advance_table(prng):
        count += num

        if i == 0:
            for j in range(2 if memory else 3):
                count += 1
                prng = next(prng)

    for i in range(100):
        count += 3

        prng = next(prng)
        rand1 = ((prng >> 32) * 15) >> 32

        prng = next(prng)
        rand2 = ((prng >> 32) * 15) >> 32

        prng = next(prng)
        rand3 = ((prng >> 32) * 15) >> 32

        if rand1 != rand2 and rand1 != rand3 and rand2 != rand3:
            break

    return count
```

Much like Black/White, for starting a new save the number of rounds will be 2 if there is an existing save file otherwise it will be 3.

```
def initial_frame_bw2_id(prng, rounds):
    count = 1

    for i in range(rounds):
        prng, num = advance_table(prng)
        count += num

    return count
```