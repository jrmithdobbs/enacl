# Erlang bindings for NaCl

This library provides bindings for the NaCl cryptographic library for Erlang. Several such libraries exist, but this one is a re-write with a number of different requirements, and foci:

### INSTALL/Requirements:

* Erlang/OTP 17.3. This library *needs* the newest dirty scheduler implementation.
* *Requires* the libsodium library. *Note:* libsodium is not packaged in Debian/Ubuntu by default. You need to use something to handle the installation for you. E.g., `checkinstall` or `stow` are good tools for this. For other systems, consult your package manager on how to install the package. Make sure you also get "development" packages containing the header file `libsodium.h`.

To build the software execute:

	make
	
or

	rebar compile

### Features:

* Complete NaCl library, implementing all default functionality.
* Implements a small set of additional functionality from libsodium. Most notably access to a proper CSPRNG random source
* Tests created by aggressive use of Erlang QuickCheck.
* NaCl is a very fast cryptographic library. That is, crypto-operations runs quickly on modern CPUs, with ample security margins. This makes it highly useful on the server-side, where simultaneous concurrent load on the system means encryption can have a considerable overhead.

This package draws heavy inspiration from "erlang-nacl" by Tony Garnock-Jones, and started its life with a gently nod in that direction. However, it is a rewrite and it alters lots of code from Tony's original work.

In addition, I would like to thank Steve Vinoski, Rickard Green, and Sverker Eriksson for providing the Dirty Scheduler API in the first place.

# USING:

In general, consult the NaCl documentation at

	http://nacl.cr.yp.to
	
but also note that our interface has full Edoc documentation, generated by executing

	rebar doc
	
## Hints

In general, the primitives provided by NaCl are intermediate-level primitives. Rather than you having to select a cipher suite, it is selected for you, and primitives are provided at a higher level. However, their correct use is still needed in order to be secure:

* Always make sure you obey the scheme of *nonce* values. If you ever reuse a nonce, and an attacker figures this out, the system will leak the XOR difference of messages sent with the same nonce. Given enough guessing, this can in turn leak the encryption stream of bits and every message hereafter, sent on the same keypair combination and reusing that nonce, will be trivially breakable.
* Use the beforenm/afternm primitives if using the `box` public-key encryption scheme. Precomputing the Curve25519 operations yields much faster operation in practice for a stream. Consult the `enacl_timing:all/0` function in order to see how much faster it is for your system. The authors Core i7-4900MQ can process roughly 32 Kilobyte data on the stream in the time it takes to do the Curve25519 computations. While NaCl is *fast*, this can make it even faster in practice.
* Encrypting very large blocks of data, several megabytes for instance, is problematic for two reasons. First, while the library attempts to avoid being a memory hog, you need at least a from-space and a to-space for the data, meaning you need at least double the memory for the operation. Furthermore, while such large blocks are executed on the dirty schedulers, they will never yield the DS for another piece of work. This means you end up blocking the dirty schedulers in turn. It is often better to build a framing scheme and encrypt data in smaller chunks, say 64 or 128 kilobytes at a time. In any case, it is important to measure. Especially for latency.
* The library should provide correct success type specifications. This means you can use the dialyzer on your code and get hints for incorrect usage of the library.
* Note that every "large" input to the library accepts `iodata()` rather than `binary()` data. The library itself will convert `iodata()` to binaries internally, so you don't have to do it at your end. It often yields simpler code since you can just build up an iolist of your data and shove it to the library. Key material, nonces and the like are generally *not* accepted as `iodata()` however but requires you to input binary data. This is a deliberate choice since most such material is not supposed to be broken up and constructed ever (except perhaps for the Nonce construction).
* The `enacl:randombytes/1` function provides portable access to the CSPRNG of your kernel. It is an *excellent* source of CSPRNG random data. If you need PRNG data with a seed for testing purposes, use the `random` module of Erlang or Kenji Rikitake's `sfmt` bindings instead. But do note these do not provide cryptographically secure random numbers. The other alternative is the `crypto` module, which are bindings to OpenSSL with all its blessings and/or curses.
* Beware of timing attacks against your code! A typical area is string comparison, where the comparator function exits early. In that case, an attacker can time the response in order to guess at how many bytes where matched. This in turn enables some attacks where you use a foreign system as an oracle in order to learn the structure of a string, breaking the cryptograhic system in the process.

