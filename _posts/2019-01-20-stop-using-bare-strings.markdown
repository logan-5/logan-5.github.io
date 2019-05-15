---
author: logan
comments: true
date: 2019-01-20 05:28:10+00:00
excerpt: "\n\t\t\t\t\t\t"
layout: post
link: http://noisecode.net/blog/2019/01/20/stop-using-bare-strings/
slug: stop-using-bare-strings
title: "\n\t\t\t\tStop Using Bare Strings\t\t"
wordpress_id: 55
tags:
- c++
- class
- constructor
- explicit
- implicit
- scott meyers
- shared_ptr
- standard
- std::string
- string
- struct
- template
- type safety
- unique_ptr
---


				


I was going to use the word "naked" in the title, but decided against it when I realized there's enough nudity on the Internet already.







[This amazing-as-usual talk by Scott Meyers](https://www.youtube.com/watch?v=5tg1ONG18H8) has been haunting my brain recently, and it actually has nothing to do with his hair. He drives home important points that _should_ haunt the brain of anyone who strives to leave a positive legacy with the work they've done. Anyone whose biggest fear is poor innocent souls using his or her code down the line, staring in uncomprehending horror or aching disgust, and when the `git blame` comes with its unwavering judgment, they sigh and mutter, "Yep, the usual suspect." Anyone whose genuine desire is to write code that is, to harp on another Meyers-ism, "easy to use correctly, and hard to use incorrectly."







Watch the talk.







Near the end of the talk, Scott Meyers brings down his flaming sword upon the habit of many programmers, myself often included, to over-rely on strings. It was an argument I hadn't thoroughly considered before, and it left me feeling exposed, ashamed, and penitent, as divine judgment always does.







I want to contribute some thoughts on the matter, some shapes the problem takes, and some possible solutions.







## The Problem







Consider the following function:






    
    std::unique_ptr<Texture> load(const std::string& path);







Without being able to see its definition, what do we know about it? What do we _not_ know about it?







Well, clearly it's going to do some interaction with the file system to load some type of file, and create some sort of in-memory representation of a `Texture`, whatever that is (maybe we're in a graphics application?). A `Texture` is big enough that we want to put it on the heap (or perhaps it's polymorphic), and we're getting a `unique_ptr` to it, which is good--safety and efficiency by default, and we can always move it into a `shared_ptr` later if we need to.







What are we ignoring? If you're anything like me (and it's quite possible I'm alone here), there is an actual pang of anxiety when you see the `path` parameter, accompanied by a question: what kind of path? Absolute? Relative? If relative, relative to where? If I got the string I'm going to call this function with from somewhere else, how can I be sure it's the right type of path? Is it my job to do that, or is the function going to somehow validate the path, or convert relative paths to absolute paths, or something?







What's even more interesting than the pang of anxiety is that I, at least, have _learned to ignore_ it, because having absolute certainty about how to call functions is not something I have come to expect from programming. I expect to probably get it wrong the first time... I'll spend a couple minutes looking for the problem, and then I'll figure it out, and it'll be fine. Except when that couple minutes turns into a couple hours, or a couple days....







Alright, sorry. Less pontificating, more code. We get it, the string parameter is scary. I get rambly like this sometimes.







## Some Solutions







Here's one possible, tempting solution to this problem:






    
    using AbsolutePath = std::string; 
    // ... 
    std::unique_ptr<Texture> load(const AbsolutePath& path);







This is definitely a step in the right direction. The function definition is now _much_ more readable--it clearly expects an absolute path, and to be honest, it goes a long, long way toward eliminating my anxiety about calling this function--I know exactly how to do it correctly now. In codebases I've worked on, I've stopped here, and felt relatively good about it.







But of course, this relies on users of the function to go to the declaration to read the signature, and isn't resistant to incremental refactoring, nor does it have even a smidgen of type safety. Since `AbsolutePath` is just an alias for `std::string`, we have gained nothing at the call site:






    
    const std::string relativePath = "my_stuff/texture.png";
    auto texture = load(relativePath);







Even more blatantly/offensively:






    
    const std::string relativePath = "my_stuff/texture.png";
    const AbsolutePath absolutePath = relativePath;







In a perfect world, I would be warned, by the compiler or the linter or something, about making that mistake. No, scratch that--in a perfect world, it should be _impossible_ to make that mistake.







In fact, this is something that type systems are great for, and C++ comes with a very sophisticated, robust, and expressive one.






    
    struct AbsolutePath {
         explicit AbsolutePath(std::string s) : path{std::move(s)} {}
         std::string path;
    }; 
    struct RelativePath {
         explicit RelativePath(std::string s) : path{std::move(s)} {}
         std::string path;
    };







Now we get a compiler error:






    
    const RelativePath relativePath = {"my_stuff/texture.png"};
    const AbsolutePath = relativePath; // error: no viable conversion from 'RelativePath' to 'AbsolutePath'







This is good. We are prevented by the compiler from passing in the wrong type of path to `load`. Also, we can't just pass in plain strings to `load` anymore either (note the `explicit` constructors), giving the programmer an error if they try, and more importantly an impetus to stop and think about what they are trying to pass in. The only cost of these benefits is changing usage of the `path` argument in the implementation of `load` to `path.path` instead, grabbing the inner string out of the wrapper.







This API is now "hard to use incorrectly," to echo the Meyers mantra. But we still have (at least) one more step we can take toward making it "easy to use correctly." A key observation is that we can (probably, depending on the exact situation) create AbsolutePaths from RelativePaths in a programmatic way--appending the `pwd` and doing tilde expansion and all that jazz. Let's say we have correctly and neatly implemented that functionality in this function:






    
    std::string makeAbsolutePath(const std::string& relativePath);







_(I'm not too worried about the `std::string`s in this signature--this function's job is to take the actual sequence of characters that make up a relative path, and transform them into the sequence of characters that comprise an absolute path. Hopefully it's a static function tucked away in an implementation file somewhere, away from possible misuse.)_







Look what happens now, if we juggle our implementations of our path structs around a bit:






    
    struct RelativePath {
        explicit RelativePath(std::string s) : path{std::move(s)} {}
        std::string path;
    };
    struct AbsolutePath {
        explicit AbsolutePath(std::string s) : path{std::move(s)} {}
        AbsolutePath(const RelativePath& r) : path{makeAbsolutePath(r.path)} {}
        std::string path;
    };







What happened here? We have a new constructor taking a `RelativePath` which calls the appropriate conversion on the internal `std::string`s. Notably, this new constructor is not `explicit`, leading to some cool conveniences:






    
    auto texture = load(RelativePath{"my_stuff/texture.png"});







Wait, we passed a `RelativePath` to `load`? That's right! Because the `AbsolutePath` constructor in question was not made `explicit`, you can *implicit*ly construct one from a relative path, and the newly-constructed `AbsolutePath` will have called `makeAbsolutePath` to do all the work to make the actual absolute path needed. You can even do the super offensive version in this same way:






    
    const RelativePath relativePath{"my_stuff/texture.png"};
    const AbsolutePath absolutePath = relativePath;







The new `absolutePath` variable is now tilde-expanded and all that stuff, and everything "just works."







(Note: some people won't like this, and it legitimately may not suit your application. The implicit call to `makeAbsolutePath` is almost certainly going to involve dynamic memory allocation and lots of complicated logic, and having it happen without your explicit intention may be a performance bug, if you're programming on the chip in my washing machine.)







At the end of all this, we have type-safe paths that express our intentions when reading the source code, and are easy to use correctly (pass in a relative path if you want... I'll convert it to an absolute path for you!) and hard to use incorrectly (compiler errors if you do it wrong). This makes for a clean, friendly, usable API that gives my anxiety a much-needed break.







## Addendum







If you're like me, the code duplication here makes your eyes hurt a tiny, tiny bit. Okay, not even hurt, just kind of itch:






    
    struct AbsolutePath {
        explicit AbsolutePath(std::string s) : path{std::move(s)} {}
        std::string path;
    };
    struct RelativePath {
        explicit RelativePath(std::string s) : path{std::move(s)} {}
        std::string path;
    };







We have the _exact same_ shape of struct in this example, differing only in the name. In fact, differing in only PART of the name--they're both paths, just different sorts of paths.







We can factor out this duplication by using a small, simple template, and using a trick involving empty _tag types_. This was inspired by an approach I saw recently taken by Rust's [euclid](https://github.com/servo/euclid) crate for encoding units into their different geometric types.






    
    template<typename PathTag>
    struct Path {
        explicit Path(std::string s) : path{std::move(s)} {}
        std::string path;
    };
    struct RelativeTag {};
    struct AbsoluteTag {};
    
    using RelativePath = Path<RelativeTag>;
    using AbsolutePath = Path<AbsoluteTag>;







Using this trick you get the exact same semantics as the earlier approach--you still can't assign between `AbsolutePath` and `RelativePath` (they're different types still, because the template parameters are different), and the core logic is factored out. Of course, it is more lines of code, but since you're using a template, you get to feel smart. 







When we add the other constructor to `AbsolutePath`, it gets trickier. `RelativePath` can stay just being a type alias, but we actually need to create a new type for `AbsolutePath` so we can add a constructor to it. I solved it this way:






    
    struct AbsolutePath : Path<AbsoluteTag> {
        using Path::Path; // inherit base class constructors
        AbsolutePath(const RelativePath&); // yada yada
    };







Probably an overly complicated abstraction for this simple case, where we only have two types of paths, and their internal structure is really simple. It might be useful, though, for other applications of this pattern of wrapping simple, less-meaningful types in stronger wrappers.







;


		
