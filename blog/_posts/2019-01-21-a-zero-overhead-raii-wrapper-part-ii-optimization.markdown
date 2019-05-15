---
author: logan
comments: true
date: 2019-01-21 06:46:09+00:00
excerpt: "\n\t\t\t\t\t\t"
layout: post
link: http://noisecode.net/blog/2019/01/21/a-zero-overhead-raii-wrapper-part-ii-optimization/
slug: a-zero-overhead-raii-wrapper-part-ii-optimization
title: "\n\t\t\t\tA Zero-Overhead RAII Wrapper, Part II: Optimization\t\t"
wordpress_id: 65
tags:
- base
- boost
- c++
- class
- compressed
- ebo
- empty
- metaprogramming
- optimization
- pair
- raii
- struct
- template
- tuple
- unique_ptr
---


				


In the last part, we implemented a very simple stack-based RAII wrapper for an OpenGL buffer object. The code was simple enough, clean, and easy to use, but we paid the price of a tiny amount of space overhead required for storing a deleter along with the buffer's handle, which is a price we didn't have to pay in the handwritten version of the code. This overhead came from the fact that empty structs have a size of one byte in C++, and by storing the deleter along with the handle naively in a flat structure, this extra byte affected the total size of our RAII wrapper. We'll spend this part Doing Something About That.







#### A Short Disclaimer







We are literally talking about one byte here. The following is a dive into a rabbit hole, for the joy and academic thrill of discovery, for the fun of learning something new and maybe brushing up on our template metaprogramming skills. I do not claim that optimizing away one byte is a good use of substantial amounts of your personal time, or your company's time, if it's not something you are interested in for its own sake.







Alright, on with the geeking.







## Optimization







As it happens, implementations of `std::unique_ptr` are faced with this same problem: the `unique_ptr` instance has to carry around an instance of a (possibly/probably empty) user-provided deleter, or else the standard `std::default_delete`, which is also an empty struct, and thus has nonzero size. Their cause for fixing this nonzero-size problem is perhaps somewhat better motivated than our little use case. The standards committee is thoroughly concerned with moving C++ in a direction of simpler, safer, and better code, and `unique_ptr` (among other smart pointers) is one of the most exciting newer tools for attaining that. However, `std::unique_ptr` is up against the Goliath of _decades_ of programmers using owning raw pointers. In order to be a competitive, viable option that can be recommended wholeheartedly and unequivocally for all single-owner dynamically-allocated objects in every new C++ program, present and future, there simply has to be a way to implement it in a way that suffers _zero_ overhead over the equivalent raw pointer. 







And, you guessed it, there is.







In order to shave off that extra byte of overhead we introduced by having to cart around an instance of our empty deleter struct, we're going to use a tool that many `unique_ptr` implementations also use internally called a `compressed_pair`. A `compressed_pair` is fundamentally similar to a `std::pair`, in that it models two objects bundled together, but it takes advantage of a very common compiler optimization to store empty structs without using any extra space.







`compressed_pair` isn't provided by the standard library (though many standard library implementations have one they use internally). If your project uses Boost, go ahead and whip out `boost::compressed_pair` and you're done; you can skip the rest of this section. Your standard library may also implement `std::tuple` using the "compressed" technique; if so, use that, and you too can be excused. For the rest of us less fortunate (or those who want to learn), we're going to bang out a simple, bare-bones `compressed_pair` that will eliminate the overhead of the deleter in our RAII wrapper.