# TODO

* Write simple correctness unit tests for the different NaCl primitives. This will verify the integrity towards the underlying NaCl system.

# Versions

### v0.12.1

* Provide the `priv` directory for being able to properly build without manual intervention.

### v0.12.0

* Introduce an extension interface for various necessary extensions to the eNaCl system for handling the Tor network, thanks to Alexander Færøy (ahf).
* Introduce Curve25519 manipulations into the extension interface.
* Write (rudimentary) QuickCheck tests for the new interface, to verify its correctness.

### v0.11.0

* Introduce NIF layer beforenm/afternm calls.
* Introduce the API for precomputed keys (beforenm/afternm calls).
* Use test cases which tries to inject `iodata()` rather than binaries in all places where `iodata()` tend to be accepted.
* Fix type for `enacl:box_open/4`. The specification was wrong which results in errors in other applications using enacl.

### v0.10.2

Maintenance release. Fix some usability problems with the library.

* Do not compile the C NIF code if there are no dirty scheduler support in the Erlang system (Thanks to David N. Welton)
* Fix dialyzer warnings (Thanks Anthony Ramine)
* Fix a wrong call in the timing code. Luckily, this error has not affected anything as it has only replaced a verification call with one that does not verify. In practice, the timing is roughly the same for both, save for a small constant factor (Thanks to the dialyzer)
* Improve documentation around installation/building the software. Hopefully it is now more prominent (Thanks to David N. Welton)

### v0.10.1

This small patch-release provides tests for the `randombytes/1` function call, and optimizes EQC tests to make it easier to implement `largebinary`-support in EQC tests. The release also adds an (experimental) scrambling function for hiding the internal structure of counters. This is based on an enlarged TEA-cipher by Wheeler and Needham. It is neccessary for correct operation of the CurveCP implementation, which is why it is included in this library.

### v0.10.0

Ultra-late beta; tuning for the last couple of functions which could be nice to have. Added the function `randombytes/1` to obtain randombytes from the operating system. The system uses the "best" applicable (P)RNG on the target system:

* Windows: `RtlGenRandom()`
* OpenBSD, Bitrig: `arc4random()`
* Unix in general: `/dev/urandom`

Do note that on Linux and FreeBSD at the *least*, this is the best thing you can do. Relying on `/dev/random` is almost always wrong and gives no added security benefit. Key generation in NaCl relies on `/dev/urandom`. Go relies on `/dev/urandom`. It is about time Erlang does as well.

### v0.9.0

Ultra-late beta. Code probably works, but it requires some real-world use before it is deemed entirely stable.

Initial release.

# Overview

The NaCl cryptographic library provides a number of different cryptographic primitives. In the following, we split up the different generic primitives and explain them briefly.

*A note on Nonces:* The crypto API makes use of "cryptographic nonces", that is arbitrary numbers which are used only once. For these primitives to be secure it is important to consult the NaCl documentation on their choice. They are large values so generating them randomly ensures security, provided the random number generator uses a sufficiently large period. If you end up using, say, the nonce `7` every time in communication while using the same keys, then the security falls.

The reason you can pick the nonce values is because some uses are better off using a nonce-construction based on monotonically increasing numbers, while other uses do not. The advantage of a sequence is that it can be used to reject older messages in the stream and protect against replay attacks. So the correct use is up to the application in many cases.

## Public Key cryptography

This implements standard Public/Secret key cryptography. The implementation roughly consists of two major sections:

* *Authenticated encryption:* provides a `box` primitive which encrypts and then also authenticates a message. The reciever is only able to open the sealed box if they posses the secret key and the authentication from the sender is correct.
* *Signatures:* allows one party to sign a message (not encrypting it) so another party can verify the message has the right origin.

## Secret key cryptography

This implements cryptography where there is a shared secret key between parties.

