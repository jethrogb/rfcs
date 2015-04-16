- Feature Name: visibility_override
- Start Date: 2015-04-15
- RFC PR: 
- Rust Issue: 

# Summary

Allow privacy violations in certain contexts when explicitly indicated to 
reduce the need to copy around code.

# Motivation

It's often the case that you want to augment some piece of library code you 
didn't write yourself with minor changes. For example, you might want to 
re-implement a specific accessor to a container for speed reasons. Or you want 
to re-use a large piece of logic with some small modifications. When doing 
this, you inevitably run into privacy violations. See also the discussion on 
the Rust-internals forum: [1] and [2].

Currently, you have two options in Rust. The first is making a local copy of 
the library source code for your modifications and the second is copying the 
struct definition and doing an unsafe typecast. The reasons why these options 
are not sufficient are discussed in the *alternatives* section below.

Most, if not all, languages that implement information hiding also have some 
way around it through protected members with inheritance, reflection, or 
context switching.

# Detailed design

To provide much flexibility while limiting programmer mistakes due to 
inadvertent privacy violations, the following change to the privacy checker is 
proposed:

*Accessing private members of a `struct X` is allowed in an implementation 
of a `trait Y` for that `struct X` when the `impl` has the 
`visibility_override` attribute.*

Here's an example of how this would work:

```rust
use std::intrinsics::cttz32;
use std::collections::BitVec;

pub trait LowestBit {
    fn lowest_bit(&self) -> Option<usize>;
}

#[visibility_override]
impl LowestBit for BitVec {
    fn lowest_bit(&self) -> Option<usize> {
        // Currently: field `storage` of struct `collections::bit::BitVec` is private
        for (idx, &word) in self.storage.iter().enumerate() {
            if (word==0) { continue }
            return Some(idx * 32 + (unsafe {cttz32(word) as usize}));
        }
        return None;
    }
}
```

# Drawbacks

In general, for all the following drawbacks, keep in mind that all this this is 
already possible using unsafe type casts (see below), and doing that provides 
none of the compile-time guarantees that this implementation would provide.

## Circumvents information hiding

This feature circumvents information hiding/encapsulation. This might induce 
programming mistakes. However, the proposed implementation should prevent 
accidental use and only override visibility for programmers who know what they 
want to do.

## Unsafety

Using this feature can cause major bugs when the code you're interacting with 
assumes invariants that one is now able to modify and break. For example, if 
you change `Vec.cap` you can cause buffer overflows or even just reading 
`Mutex.data` might introduce data races.

## Backwards compatibility

Allowing access to private members makes backwards compatibility harder, as 
library users may use this functionality to depend on internal private 
definitions. Using this attribute can be thought of as a declaration that "if 
upstream breaks me it's my fault," but similar situations in the past have not 
always worked out that way.

# Alternatives

## Existing alternatives

As mentioned in the *motiviation*, there are two existing ways to get around the 
privacy problem.

### Making a local copy of the library source code for your modifications

While this is a safe and functional solution to the problem, it significantly 
burdens the developer, as they now need to distribute their own version of the 
library with their software, and they have to maintain this copy and keep it 
up-to-date.

In the case of the standard library, this might not even be possible. In that 
case, one might try to "rip out" the part of the library that the developer 
wants to modify, but this might be much larger than expected because of private 
dependencies and does not solve the maintenance problem.

### Copying the struct definition and doing an unsafe typecast

While possible, this turns something that used to be a privacy violation into a 
potential memory safety error. All compile-time guarantees are thrown out of 
the window. The Rust [Reference] guide explicitly states that "Reading data 
from private fields" is not considered unsafe.

## Alternative implementations

### Disabling privacy in unsafe blocks

This seems like an overly broad solution to the problem.

# Unresolved questions

Is the proposed change sufficient to cover all use cases accessing private 
members?

It has been suggested the attribute be called `unsafe_visibility_override` or 
similar so it shows up in greps for `unsafe`.

[1]: http://internals.rust-lang.org/t/add-an-allow-ignore-field-privacy-annotation/625
[2]: http://internals.rust-lang.org/t/extending-existing-functionality/1289
[Reference]: https://doc.rust-lang.org/reference.html#behaviour-not-considered-unsafe
