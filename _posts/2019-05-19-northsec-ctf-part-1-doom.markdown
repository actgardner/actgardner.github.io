---
layout: post
title:  "NorthSec CTF 2019 Part 1: DOOM"
date:   2019-05-19 00:00:00
categories: securit ctf northsec 2019 
---

Last weekend I was in Montreal for the [NorthSec](https://nsec.io) conference and Capture The Flag (CTF). 
Briefly, the CTF is a 48-hour event where teams compete to solve security problems in order to earn points.  
Each problem involves finding or otherwise earning a flag, some unique, hard-to-guess text like `FLAG_{blah blah}`, which you then submit to prove you've solved the problem.
Flags can be in plain sight around the event site, they can be rewards for completing physical challenges, hidden in audio or image files, or they can be in a file or in memory on some system you need to crack.

One of the interesting problems I (mostly) solved this year was to play DOOM. 
The problem description offered a ZIP file and four goals:

- find the secret room
- find the red key
- find the blue key
- finsh the level

The "secret room" key was available in-game but the other goals would need to be demonstrated to the organizers on a clean system - you couldn't just hack your copy of the game to add keys.

The ZIP file contained a Windows build of (ZDoom)[https://zdoom.org/downloads] with a bunch of files, including two called `zoombies.wad` and `textures.wad` that stood out. 
ZDoom itself is a game engine, and [WAD (Where's All the Data) files](https://zdoom.org/wiki/WAD) contain the data for the DOOM engine - levels, menu configs, etc. This ZIP was a custom game that ran on the DOOM engine!

To start I wanted to play the game. The ZIP only had a Windows binary, so I went out to find an OS X build of ZDoom. 
The newest ZDoom (v4.1.2) segfaulted trying to play, and the Windows binary had a date from August 2018, so I found an appropriately-aged OS X binary (v3.5.1) that worked better. 

Once the game would run I was able to play through and explore. The first area had the following features:

- some demons
- a button to open a door
- 2 locked doors
- a door that opened to a small room
- 2 more buttons

Pressing the buttons would print a message like `the current key is <x><y>`, and increment one digit or the other up to 10 before wrapping around. Flippantly I typed in "42" and one of the locked doors opened! One flag down.

The new area that was unlocked had some more demons, a locked door and _6_ buttons. Pushing the buttons produced a similar `current key` message with 6 digits that were letters and numbers. So the challenge was to find this longer, base36 key.
 
Looking at the contents of the WAD with `xxd` there were lots parts that looked like strings with some inary framing, so I decided to see if I could unpack the WAD into parts I could edit. 
WADs are basically a header, a concatenation of many data "lumps", and a directory of names and pointers to each lump. 
I wrote a small Python script according to the specification from the ZDoom wiki and got a folder full of the different "lumps" of data.

Poking at some of the lumps (ewww) and reading the docs it seemed like `BEHAVIOUR` was a good place to look.
This file holds all the scripts that drive logic in a map, like opening doors and displaying messages, in [ACS bytecode](http://maniacsvault.net/manual.pdf). 
Foolishly assuming this would be a relatively simple binary format like the WAD, I started working on a disassembler to print the instructions for each method. 

This was a huge waste of time. 
The documentation is not very precise, the instruction set is huge, and not only can instructions take different numbers of operands but those operands can be either 1 or 4 bytes and it's not clear which ones are which.
I spent way too much time trying to parse the `ACS0` format before re-reading the spec and realizing I had an `ACSe` file, which starts with a valid-but-bogus `ACS0` file to trick old engines and stupid developers. 
After a few hours of work I was able to read the directory of the ACS file and extract the string table, method names and methods but it was impossible to interpret any of the methods.

A lot this challenge was spent Googling fruitlessly. 
The ecosystem of DOOM-related tools is vast and confusing. Even trying to find a good build of ZDoom for OS X had been a challenge, and based on my experience looking for ACS documentation I assumed no good disassembler existed.
The few projects in my search results were a) Windows binaries b) focused on compiling ACS scripts into byte code, not disassembly.
I tried reading the source of some ACS compilers on Github but nothing was coming together. 
As a last-ditch effort I downloaded [ListACS](https://github.com/alexey-lysiuk/ListACS) from Github, which had a measly two stars and very little activity.

ListACS was just what I needed! 
It printed the method and script bodies with the mnemonics and arguments for every instruction. 
Combined with the strings table and the method names it was possible to deduce which methods were used to print the text on the screen, to check the value of the key, etc.
I've included the disassembly of the key-checking routine, with some comments:

``` 
...

int map2[10] = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0}; 		// Scratch space for the key
int map3[10] = {2, 3, 5, 7, 13, 17, 19, 23, 31, 27}; 	// The list of factors to test

...

       // This function is called 'validateserverkey'
       399: function 4 (0, 11, 0) -> 1
         399: CALL 6 			// Convert the current key to an int and put it on top of the stack
         401: ASSIGNSCRIPTVAR 0         // Put the current base10 int key on the stack and also in SV0
         403: PUSHSCRIPTVAR 0		// Also store the current key in SV1
         405: ASSIGNSCRIPTVAR 1		
         407: PUSHBYTE 1		// SV2 is a redundant iteration variable
         409: ASSIGNSCRIPTVAR 2         
         411: PUSHBYTE 10		// SV3 = 10 (length of the key digit array)
         413: ASSIGNSCRIPTVAR 3         // If SV1 > 0 continue
         415: PUSHSCRIPTVAR 1
         417: PUSHBYTE 0		 
         419: GT			// This loop populates MA2 with the current key from most-significant bit (MV2[0]) to least (MV2[9])
         420: IFNOTGOTO 461             // Exit the loop
         425: PUSHSCRIPTVAR 1		// SV4 = SV1 / 10
         427: PUSHSCRIPTVAR 3
         429: DIVIDE
         430: ASSIGNSCRIPTVAR 4		
         432: PUSHSCRIPTVAR 1		// SV5 = SV1 % 10
         434: PUSHSCRIPTVAR 3
         436: MODULUS
         437: ASSIGNSCRIPTVAR 5		
         439: PUSHSCRIPTVAR 4		// SV1 = SV4
         441: ASSIGNSCRIPTVAR 1		
         443: PUSHBYTE 10		// Put the current value of SV5 in MA2[10-SV2]
         445: PUSHSCRIPTVAR 2		
         447: SUBTRACT
         448: PUSHSCRIPTVAR 5
         450: ASSIGNMAPARRAY 2		
         452: PUSHBYTE 1		// Increment SV2, our duplicate variable
         454: ADDSCRIPTVAR 2
         456: GOTO 415			// Repeat the loop to fill in the next digit of MV[2]
         461: PUSHBYTE 1		// SV6 = 1
         463: ASSIGNSCRIPTVAR 6
         465: PUSHBYTE 10		// SV7 = 10
         467: ASSIGNSCRIPTVAR 7
         469: PUSHBYTE 0		// SV8 = 0
         471: ASSIGNSCRIPTVAR 8		
         473: PUSHSCRIPTVAR 8		// IF SV8 < SV7 GOTO 495
         475: PUSHSCRIPTVAR 7
         477: LT
         478: IFGOTO 495
         483: GOTO 557
         488: INCSCRIPTVAR 8
         490: GOTO 473
         495: PUSHSCRIPTVAR 8		// Push MV2[SV8] on the stack
         497: PUSHMAPARRAY 2
         499: PUSHSCRIPTVAR 8		// Push MV2[(SV8 + 1) % SV7] on the stack
         501: PUSHBYTE 1
         503: ADD
         504: PUSHSCRIPTVAR 7
         506: MODULUS
         507: PUSHMAPARRAY 2
         509: PUSHSCRIPTVAR 8		// Push MV2[(SV8 + 2) % SV7] on the stack
         511: PUSHBYTE 2
         513: ADD
         514: PUSHSCRIPTVAR 7
         516: MODULUS
         517: PUSHMAPARRAY 2
         519: CALL 5			// SV9 = F5() (consumes the top 3 values from the stack)
         521: ASSIGNSCRIPTVAR 9	
         523: PUSHSCRIPTVAR 9		// SV10 = (SV9 % MA3[SV8]) 
         525: PUSHSCRIPTVAR 8
         527: PUSHMAPARRAY 3
         529: MODULUS
         530: ASSIGNSCRIPTVAR 10
         532: PUSHSCRIPTVAR 10		// if SV10 == 0 and SV9 != 0 goto 552
         534: PUSHBYTE 0
         536: NE
         537: PUSHSCRIPTVAR 9
         539: PUSHBYTE 0
         541: EQ
         542: ORLOGICAL
         543: IFNOTGOTO 552
         548: PUSHBYTE 0		// SV6 = 0 (this means we failed)
         550: ASSIGNSCRIPTVAR 6
         552: GOTO 488			// Jump to the beginning of the loop
         557: PUSHSCRIPTVAR 6
         559: IFNOTGOTO 577		// If SV6 > 0 jump to 577 (we failed)
         564: PUSH2BYTES 107, 1		// Call OpenDoor(107, 1) built-in special
         567: LSPEC2 11
         569: BEGINPRINT		// Print STR8
         570: PUSHBYTE 8
         572: PRINTSTRING
         573: ENDPRINT
         574: PUSHBYTE 1
         576: RETURNVAL
         577: PUSHBYTE 0
         579: RETURNVAL
...

    // Compute the decimal value of three digits from the top of the stack
    580: function 5 (3, 1, 0) -> 1
         580: PUSHBYTE 0	// SV3 = 0
         582: ASSIGNSCRIPTVAR 3
         584: PUSHSCRIPTVAR 0	// SV3 += SV0 * 100
         586: PUSHBYTE 100
         588: MULTIPLY
         589: ADDSCRIPTVAR 3
         591: PUSHSCRIPTVAR 1 	// SV3 ++ SV1 * 10
         593: PUSHBYTE 10
         595: MULTIPLY
         596: ADDSCRIPTVAR 3
         598: PUSHSCRIPTVAR 2	// SV3 += SV2
         600: ADDSCRIPTVAR 3
         602: PUSHSCRIPTVAR 3
         604: RETURNVAL		// return SV3
```

For brevity I've left out function 6, which converts the 6-digit base36 key into a decimal number.

There's only a few elements of the ACS VM required to understand this:

- the VM is stack-based 
- `SCRIPTVAR` (`SV` in the comments) are scalar variables scoped to a script
- `MAPARRAY` (`MA` in the comments) are arrays scoped to a map

`MAPARRAY` is a quirk of the ACS VM - reads and writes pop a value from the stack for the index within the array, in addition to taking an operand for the maparray to access. 
For example, `PUSH 1; PUSHMAPARRAY 2;` pushes the element at index 1 from maparray 2 onto the stack.

So this code:
- puts the decimal digits of the key in `MA2`
- iterates over the factors in `MA3`
- tests whether `MA2[i]*100 + MA2[(i+1)%10]*10 + MA2[(i+2)%10]` is divisible by `MA3[i]` and is not zero

So our goal is to produce a 6-digit base64 number which satisfies all ten of these constraints when converted to decimal digits:

- `(value[0] * 100 + value[1] * 10 + value[2]) % 2 == 0`
- `(value[1] * 100 + value[2] * 10 + value[3]) % 3 == 0`
- ...
- `(value[9] * 100 + value[0] * 10 + value[1]) % 27 == 0`

We could brute-force the 308 million possible numbers, but there's a more elegant solution:

```
factors = [2, 3, 5, 7, 13, 17, 19, 23, 31, 27]

def test_solution(key):
    for i in range(9, -1, -1):
        product = key[i] * 100 + key[(i+1)%10]*10 + key[(i+2)%10]
        if product % factors[i] != 0 or product == 0:
            return False
    return True

# For our initial set of candidates, try every multiple of 27 that's less than 1000
candidates = [[(x%100)/10, (x%10), 0, 0, 0, 0, 0, 0, 0, x/100] for x in range(0, 999, 27)]

# For the remaining digits, test every possible addition to the current candidates
for x in range(8, 1, -1):
    new_candidates = []
    for c in candidates:
        for i in range(10):
            t = list(c)
            t[x] = i

            # We know this candidate is valid for every digit after x, so only test x
            product = t[x] * 100 + t[(x+1)%10]*10 + t[(x+2)%10]
            if product > 0 and product % factors[x] == 0:
                print("Position {} candidate {}".format(x, t))
                new_candidates += [t]
    candidates = new_candidates

# Do a final pass because we never confirm the first 2 factors
for c in candidates:
    if test_solution(c):
        print("Solution: {}".format(c))
```

Because there's very few multiples of 27 that are less than 1000, we can seed the last digit and the first two digits with all the possible options.
After that we can try every possible digit in the ninth position and check whether the last three digits are a multiple of 31.
Repeating this process working from right to left we are able to prune the set of candidates for each position until we've filled in all the remaining positions. All that remains is to check the first 2 factors (2 and 3), which yields the single valid solution. 

Playing through the level on my laptop I was able to confirm that this number, base36-encoded, made the door open. I was able to get a key and access the exit of the level! Triumphantly I approached the organizers, and they pointed me to a back room with a square marked out on the floor in painter's tape... the clean system you had to demo on was an Oculus Rift. 

The DOOM build was a little awkward on Oculus - you aimed in the Z axis by moving the controllers, but you rotated your field of view with one of the joysticks, which produced a lot of nausea. Through a combination of awkwardly runnning past demons and shotgun blasts I was able to make it to the switches, punch in the code, and complete the level. Overall this was worth 4 points - one for the 42 answer, and 3 for completing the level using the 6-digit code.

Overall the CTF was a great time, and I'm grateful to the organizers and the Shopify team for including me despite my complete lack of experience or qualifications! I hope to post a few more write-ups about other interesting problems this week.