* *Authenticated encryption:* provides a `secret box` primitive in which we can encrypt a message with a shared key `k`. The box also authenticates the message, so a message with an invalid key will be rejected as well. This protects against the application obtaining garbage data.
* *Encryption:* provides streams of bytes based on a Key and a Nonce. These streams can be used to `XOR` with a message to encrypt it. No authentication is provided. The API allows for the system to `XOR` the message for you while producing the stream.
* *Authentication:* Provides an implementation of a Message Authentication Code (MAC).
* *One Time Authentication:* Authenticate a message, but do so one-time. That is, a sender may *never* authenticate several messages under the same key. Otherwise an attacker can forge authenticators with enough time. The primitive is simpler and faster than the MAC authenticator however, so it is useful in some situations.

## Low-level functions

* *Hashing:* Cryptographically secure hashing
* *String comparison:* Implements guaranteed constant-time string comparisons to protect against timing attacks.

# Rationale

Doing crypto right in Erlang is not that easy. For one, the crypto system has to be rather fast, which rules out Erlang as the main vehicle. Second, cryptographic systems must be void of timing attacks. This mandates we write the code in a language where we can avoid such timing attacks, which leaves only C as a contender, more or less. The obvious way to handle this is by the use of NIF implementations, but most C code will run to its conclusion once set off for processing. This is a major problem for a system which needs to keep its latency in check. The solution taken by this library is to use the new Dirty Scheduler API of Erlang in order to provide a safe way to handle the long-running cryptographic processing. It keeps the cryptographic primitives on the dirty schedulers and thus it avoids the major problem.

Focus has first and foremost been on the correct use of dirty schedulers, without any regard for speed. The plan is to extend the underlying implementation, while keeping the API stable. We can precompute keys for some operations for instance, which will yield a speedup.

Also, while the standard `crypto` bindings in Erlang does a great job at providing cryptographic primitives, these are based on OpenSSL, which is known to be highly problematic in many ways. It is not as easy to use the OpenSSL library correctly as it is with these bindings. Rather than providing a low-level cipher suite, NaCl provides intermediate level primitives constructed as to protect the user against typical low-level cryptographic gotchas and problems.

## Scheduler handling

To avoid long running NIFs, the library switches to the use of dirty schedulers for large encryption tasks. The target is roughly set at 1/10th of the 1ms budget at 100μs. That is, we have a threshold set such that work taking more than roughly 100μs will invoke the dirty scheduler. We currently care much more about the *progress* of the system rather than the *precision*. We care that another Erlang process gets to use the core so one process is unable to monopolize the scheduler thread. On the other hand, the price that a process pays to use encryption is something we care less about. A process may get a free ride or it may get penalized more than it should if it invokes crypto-code.

We currently use measurements to obtain some rough figures on the reduction counts different operations take. You can run these measurements by invoking:

	enacl_timing:all().
	
The current "typical modern machine" is:

	Intel Core i7-4900QM
	
When running benchmarks, we warm the CPU a bit before conducting the benchmark. Also, the script `benchmark.sh` can be used (altered to your CPU type), to disable the powersave mode of CPUs in order to obtain realistic benchmarks. Do note nothing was done to get a realistic disable of Intel's Turbo Boost functionality and this is a one-core benchmark. The numbers given are used as an input to the reduction budget. If a task takes roughly 134μs we assume it costs `134*2` reductions.

I'm interested in machines for which the schedules end up being far off. That is, machines for which the current CPU schedule takes more than 250μs. This is especially interesting for virtual machines, and machines with ARM cores. If you are running on very slow machines, you may have to tune the reduction counts and threshold sizes to get good latency on the system.

# Testing

Every primitive has been stress-tested through the use of Erlang QuickCheck with both *positive* and *negative* testing. This has been used to check against memory leaks as well as correct invocation. Please report any error so we can extend the test cases to include a randomized test which captures the problem so we generically catch every problem in a given class of errors.

Positive and negative testing refers to Type I and Type II errors in statistical testing. This means false positives—given a *valid* input the function rejects it; as well as false negatives—given an *invalid* input the functions fails to reject that input.

The problem however, is that while we are testing the API level, we can't really test the strength of the cryptographic primitives. We can verify their correctness by trying different standard correctness tests for the primitives, verifying that the output matches the expected one given a specific input. But there is no way we can show that the cryptographic primitive has the strength we want. Thus, we opted to mostly test the API and its invocation for stability.

Also, in addition to correctness, testing the system like this makes sure we have no memory leaks as they will show themselves under the extensive QuickCheck test cases we run. It has been verified there are no leaks in the code.

