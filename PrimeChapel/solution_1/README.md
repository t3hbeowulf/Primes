# Chapel implementations by GordonBGood

![Algorithm](https://img.shields.io/badge/Algorithm-base-green)
![Faithfulness](https://img.shields.io/badge/Faithful-yes-green)
![Parallelism](https://img.shields.io/badge/Parallel-no-green)
![Bit count](https://img.shields.io/badge/Bits-1-green)

## Run

### Run locally

You may have [to build the Chapel compiler yourself](https://chapel-lang.org/docs/usingchapel/QUICKSTART.html) on many systems; on Windows, it only runs under Cygwin (or WSL, of course). The following commands run in the same directory to which you have copied the "primes.chpl" source file should get you started once you have the Chapel compiler installed:

```
chpl --fast primes.chpl
./primes
```

### Docker

Once you have docker installed on your system, as per the usual minimal solution with just a minimal set of commands, with the "Dockerfile" also copied into the same directory as above:

```
docker build -t primes .
docker run --rm primes
```

The Dockerfile uses an official Debian image and then builds and installs Chapel on it.

## Output
```
GordonBGood_bittwiddle;7776;5.00044;1;algorithm=base,faithful=yes,bits=1
GordonBGood_unpeeled;9838;5.00047;1;algorithm=base,faithful=yes,bits=1
GordonBGood_unpeeled_block;10319;5.00005;1;algorithm=base,faithful=yes,bits=1
GordonBGood_unrolled;15865;5.0002;1;algorithm=base,faithful=yes,bits=1
GordonBGood_unrolled_hybrid;28818;5.00012;1;algorithm=base,faithful=yes,bits=1
```

## Benchmarks

Running locally on my Intel SkyLake i5-6500 at 3.6 GHz when single threaded, I get some astounding numbers:

```
GordonBGood_bittwiddle;7803;5.00046;1;algorithm=base,faithful=yes,bits=1
GordonBGood_unpeeled;9768;5.00028;1;algorithm=base,faithful=yes,bits=1
GordonBGood_unpeeled_block;10356;5.00035;1;algorithm=base,faithful=yes,bits=1
GordonBGood_unrolled;16259;5.00009;1;algorithm=base,faithful=yes,bits=1
GordonBGood_unrolled_hybrid;28948;5.00005;1;algorithm=base,faithful=yes,bits=1
```
Which matches the results when run with Docker on the same machine as follows:

```
                                                                Single-threaded                                                                
┌───────┬────────────────┬──────────┬─────────────────────────────┬────────┬──────────┬─────────┬───────────┬──────────┬──────┬───────────────┐
│ Index │ Implementation │ Solution │ Label                       │ Passes │ Duration │ Threads │ Algorithm │ Faithful │ Bits │ Passes/Second │
├───────┼────────────────┼──────────┼─────────────────────────────┼────────┼──────────┼─────────┼───────────┼──────────┼──────┼───────────────┤
│   1   │ chapel         │ 1        │ GordonBGood_unrolled_hybrid │ 29556  │ 5.00014  │    1    │   base    │   yes    │ 1    │  5911.03449   │
│   2   │ chapel         │ 1        │ GordonBGood_unrolled        │ 16499  │ 5.00002  │    1    │   base    │   yes    │ 1    │  3299.78680   │
│   3   │ chapel         │ 1        │ GordonBGood_unpeeled_block  │ 10499  │ 5.00003  │    1    │   base    │   yes    │ 1    │  2099.78740   │
│   4   │ chapel         │ 1        │ GordonBGood_unpeeled        │  9969  │ 5.00005  │    1    │   base    │   yes    │ 1    │  1993.78006   │
│   5   │ chapel         │ 1        │ GordonBGood_bittwiddle      │  7851  │ 5.00032  │    1    │   base    │   yes    │ 1    │  1570.09951   │
└───────┴────────────────┴──────────┴─────────────────────────────┴────────┴──────────┴─────────┴───────────┴──────────┴──────┴───────────────┘
```

## Notes

The techniques used are as follows:

1. The very basic standard odds-only packed-bit "bit-twiddling" sieving (1of2).
2. Loop "unpeeling" where the culling marking loops are simplified by recognizing that modulo math means every eighth culling/marking position is at the same bit index per byte with the byte indexes separated by the base prime value span so that are eight loops culling/marking with each using a different fixed byte mask.  This is faster because the simple non-"bit-twiddling" masking operations can be very simple, but meaning that there is more loop overhead (eight loops per base prime value instead of one) and there is still "cache thrashing" as each loop culls/marks across the entire sieve buffer which spans several CPU caches in the case of common CPU's.
3. Loop "unpeeling" by cache sized blocks, which reduces the "cache thrashing" by running the above eight loops across each block in turn, saving the next marking/culling index in an array for use on the next block.  As full page segmentation isn't allowed (determining all base prime values as a separate process), this reduces but can't eliminate "cache thrashing".
4. Extreme loop unrolling, which combines the eight loops above into a single loop running across the entire sieve buffer; since the eight loops are combined and as page segmentation is not allowed, there is no advantage (and an increased overhead) to culling by cache-sized blocks.  Since Chapel does not have computed goto's, this feature is emulated manually by creating an array containing "functions" that cull by each of the possible 64 culling patterns, which patterns depend on the modulo eight of the span (the base prime value) and the modulo eight of the starting bit index (eight times eight is 64 elements), of which not all are used for this program.  Since only odd base prime values are used, there are only four base prime modulo eight entries for span value, and since we are starting all of them at the index of the base prime squared, then each base prime modulo creates a constant start bit index modulo eight; thus, there are only four out of the 64 entries used for this program, but the rest are kept for completeness in other applications.
5. A hybrid approach where extreme loop unrolling as per the above is used for larger base prime values above a threshold and a dense algorithm for those base prime values below.  For this implementation the threshold is chosen so dense base primes are considered to be nineteen and below; larger values up to about 31 would likely make the code even slightly faster, but as Chapel does not have macros, a code generator must be used for the cases handling each odd value up to the threshold and the source could becomes rather long for only a small extra benefit.  Instead of addressing the sieve buffer array by byte indices as for extreme loop unrolling, the dense processing addresses the buffer by 64-bit words so that the maximum number of small-span composites are contained in each word.  There have been solutions that correctly haven't been accepted as `faithful to base` because in addressing by words they cleared all the dense composites in the word with a single instruction using a mask, but this implementation clears them bit by bit with individual operations in a manner similar to the technique used in the Rust "striped hybrid reset" technique but using larger words instead of bytes; thus this version should be accepted as `faithful to base`.  As well, as it is expected that we don't use any knowledge of base primes other than the only even prime of two, the implementation covers all of the odd value possibilities and thus includes the values of nine and fifteen even thought these are not prime and the code will never be used.

In implementing "Extreme Loop Unrolling", since Chapel does not have general-use first class functions (the current implementation can only access global and local (inside the scope of the function) variables, these are emulated using a class hierarchy, which classes can emulate closures by using the class fields to capture variables from the environment (not used here) and the default `this` method as the "apply" method of a Functor class.  Chapel's parameterized generic classes act as class/Functor templates for this application.  This class "template" is then instantiated once for each of the 64 "goto" table entries.  The use of this table is then structured to look like a call to a "setbit(array, start, limit, step)" function to make it clear that there is one operation per composite number culling/marking.

For the dense hybrid technique add-on, the same general form using an array of classes-representing-functions is used, but these can't be generated automatically from a single parameterized generic class/template as the code varies per step value and thus code generation had to be used.  In another language that had true macros and not just templates, the code could all be generated in just a few lines of code, possibly inline to the source.  For languages with this facility, the threshold of dense base prime values could be extended to include 31 for an extra at least perceptible speed increase.
