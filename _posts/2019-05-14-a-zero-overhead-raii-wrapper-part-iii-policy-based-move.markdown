---
author: logan
comments: true
layout: post
title:  "A Zero-Overhead RAII Wrapper, Part III: Policy-Based Move"
tags:
- c++
- metaprogramming
- move semantics
- move constructor
- raii
- template
---
We left off last time with a nice zero-space-overhead class template for RAII-style management of resources without using the heap. The motivating example was wrapping OpenGL buffer handles, which need to be created and destroyed responsibly, in a wrapper class that would be as lightweight as if we carried out all of our doings using the integer handle directly. Unfortunately, at the end of last time, we realized our wrapper class suffers a massive shortcoming that integers do not: it can't be copied or moved. This is particularly devastating for the motivating example of an OpenGL buffer, which is something you'd probably like to create and keep around for a while--maybe stick in a data structure, give to your `Mesh` class as a member, etc. With the current can't-copy-or-move situation, you can't even get a buffer to survive a function return without putting it on the heap.

Why not? I specifically deleted the copy constructor for the wrapper in the sample code, but why on earth would I do that? The answer lies in the essence of the abstraction we are modeling. An RAII class represents an owner of a resource, one that is responsible for carefully and correctly disposing of that resource when the time comes. If you naively copy an instance of the wrapper class, you end up with two instances who believe they are responsible for destroying the resource, and that leads to Bad Things, trust me. The only meaningful way to copy an RAII wrapper like the one we are writing (without changing to a more sophisticated ownership model, like e.g. `std::shared_ptr` does) is to _deep copy_ the data it owns. In this case, that would mean asking OpenGL to create a new buffer, and copying the existing data over to it; all told, a potentially incredibly expensive operation, and one we'd like to avoid if possible, especially in the likely case that the user of our class didn't intend for it to happen. `std::unique_ptr` avoids this potential for hugely expensive copies by disabling copying outright--we'll follow its lead and do the same. If the user wants a copy of their vertex buffer, they'll have to do it themselves.

_Moves_, on the other hand, are totally reasonable for an RAII wrapper to support. By moving an instance of an RAII wrapper, you are transferring ownership from one instance to another. The old instance should "forget" about the resource, and the new instance should take on the responsibility of freeing it. This means that each instance of the wrapper needs some new piece of state to keep track of whether or not it is still responsible for freeing its resource. For a wrapper around a pointer-like type, the value `nullptr` for the held pointer can conveniently signal that we no longer need to free whatever resource we once held. For more general wrappers, it's most likely a job for a separate `bool`. The logic is relatively simple, but since it does introduce the overhead of an added `bool` in our wrapper class (and we already spent an entire blog post shaving off a single byte to get it to zero-space-overhead), we'll get fancy by making movability an opt-in feature of the wrapper, so you don't have to pay for it if you don't need it.

_Note: this added complexity is all due to the fact that in C++, values that are moved-from still have their destructors run. That means when the destructor runs, we need to have kept track of whether we are moved-from or not, so that we know what we should do. If C++ had "destructive moves," like some other languages (e.g. Rust), we wouldn't need the extra bool or the logic--we could have our cake and eat it too._

We'll implement the opt-in fanciness taking inspiration from a pattern called "policy-based design."

