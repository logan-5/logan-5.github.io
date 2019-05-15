---
author: logan
comments: true
date: 2019-01-21 00:17:22+00:00
excerpt: "\n\t\t\t\t\t\t"
layout: post
link: http://noisecode.net/blog/2019/01/21/a-zero-overhead-raii-wrapper-part-i/
slug: a-zero-overhead-raii-wrapper-part-i
title: "\n\t\t\t\tA Zero-Overhead RAII Wrapper, Part I\t\t"
wordpress_id: 63
tags:
- c++
- destructor
- idiom
- metal
- opengl
- raii
- shared_ptr
- unique_ptr
- vulkan
---


				


I've mentioned on this blog that I've been aspiring to hone my graphics-programming-fu, and part of this is involves using some serious APIs. There's OpenGL. There's Metal. There's Vulkan. There are [projects](https://github.com/KhronosGroup/MoltenVK) by certain well-known entities that implement Vulkan, but are _actually_ Metal under the hood. It's chaos.







I've been using OpenGL for a recent demo I've been spinning up, and it's been highly stimulating to my inner C++ developer. OpenGL is a C API, and an aging one at that, and much of it involves manipulating global state, and then remembering to undo those manipulations later. Any C++ programmer should start using OpenGL and immediately have their brain start buzzing about possible wrapper classes to encapsulate OpenGL objects (most of which are created somewhere mysteriously by OpenGL and handed back to you as opaque integer handles, by which you later reference, and then much later destroy, the object).







Most enticing is the thought of leveraging the C++ idiom known as _RAII_, which encapsulates resource management in an easy-to-use, hard-to-misuse way. Creating and destroying OpenGL objects using an RAII wrapper is a no-brainer, but there are other very common patterns when using OpenGL, such as binding and unbinding buffers, that can also be thought of as acquiring and later relinquishing resources.







## RAII Recap







There are exactly one billion resources that discuss this topic on the Internet, so I'll keep this very brief. RAII stands for the abstruse and verbose _Resource Acquisition Is Initialization_. I prefer to think of it as something more like DDSI: Destructor Does Something Interesting. When we acquire a resource, like some dynamic memory, a file handle (everyone's favorite example), an OpenGL object handle, or a binding of global OpenGL state, we want to stick the _relinquishment_ of that resource in some object's destructor. That way, relinquishment of the resource will happen automatically for us when the object is destroyed: if the object is on the stack, that means at the end of the block it lives in, or perhaps more compellingly, during stack unwinding--if and when we are rudely and violently interrupted by an exception, our resources will still be cleaned up properly.







There are a few ways to approach creating an RAII wrapper for an arbitrary resource, which we'll look at next. For the rest of this post, I'll be basing our RAII wrapper around the following OpenGL thingy:






    
    GLuint buffer;
    glGenBuffers(1, &buffer);
    // .. much later
    glDeleteBuffers(1, &buffer);







`glGenBuffers` asks the OpenGL driver to create some buffers; it takes an integer signifying the number of buffers we want to create, and a pointer to an array of integers that it'll populate with handles to the newly created buffers. Later, when we're completely finished with the buffers, we call `glDeleteBuffers` with the number of buffers we'd like to delete, and a pointer to an array of their handles. This API is begging for a neat way to wrap up the call to `glDeleteBuffers` in a way where we can't forget to call it, and even more importantly, a way where it'll be called automatically for us if something goes horribly wrong.







## unique_ptr







`std::unique_ptr` is a fantastic tool provided by the C++ Standard Library that provides RAII for dynamically-allocated memory. By default, when a `unique_ptr` is destroyed, it calls `operator delete` on the contained object (if any), but interestingly, it is also possible to specify a different deleter for the contained object which can perform custom cleanup logic. This sounds like a good start.






    
    struct DeleteBuffer {
        void operator()(GLuint* bufferHandle) const {
            glDeleteBuffers(1, bufferHandle);
        }
    };
    void foo() {
        GLuint bufferHandle;
        glGenBuffers(1, &bufferHandle);
        std::unique_ptr<GLuint, DeleteBuffer> bufferDeleter{&bufferHandle};
    }







Not, shall we say, the prettiest. It _works_, mind you--the `unique_ptr` calls `DeleteBuffer::operator()` when it is destroyed at the end of `foo()`, and our buffer is thereby cleaned up properly. But the initialization of the `unique_ptr` is awkward and verbose, caused in large part by the fact that `unique_ptr` expects to be dealing with, well, pointers. It stores a pointer internally, and expects its deleter to take a pointer as well (hence the `*` on the parameter of `DeleteBuffer::operator()`).







_Note: I initially thought you could work around this by adding `using pointer = GLuint;` to the definition of `DeleteBuffer`. The standard says `unique_ptr` will use (roughly) `Deleter::pointer` as the internal pointer type instead of `T*`, if that type/typedef is present. But, when I tried it, it didn't work--notably, the pointer type still has to be "pointer-like" in that it can be compared with / meaningfully assigned `nullptr`, which `GLuint` (which is just a typedef for `unsigned int`, by the way) cannot._







