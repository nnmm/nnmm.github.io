---
layout: post
title:  "Folding arguments into the macro"
date:   2023-01-11 20:02:03 +0000
author: me
categories: rust
tags:   rust
---

In Rust, you might have written macros to define structs, functions etc. that are similar but not the same. This is a technique that can make these macros nicer in some cases. I'm sure it has been used before, but I haven't seen it described anywhere yet.

For instance, say you have two structs, one dealing with `u8`, one with `u16`:

{% highlight rust %}
pub struct Xtal {
    data: *mut u8,
}

pub struct Tha {
    data: *mut u16,
}

unsafe impl Send for Xtal {}
unsafe impl Send for Tha {}
unsafe impl Sync for Xtal {}
unsafe impl Sync for Tha {}
/* … many other trait impls … */

impl Xtal {
    const FLUIDITY: f32 = 4.53;
    
    pub unsafe fn span(&self, len: usize) -> String {
        let slice = std::slice::from_raw_parts(self.data, len);
        String::from_utf8(slice.to_vec()).unwrap()
    }

    pub unsafe fn pulsewidth(&self) -> f32 {
        let bytes = [
            self.data.offset(0).read(),
            self.data.offset(1).read(),
            self.data.offset(2).read(),
            self.data.offset(3).read(),
        ];
        f32::from_ne_bytes(bytes) * Self::FLUIDITY
    }

    /* … many other methods that are the same for both structs … */
}

impl Tha {
    const FLUIDITY: f32 = 9.48;
    
    pub unsafe fn span(&self, len: usize) -> String {
        let slice = std::slice::from_raw_parts(self.data, len);
        String::from_utf16(slice).unwrap()
    }

    pub unsafe fn pulsewidth(&self) -> f32 {
        let bytes = {
            let [b0, b1] = self.data.offset(0).read().to_ne_bytes();
            let [b2, b3] = self.data.offset(1).read().to_ne_bytes();
            [b0, b1, b2, b3]
        };
        f32::from_ne_bytes(bytes) * Self::FLUIDITY
    }

    /* … many other methods that are the same for both structs … */
}
{% endhighlight %}

This is way too much repetition for my tastes. It would be easy to forget one important trait or method implementation for one struct, or for them to go out of sync in some way.

The only differences between the structs are:
1. the type of the `data` field
2. the `FLUIDITY` constant has a different value
3. as a consequence of the different data types, the bodies of `span()` are not the same
4. and the same for `pulsewidth()`

Let's put aside the possibility of reducing repetition and enforcing uniformity with a trait. Traits may be a good fit for this toy example, but they aren't always what you want.

So, how does this look when we use macros? I might write something like this:

{% highlight rust %}
macro_rules! create_struct {
    ($s:ident, $dt:ty, $fluidity:literal, $string_func:path, $get_bytes:expr) => {
        struct $s {
            data: *mut $dt,
        }

        unsafe impl Send for $s {}
        unsafe impl Sync for $s {}
        /* … many other trait impls … */

        impl $s {
            const FLUIDITY: f32 = $fluidity;

            pub unsafe fn span(&self, len: usize) -> String {
                let slice = std::slice::from_raw_parts(self.data, len);
                $string_func(slice.into()).unwrap()
            }

            pub unsafe fn pulsewidth(&self) -> f32 {
                let bytes = $get_bytes(self.data);
                f32::from_ne_bytes(bytes) * Self::FLUIDITY
            }
        }
    };
}

create_struct!(Xtal, u8, 4.53, String::from_utf8, |ptr: *mut u8| [
    ptr.offset(0).read(),
    ptr.offset(1).read(),
    ptr.offset(2).read(),
    ptr.offset(3).read(),
]);

create_struct!(Tha, u16, 9.48, String::from_utf16, |ptr: *mut u16| {
    let [b0, b1] = ptr.offset(0).read().to_ne_bytes();
    let [b2, b3] = ptr.offset(1).read().to_ne_bytes();
    [b0, b1, b2, b3]
});
{% endhighlight %}

For the `span()` function, I was able to confine the difference to just one function call through clever (or horrible?) use of `.into()`, so all that needs to be passed in is the function name.

For the `pulsewidth()` function, I abstracted the parts of the methods that are different into a lambda which needs to be passed in. It works, but the macro invocation starts to get unreadable already.

