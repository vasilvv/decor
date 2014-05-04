Decor
=======

Decor is an experimental domain-specific programming language for writing
cryptographic applications sensitive to timing attacks.  The premise of the
language is that asking humans to write constant-time code is not really the
best plan to achieve security.  There are many types of cryptographic attacks,
and while some of them (protocol design flaws, defects in cipher design) are
impossible to address on the intstrumentation level, there is no reason why
timing attacks are not one of them.

Premise
--------
Consider the following code (courtesy of http://github.com/agl/ctgrind):
```c
char check16_bad(unsigned char *a, unsigned char *b) {
  unsigned i;
  for (i = 0; i < 16; i++) {
    if (a[i] != b[i])
      return 0;
  }

  return 1;
}
```

The problem here is that attacker can send different invalid MACs, measure the
time it takes for the function above to reject that and by figuring out the
valid MAC byte-by-byte, reduce the time needed to brute-force the valid MAC from
256^16 = 3.4e8 hashes to 16 * 256 = 4096; of course, this is an oversimplification,
but timing attacks are real, and recent body of work has demonstrated that they
are easier to exploit than it was thought before, especially if the attacker and
the victim are on the same physical machine, seperated only by a hypervisor.

Traditional approach to fixing the code above would look like this:
```c
char check16_bad(unsigned char *a, unsigned char *b) {
  unsigned i;
  char bad = FALSE;
  for (i = 0; i < 16; i++) {
    bad |= (a[i] != b[i]);
  }

  return bad;
}
```

There are several issues with this approach.  The first is that modern compilers
are fairly clever and may optimize the code in unpredictable ways that may
render a seeminly constant-time code not constant time.  The second issue is
that some code which seems constant-time may not necessarily be constant time;
for example, the code may rely on integer division, which is not constant time
on modern Intel CPUs and may depend on the size of the argument.  The third
issue is that ensuring that code runs in the constant time manually is hard, and
not reliable.

Our approach is that we create a language in which the timing of the code does
not depend on the user input (or, in the first iteration of the language design,
on any input at all).  The key principles here are the following:

1. The language is minimal and the only operations it exposes are the primitives
   which are carefully tested to be constant-time.  For example, if it chooses
   to expose integer division, then this implies that the programmer may safely
   use it, without having to worry about the details of the instruction timing
   on Intel CPUs.
2. The language exposes familiar control flow syntax to the programmer; however,
   the compiler transforms all branches into the code which evaluates both
   codepaths and chooses the correct one afterwards, thus making code constant
   time.
3. All loops are iterations over arrays or buffers of well-known size.
4. All functions in the language are side-effect free.

Note that first two restrictions above are too strict, which may lead to
suboptimal code performance.  In order to solve this issue, the language
seperates public and private data on the semantic level;  non-constant-time
branches are allowed only if the compiler can prove using the information about
the data in the program that the information in question is derived solely from
public data.

Data model
---------

The language currently provides following data types:
- bool as a boolean type accepting "true" and "false"
- int8, int16, int32 and int64 for signed integers
- uint8, uint16, uint32 and uint64 for unsigned integers
- intL and uintL as analogues of ptrdiff_t and size_t in C
- bufferM for a fixed size data buffer which has length divisible by M and
  which is aligned in memory by M, where M is a power of 2 from M=8 to M=256
- tuples
- structs 
- enums
- (implicitly) iterators

Most of those types are straightforward; note the notable omissions of
floating-point types, which are not really something anyone should use in their
cryptographic code.  Also note that the only container type in traditional sense
here is buffer (and possibly array in future); this is caused by the fact that
iteration over dynamically-sized data structures in constant time is
problematic.

All buffers are immutable.

Working with buffers is not trivial, because of constant time primitive
guarantee.  One of the operations one can do straighforwardly is to iterate over
elements in the buffer:

    /**
     * Determine the number of leading zeroes in the buffer.
     */
    intL count_leading_zeroes(buffer8 data) {
        intL result = 0;
        bool stop = false;
        for (uint8 byte : data) {
            if (byte == 0 and !stop) {
                result++;
            } else {
                stop = true;
            }
        }

        return result;
    }

Note that because of that limitation, the language has to provide flexible
iterator manipulation, similar to Python itertools:

    /**
     * Determine the size of padding in the buffer.
     */
    intL count_padding_zeroes(buffer8 data) {
        intL result = 0;
        bool stop = false;

        for (uint8 byte : reverse(data)) {
            if (byte == 0 and !stop) {
                result++;
            } else {
                stop = true;
            }
        }

        return result;
    }

Iterators can be used to iterate over two buffers at once using tuples:

    /**
     * Check if a MAC is bad -- contrived version; in real code, you would write
     * expected == provided, which would probably be optimized for buffer size.
     */
    bool check_mac_match(buffer8 expected, buffer8 provided) {
        for (uint8 a, uint8 b : zip(expected, provided)) {
            if (a != b) {
                return false;
            }
        }

        return true;
    }

There are of course, more non-trivial operations one would want to perform with
buffers.  The most elementary of them is sorting.  Decor uses Batcher sorting
network (which works in O(N log^2 N) time) in order to perform sorting with
fixed memory access pattern and number of operations.  Sorting in constant time
allows to perform arbitrary permutations, hence other useful operations (like
"move a substring of a string to the beginning and fill the rest of the list
with zeroes") can be implemented in terms of it.

Scoping
-------

Since all functions are free of side-effects, there are no global variables.
All variables are local, and scoped to the block they are in.  Obstructing a
variable in the outer scope is a syntax error.