The optimization that powers `compressed_pair` is known as the _Empty Base Optimization_, and it is deliberately allowed by the C++ standard, and even required in certain circumstances. [cppreference](https://en.cppreference.com/w/cpp/language/ebo) describes it better and more succinctly than I could: the empty base optimization "[a]llows the size of an empty base subobject to be zero." We can see it in action here:






    
    struct A {};
    static_assert(sizeof(A) == 1, "");
    struct B : A { int i; };
    static_assert(sizeof(B) == sizeof(int), "your compiler is broken!");







Notice how, even though the size of `A` is normally 1, when it is used as a base class of `B`, that 1 byte disappears and the total size of `B` is simply the size of its `int` member. We can use this little interesting behavior to our advantage when writing our `compressed_pair`. If we instantiate a `compressed_pair`, for example, we'll just store `int`s as members as normal, but if one (or both) of the types given is an empty struct, we'll _inherit_ from it instead.







The following bits of code use some template metaprogramming tricks that are beyond the scope of this post to explain in detail, but I'll try to keep the results simple enough to read.







## wrap<T, bool>







Let's start by modeling those two fundamental pieces: sometimes we want to have a given type as a member, other times we want to inherit from it. We'll write a little helper that encapsulates this piece of the problem, and then we'll use it to build up the rest of the solution later.







I want a single template that represents wrapping an object. It should take a type `T` and a `bool`. When I pass `false` for the bool, I want it to have the object as a member. When I pass `true`, I want it to inherit from the object. I like calling this template `wrap` because that's all it does (this little wrapper can also be reused for other purposes, like [wrapping up your naked strings](http://noisecode.net/blog/2019/01/20/stop-using-bare-strings/)). I'll get us started:






    
    #include <type_traits>
    
    template<typename T, bool Inherit>
    struct wrap {
        T& get() noexcept { return t; }
        const T& get() const noexcept { return t; }
        T t;
    };







This is the so-called _primary template_, which, as you can see, models the case where the object is a member. We also provide two overloads of a method called `get()`, which provide a uniform interface for retrieving the object inside a `wrap` which might need to be `get`ted in different ways depending on the `Inherit` bool.







Here's the second version, a partial specialization of the primary template, where we inherit from `T` instead if the bool was given as `true`:






    
    template<typename T>
    struct wrap<T, true> : T {
        static_assert(std::is_class<T>::value && !std::is_final<T>::value,
                      "can't inherit from type given to wrap<T, true>");
        T& get() noexcept { return *this; }
        const T& get() const noexcept { return *this; }
    };







This version, as you can see, requires that `T` be a non-final class, since that's the only way we can inherit from it. (The `static_assert` is just for the user-friendly error message--you'll get an arcane error either way if you try to use the template with a non-class, or a with final class.) Its `get()` methods, curiously but reasonably, return a reference to itself, since by inheriting from a `T`, it _is_ a `T`.







We can test that it's working by instantiating it with some different types:






    
    wrap<int, false> w0{5};
    // wrap<int, true> w1{5}; // error: can't inherit from int
    wrap<DeleteBuffer, false> w2; // DeleteBuffer as member
    wrap<DeleteBuffer, true> w3; // DeleteBuffer as base class