_Note #2: Why aren't we using a lambda for the deleter instead of this verbose, hand-rolled function object? Because of its promise of being a low-to-zero-overhead abstraction, `unique_ptr` requires that the specified deleter be stored directly inside the `unique_ptr` itself, not on the heap or in some other place. Because of that, the deleter is part of the type of the `unique_ptr`, and you need to specify it when spelling out the type of the `unique_ptr` (contrast this to, say, `shared_ptr`, which uses some type erasure tricks so that the deleter actually isn't part of the type). All that to say, it's too hard to spell the type of a lambda--we need a deleter type that we can refer to by name. N.B. I couldn't get it to work with deduction guides in C++17 mode either._







What's even worse about this approach is that we are using `unique_ptr` in a way it is not designed to be used. We are sneaking our buffer object handle into a `unique_ptr` when it really lives on the stack. The `unique_ptr`, however, has no idea this is happening; it genuinely believes it owns the buffer handle, and if we, for example, somehow moved it out of this function, it would take the pointer to the buffer handle with it--a pointer that will quickly become invalid as soon as `foo()` returns.







The best way to close this gaping hole where bugs can fall through is by using `unique_ptr` for what it's actually intended to do: manage dynamically-allocated memory. Our code becomes:






    
    struct DeleteBuffer {
        void operator()(GLuint* bufferHandle) const {
            glDeleteBuffers(1, bufferHandle);
            delete bufferHandle;
        }
    };
    void foo() {
        auto bufferHandle = std::unique_ptr<GLuint, DeleteBuffer>{new GLuint};
        glGenBuffers(1, bufferHandle.get());
    }







And this works. It's actually a lot cleaner too--the `unique_ptr` _is_ the buffer handle, rather than just being this weird deleter thing. If we move the `unique_ptr` around, add it to a data structure, whatever, the buffer will remain alive as long as necessary.







But it's still ugly. We've gained some exception safety and cleanup guarantees for our buffer object, but at what cost? Sticking ints on the heap. That's a big price to pay over the hand-written version. _Who cares_, I can hear you saying, _they're ints_--and yes, I agree, it's _probably_ not going to break the bank; but it DOES involve a little dynamic allocation, and it increases memory fragmentation and decreases locality of reference, and in a high-performance context like a graphics application, that kind of stuff can be important.







So how can we get the same exception safety and cleanup guarantees for our buffer, while keeping the handle on the stack where it belongs?







## Our Own Simple Wrapper







I won't leave you in suspense.






    
    template<typename T, typename Deleter>
    class RAII {
    public:
        RAII(T t) : object{std::move(t)} {}
        ~RAII() {
            d(object);
        }
    
        T& get() noexcept { return object; }
        const T& get() const noexcept { return object; }
        operator T&() noexcept { return get(); }
        operator const T&() const noexcept { return get(); }
    
        RAII(const RAII&) = delete;
        RAII& operator=(const RAII&) = delete;
    private:
        T object;
        Deleter d;
    };







This object stores an object of arbitrary type `T` inside it, along with a `Deleter` object which it calls in its destructor, passing the `T` as a parameter. This is the beating heart of the functionality we're trying to attain, distilled to its arguably simplest form. Note that everything is stored inside the RAII object itself--nothing is put on the heap or anything like that.







Note also that copying is disabled. Moving is also implicitly disabled by the presence of our user-defined constructor and destructor. It turns out copying and moving are a bit of a thinker with RAII types, one we'll save for part II or III or something.







Using this new wrapper looks like this:






    
    struct DeleteBuffer {
        void operator()(GLuint bufferHandle) const {
            glDeleteBuffers(1, &bufferHandle;);
        }
    };
    void foo() {
        RAII<GLuint, DeleteBuffer> buffer{0};
        glGenBuffers(1, &buffer.get());
    }







The buffer lives until the end of the function, when `RAII`'s destructor is called, and `DeleteBuffer::operator()` is invoked. Note also the presence of the implicit conversion operators in the definition of the `RAII` template, so we can also often (but not always) use the buffer as naturally as we could without the wrapper:






    
    glBindBuffer(GL_ARRAY_BUFFER, buffer); // RAII<T, D>::operator T&() called







We've eliminated the usage of the heap, and we've got a nice, usable, and RAII correct wrapper, but astute readers will realize that we are still paying some tiny, tiny space overhead. In fact, I'll prove it real quick:






    
    static_assert(sizeof(RAII<GLuint, DeleteBuffer>) > sizeof(GLuint), "");







So what's going on? The RAII struct bundles together a `GLuint` and a `DeleteBuffer`, which is an empty type, so it shouldn't add any space overhead. Right?







In the category of Things Every C++ Programmer Should Know, if you rummage long enough, you will sooner or later stumble upon this devious little factoid: the size of an empty type is not zero. It is, in fact, 1. There is a good (enough) reason for this: to ensure that every object has a unique address. If empty types were zero-sized, and you had a pointer to one, how would you increment that pointer? How would you loop over an array of objects of zero size? What would that even _mean_?







Anyway, whatever the reason, the fact of lugging around an instance of our empty `DeleteBuffer` struct within our RAII wrapper is adding one byte (or quite possibly more, because of alignment stuff) of overhead.







Now, you could very well, and very reasonably, say, "I do not care about wasting one byte. I've got guaranteed cleanup and exception safety, without using the heap, and I'm good with that. Goodbye and good day!" I, on the other hand, have never been good at avoiding rabbit holes. We'll discuss and implement a common, tried-and-true approach for shaving off the extra byte of overhead in the next part.


		