## Policy-Based Design
This is a complex topic that could be explored for a while--the [Wikipedia article](https://en.wikipedia.org/wiki/Modern_C%2B%2B_Design#Policy-based_design) actually does so in some nice pedagogical depth--but we'll briefly summarize it by saying that a class that uses policy-based design accomplishes its goals by 1) being a template and 2) using its template arguments (usually via inheritance) as a way to parameterize its behavior. Consider the following example:

    struct Movable {
        Movable(Movable&&) = default;
        Movable& operator=(Movable&&) = default;
    };
    struct NonMovable {
        NonMovable(NonMovable&&) = delete;
        NonMovable& operator=(NonMovable&&) = delete;
    };
    template<typename MovePolicy>
    struct Foo : MovePolicy {};

    static_assert(std::is_move_constructible<Foo<Movable>>::value, "");
    static_assert(!std::is_move_constructible<Foo<NonMovable>>::value, "");

Because `Foo<Movable>` inherits from something with move enabled, it inherits that movability, whereas `Foo<NonMovable>` inherits the fact that its base's move constructors are disabled. So we can toggle the movability of an instantiation of `Foo` by changing which base class it inherits from.

We still need to do the bookkeeping of keeping track whether or not we have been moved _from_, so that in our RAII wrapper's destructor, we can decide whether to destroy the contained object, or whether that responsibility has been _moved_ elsewhere. We'll lay the groundwork for that decision-making within the base classes:

    struct Movable {
        Movable() : _moved{false} {}
        Movable(Movable&& other) : Movable{} { other._moved = true; }
        Movable& operator=(Movable&& other) {
            if (this != &other) {
                this->_moved = false;
                other._moved = true;
            }
            return *this;
        }

        bool moved() const { return _moved; }

    private:
        bool _moved;
    };
    struct NonMovable {
        NonMovable(NonMovable&&) = delete;
        NonMovable& operator=(NonMovable&&) = delete;

        bool moved() const { return false; }
    };

Our `Movable` base is a bit more complex than it was; it has to keep track of a bool representing whether it has moved, and make sure to set and reset the flag properly when a move occurs, both for itself _and_ for the thing it is stealing a resource _from_. It also has a `moved()` method, which simply returns the flag--the purpose of which we'll see in a moment. The `NonMovable` base, on the other hand, has only gained one line--like `Movable`, it now has a method called `moved()`, which always returns false, because of course it hasn't--it's non-movable.

Our RAII wrapper now becomes:

    template<typename T, typename Deleter, typename MovePolicy>
    class RAII : public MovePolicy {
    public:
        RAII(T t) : pair{std::move(t)} {}
        ~RAII() {
            if (!MovePolicy::moved())
                pair.second()(pair.first());
        }

        T& get() noexcept { return pair.first(); }
        const T& get() const noexcept { return pair.first(); }
        operator T&() noexcept { return get(); }
        operator const T&() const noexcept { return get(); }

        RAII(const RAII&) = delete;
        RAII& operator=(const RAII&) = delete;

        RAII(RAII&&) = default;
        RAII& operator=(RAII&&) = default;

    private:
        compressed_pair<T, Deleter> pair;
    };

Note the changes. We now take a third template parameter, `MovePolicy`, and inherit from it publicly. We also explicitly `default` our move constructors--we need to do this because we have a user-provided constructor and destructor, so `RAII`'s move constructors will be implicitly deleted otherwise. Now in the destructor, we call the `moved()` method that we inherited from the `MovePolicy` base class (the extra `MovePolicy::` prefixing the method call is just for clarity that that's where the method is coming from--we could have just as well omitted it).

Note: we don't _strictly know_ here that the type passed in as the `MovePolicy` template parameter _has_ a `moved()` method. Because C++ templates are [duck typed](https://en.wikipedia.org/wiki/Duck_typing), everything will work out if we pass in a type that does have a `moved()` method, but if we try to pass in something that doesn't, like say, `std::string`, the compiler will choke. A `static_assert`, or some kind of SFINAE/concept check/`if constexpr` dealio would make this interface nicer to use.

Anyway....

## There You Have It

We've got a zero-overhead RAII wrapper that is now actually useful, having imbued it with movability. Now we can use it in data structures, as a member inside other classes, and as function parameters and return types. This is one piece in building up a more complete useful abstraction, perhaps something like a `VertexBuffer` class, that manages the lifetime of the vertex buffer on the GPU, and also provides convenient APIs for manipulating vertex data. The possibilities for design are endless, and this is a useful tool indeed in our box.

I hope that this series has been informative, if a bit long-winded at times. The next post might be... shorter. Or maybe not. Old habits die hard.

;