What about this `bool` business? We're not going to be able to pass in literal `true` or `false` when it comes time to use this in our `compressed_pair`--we'll need to intelligently pick the one to use based on the nature of the types given to us. For this, we need a _metafunction_: a "function" that takes a type as a parameter and returns us another type as a result. (If the idea of a metafunction is new to you, take a deep breath, drink in the splendor of the wild world of C++ template metaprogramming on the horizon in front of you, and then immediately stop and watch [this incredible talk](https://www.youtube.com/watch?v=Am2is2QCvxY)).







Here's a metafunction that "returns" whether or not a given type is eligible for taking advantage of the empty base optimization (abbreviated "ebo"). Thankfully, the standard type traits library already provides all the important pieces, we just have to stick them together:






    
    template <typename T>
    struct can_ebo
        : std::integral_constant<bool,
                                 std::is_empty<T>::value &&
                                     !std::is_final<T>::value> {};







I won't explain everything metaprogramming-y going on here, but the important part is that we check whether the type is empty (which also implicitly checks if it's a class, since non-classes can't be empty), and whether it's final... alas, it's true, some poor sod could declare their empty class `final`, for some unsearchable reason, and render all of our ingenious EBO optimization non-applicable.







(Note: if you're using C++17, you can just do `std::is_empty_v<T>` above, and likewise for the other. Some platforms still have some catching up to do before this is possible... like mine.)







All that's left is to put this all together. We know how to switch between inheriting from something and having it as a member; and we know how to tell if we'll benefit from the empty base optimization. Let's take a stab at combining it all into a simple `compressed_pair`.







## compressed_pair







Our `compressed_pair` template is basically going to take two template parameters, for the first and second types, hand those two types off to two instantiations of `wrap`, and then inherit from both of them. We're also going to provide a way to access the two things within the pair, using the base `wrap`s' `get()` methods:






    
    template<typename First, typename Second>
    struct compressed_pair
        : wrap<First, can_ebo<First>::value>
        , wrap<Second, can_ebo<Second>::value> {
    private:
        using first_base = wrap<First, can_ebo<First>::value>;
        using second_base = wrap<Second, can_ebo<Second>::value>;
    public:
        First& first() noexcept { return first_base::get(); }
        const First& first() const noexcept { return first_base::get(); }
        Second& second() noexcept { return second_base::get(); }
        const Second& second() const noexcept { return second_base::get(); }
    };







We inherit from two `wrap` instantiations, one per type in the pair, with the essential bool derived from the result of our handy `can_ebo` metafunction. If we pass in `int` for `First`, our first base class will end up being a `wrap`. If we pass in `DeleteBuffer`, it'll end up being a `wrap`. The private `first_base` and `second_base` aliases are just to make the getter implementations easier to read and more resistant to copy/paste errors, and also to enable you to make jokes about how you totally got to `second_base` with a guy writing a blog post on the Internet.







(Notice how `.first()` and `.second()` are member functions, so they can do whatever the base classes deem necessary in order to give you the requested subobject. This differs from `std::pair`, where `first` and `second` are plain data members, which is the reason why a standards-conforming `std::pair` can never be implemented as a compressed pair--there's no way to be "smart" about what `first` and `second` give you. It's also the reason why `std::tuple` _can_ be implemented as a "compressed tuple," since you access its members through the function template `std::get`. There are still no guarantees in the standard that tuple will be implemented in this way, though--implementors may choose to do it, but it's purely a quality-of-implementation dealio.)







If we instantiate `compressed_pair` with the types that motivated this exercise, we can experience our moment of triumph in all its splendor:






    
    static_assert(sizeof(compressed_pair<GLuint, DeleteBuffer>) == sizeof(GLuint), "");







We have officially figured out a way to store _both_ a buffer handle and its deleter in one object, while only occupying the memory of the buffer handle. Huzzah!







## A Bug Fix







This works perfectly fine for this specific use case, but alas, for more general uses of `compressed_pair`, our implementation has a bug. You can see it quite easily, actually, if you try to instantiate a `compressed_pair<int, int>`. It immediately explodes:






    
    error: base class 'wrap<int, can_ebo<int>::value>' specified more than once as a direct base class
        , wrap<Second, can_ebo<Second>::value> {
          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~







(via clang)







The error message is quite clear, and it's true; by passing in `int` twice, we've created the exact same instantiation of `wrap` twice, and inherited directly from both of them. As you can infer from the error message, it's illegal in C++ to inherit from the same class more than once directly. ("Directly" meaning with zero intermediate base classes in between. You can definitely end up with more than one base class of the same type if there are base classes in between, leading to the dreaded _diamond problem_.)







Luckily, we can fix this very easily. We just need to make it so the first and second bases are different types. Since they're already templates, this is trivial. We can just add a dummy parameter to `wrap` that signifies _which_ base within `compressed_pair` the current instantiation is being used as. We don't even need to _do_ anything with the dummy parameter; it just needs to be present within the template parameter list, and filled in when we instantiate:






    
    template<typename T, int Which, bool Inherit>
    struct wrap {
        // same as before
    };
    
    template<typename T, int Which>
    struct wrap<T, Which, true> : T {
        // same as before
    };
    
    template<typename First, typename Second>
    struct compressed_pair
        : wrap<First, 1, can_ebo<First>::value>
        , wrap<Second, 2, can_ebo<Second>::value> {
    private:
        using first_base = wrap<First, 1, can_ebo<First>::value>;
        using second_base = wrap<Second, 2, can_ebo<Second>::value>;
    public:
        // same as before
    };
    







Now we have no problem instantiating a `compressed_pair<int, int>`, since the two base classes are no longer the same: one is `wrap<int, 1, false>` and the other is `wrap<int, 2, false>`. Maybe structurally the same, but in the eyes of the compiler, totally different types.







## RAII







Oh, right! We were doing all this because we were actually going to use it for something. Previously, in our `RAII` class template, we had two "dumb" members of type `T` and `Deleter` (much like the "dumb" `first` and `second` in `std::pair`), and we were suffering some needless overhead because of that. Let's look at the implementation using our new, fancy, if minimal, `compressed_pair`:






    
    template<typename T, typename Deleter>
    class RAII {
    public:
        RAII(T t) : pair{std::move(t)} {}
        ~RAII() {
            pair.second()(pair.first());
        }
    
        T& get() noexcept { return pair.first(); }
        const T& get() const noexcept { return pair.first(); }
        operator T&() noexcept { return get(); }
        operator const T&() const noexcept { return get(); }
    
        RAII(const RAII&) = delete;
        RAII& operator=(const RAII&) = delete;
    private:
        compressed_pair<T, Deleter> pair;
    };
    
    static_assert(sizeof(RAII<GLuint, DeleteBuffer>) == sizeof(GLuint), "");







Here we've swapped out our two members for a single `compressed_pair` instance, and adjusted our getter and destructor implementations to match. We've also got a `static_assert` just below to gloat that our RAII wrapper now has zero overhead versus manually keeping a handle around, and manually destroying it when it comes time.







## Where to now?







Use this wrapper in a few different spots in code and you'll start to see it's somewhat... limited. It's not copyable or movable, so literally its only capability is just sitting there once it's created. You can't even return it from functions, or pass it into them except by reference, and you can't really store it in a data structure in a useful way. You _can_ always imbue it with movability or copyability by sticking it in a `unique_ptr` or a `shared_ptr`, respectively, which realistically is probably a fine option, but to my fellow rabbit hole spelunkers: we just spent all this time shaving off a razor thin amount of overhead, so do we really want to just go and stick this on the heap after all that work?







Next time we'll fix this problem by introducing opt-in move semantics for instantiations of the `RAII` template, using inspiration from [policy-based design](https://en.wikipedia.org/wiki/Modern_C%2B%2B_Design#Policy-based_design). Until then.


		
