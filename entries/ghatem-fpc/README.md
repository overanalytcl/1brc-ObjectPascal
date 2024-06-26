# Georges Hatem

## Requirements
 - mORMot2 library
 - 64-bit compilation
 
## Hardware + Environment
host: 
 - Dell XPS 15 (9560, 2017)
 - OS: ArchLinux
 - CPU: Intel i7 7700 HQ (4 Cores, 8 Threads @2.80-3.80 GHz, Kaby Lake)
 - 32 GB RAM DDR4 (2400 MHz)
 - 1 TB SSD NVME
 
VM (VirtualBox):
 - OS: Windows
 - CPU count: 4 out of 8 (threads, probably)
 - 20 GB RAM
 
note about the hash:
run with DEBUG compiler directive to write from stream directly to file, otherwise the hash will not match.

## Baseline
the initial implementation (the Delphi baseline found in /baseline) aimed to get a correct output, regardless of performance: 
"Make it work, then make it work better".
It turns out even the baseline caused some trouble, namely the `Ceil` implementation was yielding different results between FPC and Delphi (and different results between Delphi Win32/Win64).
After the input of several peers including gcarreno, abouchez and paweld (thanks!), this last detail was ironed out, and the baseline yielded a matching hash.

## Single-Threaded Attempt (2024-04-03)
 
in this first attempt, the implementation is broken down into 3 major steps:
 1. read the input file
 2. process it
 3. output to file/stdout
 
a key point: 
 - the reading / writing (steps 1 an 3) will be done on the main thread.
 - the processing (step 2) is where a future submission will attempt to parallelize the work done.
 
### 1. Read The Input File

#### v1. File Stream
In the baseline implementation, a file stream is used to read line by line, which is far from optimal.

#### v2. Memory-Mapped File
An improvement was to read the whole file into memory in one shot, using memory mapping. 
In this implementation, I use `GetFileSize` and `CreateFileMapping`, procedure found online (need to find URL).  
First thing to note is, the usable memory by a Win32 process is limited to ~1.5-2 GB of RAM. Exceeding this limit yields an out-of-memory exception, so we must compile for Win64.
Some issues with this implementation (see unit FileReader.pas): 
 - `GetFileSize` was returning a size of ~3.9B, while we know the real input is ~16B.
 - `CreateFileMapping` was taking 2.5 seconds to read the file into a `TArray<Utf8Char>` the first time, and subsequent reads were down to 1.7 seconds.  But this is for only a quarter of the input size.
 - using `GetFileSizeEx` instead, we now get the real file size, ~16B
 - however, `CreateFileMapping` accepts as parameters `Cardinal` types, so a value of ~16B (Int64) would yield a `range check` error.
 - if we wanted to move forward with this implementation, we would need to call `CreateFileMapping` in 4 or 5 batches, which would take 1.7 x 5 ~= 8.5 seconds just to read the data.
 - attempt aborted, see v3.
 
#### v3. Memory-Mapped File, Provided by `mORMot2`
A v3 attempt at reading the file was using a ready-made implementation of file memory-mapping, provided by synopse/mORMot, big thanks @abouchez!
The function returns a pAnsiChar and the size as Int64 of the mapped data.  Performance-wise, it all happens in under 0.1 seconds, but now we must delve into pointers.


### 2. Process the File

Well, at a glance this is straightforward:
 - look for new-line chars to delimit each line, split it to extract StationName / Temperature.
 - Decode the temperature into a numerical value (double, or integer x 10)
 - accumulate the information into a dictionary of StationName -> Record of data
 
A few optimizations done here, to the best of my knowledge:
  
#### For Each line, Iterate Backwards
`Length(StationName) > Length(Temperature)`, so for each line, better look for the `;` starting from the end.
Given the below input:
```
Rock Hill;-54.3
            ^^^
```
the 3 last characters will be mandatorily present, so we can skip them while iterating.
I tried unrolling the loop over the last 2-3 characters that must be checked, but measuring it, it showed to be slower, don't know why.

#### Extract the Strings Using `SetString`
manual string concatenation and splitting proved to be very slow.
Using `SetString` yielded a noticeable improvement. Remaining in the realm of pointers would probably be much faster, but I haven't ventured there (yet, maybe if time is available).

#### Decode Temperature Into Numerical Value
First attempt was to use `StrToFloat`, which was pretty catastrophic. Using `Val` was still slow, but definitely a step-up. `Val` with SmallInt proved to be faster than with Double, even though there's extra operations to be done.
So now we need to get rid of the `.` character.

Again, string functions being very slow, replicating the last character at length-1, and then reduce the length of the AnsiString using `SetLength` yielded faster results. I tried doing so with pAnsiChar but got some range-check errors.  Might get back to it later on.

Finally, assuming temperatures in the range `-100` and `+100`, with 1 decimal figure, there should be 2000 different temperatures possible.
Instead of decoding the same temperature values using `Val`, do it once, store it in a TDictionary (TemperatureString -> TemperatureSmallInt). There were I believe 1998 different temperature values, so we only call `Val` 1998 times instead of 1 billion times.  Over an input size of 100M, the gain was 4-5 seconds (total 28s -> 23s)