Note that **every difference becomes a macro argument**. When you have many small differences, the macro argument list gets longer and longer.

## The technique

Create two macros, `choose_xtal!` and `choose_tha!` like this:

{% highlight rust %}
macro_rules! choose_xtal {
    (Xtal => $a:expr, Tha => $b:expr) => {
        $a
    };
}

macro_rules! choose_tha {
    (Xtal => $a:expr, Tha => $b:expr) => {
        $b
    };
}
{% endhighlight %}

and pass them into the `create_struct!` macro like so:

{% highlight rust %}
macro_rules! create_struct {
    ($s:ident, $choose:ident) => {…};
}

create_struct!(Xtal, choose_xtal);
create_struct!(Tha, choose_tha);
{% endhighlight %}

When we're generating `Xtal`, `$choose!` will always evaluate to the expression after `Xtal =>` and vice versa.

That means you're now able to express the differences *inline*:

{% highlight rust %}
pub unsafe fn span(&self, len: usize) -> String {
    let slice = std::slice::from_raw_parts(self.data, len);
    $choose!(
        Xtal => String::from_utf8(slice.to_vec()),
        Tha => String::from_utf16(slice)
    ).unwrap()
}
{% endhighlight %}

I think this is wonderful for readability. Instead of looking up the position of the `$string_func` in the argument list and then looking up the value used in the invocation of `create_struct!`, `String::from_utf8` and `String::from_utf16` are specified right where they are used. And there is no more shoehorning with `.into()`.

## The full example

The above definitions of `choose_xtal!` and `choose_tha!` only work for expression fragments. To make it work for types and literals, I added match arms with a tag in front to make sure we don't accidentally match the wrong arm:

{% highlight rust %}
macro_rules! choose_xtal {
    (ty Xtal => $a:ty, Tha => $b:ty) => {
        $a
    };
    (literal Xtal => $a:literal, Tha => $b:literal) => {
        $a
    };
    (Xtal => $a:expr, Tha => $b:expr) => {
        $a
    };
}
{% endhighlight %}

Same for `choose_tha!` with `$b`, and of course more fragment specifiers can be added as needed. And here is the full example.

{% highlight rust %}
macro_rules! create_struct {
    ($s:ident, $choose:ident) => {
        struct $s {
            data: *mut $choose!(ty
                Xtal => u8,
                Tha => u16
            ),
        }

        unsafe impl Send for $s {}
        unsafe impl Sync for $s {}
        /* … many other trait impls … */

        impl $s {
            const FLUIDITY: f32 = $choose!(literal
                Xtal => 4.53,
                Tha => 9.48
            );

            pub unsafe fn span(&self, len: usize) -> String {
                let slice = std::slice::from_raw_parts(self.data, len);
                $choose!(
                    Xtal => String::from_utf8(slice.to_vec()),
                    Tha => String::from_utf16(slice)
                ).unwrap()
            }

            pub unsafe fn pulsewidth(&self) -> f32 {
                let bytes = $choose!(
                    Xtal => [
                        self.data.offset(0).read(),
                        self.data.offset(1).read(),
                        self.data.offset(2).read(),
                        self.data.offset(3).read(),
                    ],
                    Tha => {
                        let [b0, b1] = self.data.offset(0).read().to_ne_bytes();
                        let [b2, b3] = self.data.offset(1).read().to_ne_bytes();
                        [b0, b1, b2, b3]
                    }
                );
                f32::from_ne_bytes(bytes) * Self::FLUIDITY
            }
        }
    };
}

// See above for choose_xtal! and choose_tha!
create_struct!(Xtal, choose_xtal);
create_struct!(Tha, choose_tha);
{% endhighlight %}



## So I should use this argument passing style everywhere, right?

Of course not. When the argument is used multiple times, you'd need to repeat yourself when inlining it into the macro body. So in these cases, you might not win anything with this technique and instead introduce new opportunities for errors.

Even when an argument is just used once, I wouldn't blindly do this – e.g. maybe using `$choose` for the data type is already too much here, maybe it's clearer to see the data type mentioned in the macro invocation.

I think the biggest benefit comes from using `$choose` for expressions, like the ones in `span()` and `pulsewidth()`.

This technique can probably be extended and prettified, e.g. to allow something like `$choose!(Xtal => foo, _ => bar)`. But the basic version has already been useful for me.
