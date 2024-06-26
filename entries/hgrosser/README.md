# Hartmut Grosser

**1 billion row Challenge entry**

## Version
Version 1.51

## How to compile
The program was developed with FPC 3.2.2 and Lazarus 2.2.4

## How to start
```
Usage:   <path to input file> <bit-width for hash-list (14..28)>
Example: hgrosser measurements.txt 15
 - bit-width for hash-list: sets the size of the hash list, e.g. '16' => 65536 entries
```
There are no switches like `-i` etc, only values.

### Optimizing the 2nd command line parameter

In theory the program should run faster with greater bit-widths for the hash-list (because of less collisions), but on my own computer (8 GB RAM) in praxis a small value of 15 is the fastest way, allthough this causes many collisions.

Please (if possible) try all values from 14 to 24 (maybe in a for-loop). Thanks a lot.

## How the program works
The Program works with 1 thread.

To speed things up:

- the input file is read via procedure 'blockread' ...
- into an AnsiString, so that function 'PosEx' can be used to parse it
- to manage the city names, a self made hash-list is used
- temperatures are stored as integers (multiplied by 10)

## History

- Version 1.00: initial version
- Version 1.50: hash-list optimized, small improvements in parsing the file
- Version 1.51: small improvements in asm function