#### Accumulate Data Into a Dictionary of Records
 - the records are packed, with minimal size
 - the dictionary maps StationName -> Pointer to record, to avoid passing around full records
 - records are pre-allocated in an array of 45,000, instead of allocating them on-the-fly.
 - when a station is not found in the dictionary, we point to the next element in the records-array.
 - with an input of size 100M, this accumulation step takes a considerable amount of time (9 seconds out of 23 total). I haven't identified yet if it is the `dict.Add` that takes time, the `dict.TryGetValue`, or just generally the dictionary hash collisions.  Even though the dictionary is pre-allocated with a capacity of 45,000, but that did not seem to improve much. I also tried the dictionary implementation of Spring4D, but also no improvements.
 
### 3. Output the Results
Since I started using pointers (pAnsiChar), getting a matching hash was a bit of a pickle:
Some Unicode characters were messed up in their display, or messed up in their ordering.
Eventually, the ordering issue was resolved by using `AnsiStrings.CompareStr` instead of `SysUtils.CompareStr`. This step will clearly remain single-threaded, but takes 0.15 seconds for all 45,000 stations, so it is not a big deal.

### Conclusion

If there are any improvements to be done in this single-threaded version, they would be in the following order, from most impactful to least impactful (performance numbers over an input of 100M, not 1B):
 - try improve storing / retrieval from the dictionary of station data, cost: 9 sec / 23 sec
 - try improve the extraction of string data, maybe using pointers (6.5 sec / 23 sec)
 - try improve the type conversion, though not sure how at this point (4.5 sec / 23 sec)
 - somehow, incrementing an integer 1B times is taking 1.2 seconds, while incrementing the main input index (16B times) is only taking 0.5 second.  It's just 1.2 seconds, but I'm not understanding why it should behave that way. 


## Single-Threaded Attempt (2024-04-09)

Turns out mORMot2 does not compile under Delphi for Linux x64 due to poor Delphi support for Linux.
This implementation was migrated as-is with Lazarus (FPC), which is much slower than Delphi on Windows x64.

**Timing as per gcarreno: 176 seconds, single-threaded**

Since then, a few improvements:

### better temperature conversion to SmallInt

In the previous implementation, we identified `Val` and `StrToFloat` as a source of slowness.  @abouchez suggested that the numerical value could be obtained without those.
Indeed, by using Ord of the needed characters, offsetting by the ASCII value of '0', and multiplying by the right power of 10, the temperature can be extracted as integer at a fraction of the cost.

### better usage of dictionary data-structure:

Instead of extracting the station name as a string 1B times, and use it as a dictionary key, instead use the FPC hashing function to reduce the string extraction to ~45k times (once per station), and otherwise use the pointer + length (station_name) to compute the hash.
This requires us to migrate from TFPHashList to the generic TDictionary. Even though TDictionary is slower than TFPHashList, the overall improvements yielded a significant performance gain.
Using TDictionary, the code is more similar to my Delphi version, in case we get to run those tests on a Windows PC.

* expected timing: ~60 seconds, single-threaded*
** ACTUAL TIMING: 58 seconds as per gcarreno **


## Multi-Threaded Attempt (2024-04-10)

In a first attempt, using 2 threads, I evaluated how harmful were my "shared-memory" variables, namely:
 - the pre-allocated records array
 - its counter
 - the dictionary of aggregated data
 - the stringlist of station names

Using a Critical-Section, even on just the stringlist, the performance drops quite a bit.

### minimize shared-memory

I got rid of the pre-allocated records array and its counter, back to on-the-fly allocation of records
Got rid of the station names stringlist, because merging those lists at the end was very slow: instead, store the station name directly in the record.
Replaced the singular dictionary with an array of dictionaries, 1-per-thread.

### redundant work

All of this causes some redundant work to be made by various threads:
 - one record is allocated per-station-per-thread
 - since we store the station name in the record, we convert it to string (using `SetString`) once per-station-per-thread

### parallelism

Given N bytes and K threads, a basic attempt is to distribute a range `N / K` of data per thread.
A thread may get its start/end boundaries in the middle of a line: we ensure that each line is processed exactly once.

Again, just to evaluate how things will go, we wait for all threads to complete before proceeding.

### merging

Given 2 dictionaries L and R, we merge them into L. This merge is applied K-1 times, and dictionaries[0] will contain the aggregated result.
We could later consider merging every 2 threads that have completed their work.

### expectations

parallelization on my environment (see above) performed quite poorly, considering there is no shared-mem to protect against concurrent access:
4 threads were barely getting 60% improvement over 1 thread.

is it due to:
 - VM virtualization?
 - too many dictionaries were causing too many cache misses?
 - work-load distributed unevenly?

Better wait and see the results on the real environment, before judging.

### results

** ACTUAL TIMING: 6.042 seconds as per gcarreno **

Due to the unexpectedly slow performance on Craig Chapman's powerful computer, and since the results above intrigued me, I have ported my FPC code onto Delphi to be able to compare the output of both compilers on Windows x64.
Hopefully, it will help identify is the issue stems from Windows x64 or FPC, in multi-threaded implementations.