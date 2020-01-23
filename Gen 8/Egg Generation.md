# Background Info

This writeup will be split into two sections. One section will be about how the generation of the seed occurs and why this doesn't allow for any type of RNG abuse(retail or emu). The second section will detail how an egg is generated from the egg seed.

# Seed generation
The RNG for egg generation is [xoroshiro128+](http://prng.di.unimi.it/xoroshiro128plus.c). The very short details of this RNG is that it keeps an internal state of 2 64bit numbers. The 2nd number is a constant number of 0x82A2B175229D6A5B. The 1st number is a little more interesting.

The generation of the 1st number can be found in sub_710134AA70. This is the function that checks if the daycare lady should be holding an egg. The part that makes it so this value can't be RNGed is at .text:710134AC30. This is where is makes a call to nn::crypto::GenerateCryptographicallyRandomBytes. For those that don't understand what this means, basically the 1st value is created by a cryptographically secure generator that is essentially impossible to manipulate or predict.

However, it is still possible to edit the 1st number afterwards for testing purposes and prove the egg generation behaves as detailed below.

# Egg Generation

Egg generation occurs at sub_7100787540. For the purposes of sticking to just RNG generation, I will not cover parts that do not directly use an RNG call in detail. This includes things like baby species, egg moves, etc.

An important note is that the call to the RNG is a slightly modified version of xoroshiro128+. A TLDR is that it will modulo the result by the next power of two until the result is less then the desired max value. It is as follows:

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

It starts by reordering the parents(sub_71007872d0, this is right before the call to the egg generation). The following table lists how each parent is determined(listed parent1 then parent2):

| Breeding pairs   | Parent order     |
|------------------|------------------|
| Female/Male      | Male/Female      |
| Female/Ditto     | Ditto/Female     |
| Male/Ditto       | Male/Ditto       |
| Genderless/Ditto | Genderless/Ditto |

Step 1. Species

It does some kind of species determination from the parents that I didn't dig into very deeply. This most likely handles choosing the lowest evolution form.

Step 2. Nidoran

This is done if the breeding involves Nidoran. If next(2) is 1 then the egg will be Nidoran(F) otherwise it will be Nidoran(M).

```
rand = next(2)
if rand == 1:
    species = 29
else:
    species = 32
```

Step 3. Illumise/Volbeat

This is done if the breeding involves Illumise/Volbeat. If next(2) is 1 then the egg will be Illumise otherwise it will be Volbeat.

```
rand = next(2)
if rand == 1:
    species = 314
else:
    species = 313
```

Step 4. Manaphy

If the breeding involves Manaphy then the egg is set to Phione.

Step 5. Indeedee

This is done if the breeding invovles Indeedee. It sets the alt form value to be next(2).

```
altform = next(2)
```

Step 6. Baby pokemon

If the conditons of a pokemon having a baby form and one of the parents holding the proper incense item then the egg will be the baby pokemon.

Step 7. Gender

If a random gender is required (not genderless or locked male/female) then it will do next(252)+1 and compare this value to the gender ratio to determine female/male.

```
if genderRatio == 255: # locked genderless
    gender = 2
elif genderRatio == 254: # locked female
    gender = 1
elif genderRatio == 0: #locked male
    gender = 0
else:
    gender = next(252) + 1 < genderRatio # Less than gives Female, greater than gives Male
```

Step 8. Nature

The nature is set to next(25). Following this is the check for everstone. If only one parent has everstone then their nature will be passed on. If both parents have everstone then if next(2) is 1 parent 2 will pass on their nature otherwise parent 1 will pass on their nature.

```
nature = next(25)

if parent1.item == Everstone and parent2.item == Everstone:
    rand = next(2)
    if rand == 1:
        nature = parent2.nature
    else:
        nature = parent1.nature
elif parent1.item == Everstone:
    nature = parent1.nature
elif parent2.item == Everstone:
    nature = parent2.nature
```

Step 9. Ability

If parent2 is ditto it will use parent1 otherwise parent2. The ability of the parent just described is used.

If the ability is 0(first ability) then if next(100) is less than 80 the egg will have ability0 otherwise ability1.

If the ability is 1(second ability) then if next(100) is less than 20 the egg will have ability0 otherwise ability1.

If the ability is 2(hidden ability) then if next(100) is less than 20 the egg will have ability0, less than 40 the egg will have ability1 otherwise ability2.

```
rand = next(100)
if parentAbility == 0:
    ability = rand < 80
elif parentAbility == 1:
    ability = rand < 20
elif parentAbility == 2:
    ability = 0 if rand < 20 else 1 if rand < 40 else 2
```

Step 10. Friendship

A friendship value is assigned. I know in past gens the friendship value has also doubled for the egg step counter so I'm just gonna assume it is the same situation here.

Step 11. IV

I am going to skip power items since destiny knot is a much better item(but note power items is handled in this step). There will be 3 or 5 inherited IVs depending on destiny knot. The game determines which parents will be passing on IVs using next(6) to determine which stat and next(2) to determine which parent. Next the egg gets IVs from 6 next(32) calls then the necessary inherited IVs are copied over.

```
if parent1.item == DestinyKnot or parent2.item == DestinyKnot:
    inheritCount = 5:
else:
    inheritCount = 3

inheritIVs = [-1]*6
inherit = 0
while inherit < inheritCount:
    stat = next(6)
    if inheritIVs[stat] == -1:
        inheritIVs[stat] = 2 if next(2) == 1 else 1
        inherit += 1

ivs = [next(32) for i in range(6)]
for i in range(6):
    if inheritIVs[i] == 1:
        ivs[i] = parent1.ivs[i]
    elif inheritIVs[i] == 2:
        ivs[i] = parent2.ivs[i]
```

Step 12. EC

The encryption constant is set a value of next(0xFFFFFFFF).

```
ec = next(0xFFFFFFFF)
```

Step 13. PID

It checks for masuada method and shiny charm. Masuada method gives 6 rolls and shiny charm gives 2 rolls. Without either of these the PID is generated by something else(unknown to me at this time). It will generate PID values from next(0xFFFFFFFF) until it has hit the roll limit or it finds a shiny PID.

```
rerolls = 0

if masuada_method:
    rerolls += 6

if shiny_charm:
    rerolls += 2

for i in range(rerolls):
    pid = next(0xFFFFFFFF)

    if isShiny(pid):
        break
```

Step 14. Light Orb

If the egg is Pichu and one of the parents is holding a Light Orb it gives the egg move Volt Tackle.

Step 15. Ball

If parent2 is ditto it will pass down the ball of parent1.

If both parents are the same species then if next(100) + 1 is less than 51 it will pass down the ball of parent2 otherwise parent1.

If neither of the previous conditions are true then it will pass down the ball of parent2.

There is one last check to make sure the ball being passed down isn't Master Ball or Cherish Ball. If this happens it will force Poke Ball.

```
if parent2.species == Ditto:
    ball = parent1.ball
else:
    if parent1.species != parent2.species:
        ball = parent2.ball
    else:
        rand = next(100) + 1
        if rand < 51:
            ball = parent2.ball
        else:
            ball = parent1.ball

if ball == MasterBall or ball == CherishBall:
    ball = PokeBall
```
