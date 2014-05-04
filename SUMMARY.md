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
- bytearray(N), which behaves itself like buffer, but with fixed size
- tuples
- structs 
- enums
- fixed-size arrays of anything above
- (implicitly) iterators

Most of those types are straightforward; note the notable omissions of
floating-point types, which are not really something anyone should use in their
cryptographic code.  Also note that the only container type in traditional sense
here is buffer (and possibly array in future); this is caused by the fact that
iteration over dynamically-sized data structures in constant time is
problematic.

Buffers are normally immutable.  Mutable buffers form a special type mbuffer,
which is not passed into the functions as an argument, but only returned.  Such
restricted model of buffers being passed around is motivated by the fact that
functions have to be side-effect free, hence passing arguments by reference is
out of question.  We assume that most cryptographic primitives can do well
within this model.

Working with buffers is not trivial, because of constant time primitive
guarantee, and due to the fact that the memory access patterns have to be
consistent regardless of data.  One of the operations one can do
straighforwardly is to iterate over elements in the buffer:

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

bufferM can be iterated as uintN as long as N is a factor of M.  Larger sizes of
buffer, like buffer128, are useful because, even though we do not provide data
types like uint128, compiler can use that to optimize copying and manipulating
those buffers using SSE.  Bytearrays are identical to buffers in how they are
maniupulated, except their type explicitly specifies the size of buffer.
Regular arrays are fixed-size as well, but they can be only addressed using
defined type; that type can be anything, except for mutable buffers.

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

Private and public data
-----------------------

The data in Decor programs may be either private or public.  The language allows
inference of privateness status of data based on the following rule: any data is
inferred to be public if it is derived solely from public data; otherwise, it is
considered private.  Any private data can go into public only if explicitly
exported.

There are certain constrains inherent in the language about how public and
private data may be used.  The length of any buffer passed to the program is
considered to be public even if buffer is private (FIXME: we should think this
through better? I see practical cases where this is not feasible, but I don't
see how to approach that problem).  The length of any buffer or array allocated
by program has to be public, and any buffer or array offset that is accessed
explicitly (as opposed to access through iteration) has to be public.

There are two types of functions in terms of how they track private/public data
controlled and uncontrolled.  Uncontrolled functions make no assertions
regarding the privateness of input or output; hence, it is inferred from
function invocation.  Controlled functions explicitly specify those parameters,
like this:

    public mbytearray(32) sha256_hmac(public buffer8 data, private buffer8 key);

Note an important issue here.  This prototype is consistent with how we use
HMAC when we send message in encrypt-then-MAC scheme: we input a publicly
accessible ciphertext with secret key, and the result is the MAC which is sent
on the wire.  In case of receiver, however, that assumption is not true: the
output is the **MAC which the other party have to demonstrate knowledge of**, it
is a secret MAC, and the only public data is the result of comparison of that
MAC against public MAC (and in some cases, that may not even be public, if the
context allows).  Because of that, Decor allows callers to pass private data
into functions where an argument is marked as public; the compiler will handle
that and compile different versions of functions optimized for different
assumptions of privacy.  Note that because of the limitations how private data
may be used, some public data arguments cannot be converted into private;
attempting to do so will result in compiler emitting an error explaining why
this is not possible.

(FIXME: maybe we should have different qualifiers for data which can be made
private and data which cannot?)

Whenever a function generates data which is derived from private data, but,
since it had been passed through a cryptographic algorithm which is believed to
protect that private data or for some other reason we believe that data is no
longer private, we can *export* it.

For example, consider this code:

    public mbytearray(32) sha256_hmac(public buffer8 data, private buffer8 key) {
        mbytearray(32) result;

        // Compute actual MAC here

        export(key) result;
    }

Note that we explicitly specify which private data we consider ourselves to be
not leaking due to that invocation; this way, if we ever compute HMAC over
something secret, the result will not become public.

Or let us revisit another example of function which needs such feature above and
see how things can get complicated.

    public bool check_mac_match(private buffer8 expected, public buffer8 provided) {
        for (uint8 a, uint8 b : zip(expected, provided)) {
            if (a != b) {
                export(expected) false;
            }
        }

        export(expected) true;
    }

In both cases, an export case is needed because the fact that the program
reached that codepath depends upon the private data (value of expected).  Note,
however, that export of value false inside the loop *does not* mean that 
this value is public in the context of determinig function's evaluation flow.
The data becomes public only as a result.

(FIXME: does it make sense to make an export list a part of the prototype?)

Error handling
--------------

(FIXME: there are some serious implications of this section, which I have not
thought through well; this might be a bad idea)

The language supports exceptions.  Exceptions work just as you expect them to
do: all the code is executed, as it would normally, except that in the return to
the outer program (about which we are about to talk next) instead of getting
meaningful results it gets an error.  In case where the error happens due to
out-of-bounds array access, division by zero and other error which normally
implies impossibility to resume working, the resulting data is filled with
garbage.

Integration into other programs
-------------------------------

Decor code is not meant to be ran standalone; it is intended to be compiled into
an object file which is then linked to a C binary (which itself may then be
exported into other languages).  The compiler produces two files: object file
with compiled Decor routines and a header file with C code needed to call those
functions.

The C code has a single function, which determines the capabilities of the CPU
running on the system and returns a struct with pointers to all exposed
functions.

Whenever a variable length buffer is passed in, it is passed as (unsigned char \*).
Output buffers are allocated with user-specified malloc by the decor code itself,
bytearrays are to be allocated by the application and passed in as pointer to
the fixed-size buffer where it is supposed to be sized.  It is up to application
to:
1. Free the memory returned as a variable-length buffer.
2. Ensure proper alignment of all input buffers and their size.
3. Not screw up on buffer checking.

(FIXME: it is possible that entrusting those tasks to C programmers is a bad
idea, and the resulting code would be to memory leaks and heartbleeds)
