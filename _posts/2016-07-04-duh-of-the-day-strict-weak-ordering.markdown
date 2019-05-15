---
author: logan
comments: true
date: 2016-07-04 07:50:38+00:00
excerpt: "\n\t\t\t\t\t\t"
layout: post
link: http://noisecode.net/blog/2016/07/04/duh-of-the-day-strict-weak-ordering/
slug: duh-of-the-day-strict-weak-ordering
title: "\n\t\t\t\t'Duh' of the Day: Strict Weak Ordering\t\t"
wordpress_id: 26
---


				


Today I was nearly bludgeoned to death by `std::map` and my ignorance of strict weak ordering.




`std::map` is the best. In short, it implements an associative container of keys and values. You know how your contacts in your phone are sorted by name? You say to your address book, "Hey, I know the name of the person, now give me the phone number associated with that person' fs name." `std::map` is the tool for describing such a relationship in C++ code; you could create a `std::map` and have that represent your address book data. You can create any arbitrary key-value relationship; you provide the key, the map knows the value. `std::map`, `std::map<uspresident, favoritecolor>`, `std::map<species, endangeredstatus>`, `std::map<hotsauce, goodorbad>` would all be legitimate uses (assuming you had defined the types in question, of course).




It even provides some nice syntax, in particular the `[]` operator. If I want to access a member of the map, like Barack Obama's favorite color, I can use `presidentsFavoriteColors["Obama"]` to get it. But the nice thing is, if I haven't actually specified Obama's favorite color yet, the map won't yell at me; it will be nice and create the entry for Obama for me, and it will "just work." (The `mapped_type`, in this case `FavoriteColor`, must be default-constructible for this to work, which in this example may actually not be apt--what's the default favorite color?) This syntax is reminiscent of PHP's associative arrays, and I think discovering it was the first time something in C++ made me think, "Wow, how friendly!" Of course, there is also the often-important "unfriendly" (bounds-checking, exception-throwing) version of this same accessor, the `.at()` method.




Anyway, I've created a new type that I'm very proud of, and I want to use that type as the key in a `std::map`. Let's say the type looks like this:



    
    struct KeyType {
    public:
        int data;
        float otherData;
        KeyType() : data(0), otherData(0.f) {}
    };




Great! Now, I just have to declare a `std::map<KeyType, int>` and start using it.



    
    // somewhere in main()
    std::map<KeyType, int> t;
    t[KeyType()] = 1;




Shoot! Compiler error. `Invalid operands to binary expression ('const KeyType' and 'const KeyType')`. (That's `Apple LLVM version 7.3.0 (clang-703.0.31)`.)




Googling a bit, I see my mistake. You can't just use any object willy-nilly as a key for `std::map` (same goes for `std::set`). The reason is that objects in a map are sorted by their keys; much like the names in your address book are sorted alphabetically. This allows for very fast (`O(log n)`) insertion, deletion, and search operations. For trivial key types like `int`, this is innate; obviously 3 is less than 7, and so the object with a key of 3 should go first. For text strings, this also makes sense; sort them in alphabetical order. For my new type, `KeyType`, how is the map to know which one comes first?




In short, I need to overload `operator<`, so that the map knows how to tell if one key should come before another. (Don't try to be clever about this. Overriding `operator==` or `operator<` won't "just work" here, you'll have to provide the map with a custom comparator function. Just stop.)




_This is because the C++ standard library often defines equality by `!(a, not `a==b`._




_``_Okay, so I need to specify when one `KeyType` is less than another; easy.



    
    bool operator<(const KeyType& lhs, const KeyType& rhs) {
        return lhs.data < rhs.data || lhs.otherData < rhs.otherData;
    }
    




_your mileage may vary; if your object is not a `struct` with all public members, you may need to make your operator a member of your object, or declare it a `friend` within your object._




Obviously, if I'm comparing two `KeyType`s, and either of the data members of A is less than that same data member of B, then A should be considered less than B, right?




Under this same (dumb) assumption, hours of my life and several magnitudes of joy went down the drain.




The `operator<` I've defined above does not conform to "Strict Weak Ordering." Strict Weak Ordering is defined as such: `if A is less than B, then B is not less than A`. Makes sense, and turns out to be crucial for `std::map` to function properly.




Consider two `KeyType` objects. Pseudocode:



    
    KeyType A = KeyType ( 1, 0.0 );
    KeyType B = KeyType ( 0, 1.0 );
    




Using the `operator<` I've defined above, think about `(A < B)`. `A`'s `int` member is greater than `B`'s, but its `float` field is less than `B`'s, so `operator<` returns `true`. However, by the same token, `(B < A)` also returns `true`, because the `int` in `B` is less than that in `A`.




Hence, `(A < B)`, AND `(B < A)`. This is mathematical nonsense, and it turns out, `std::map` doesn't like it either. If I were a clickbait article, I might say, "`std::map` HATES this!" It breaks everything internally, including inserting things in the first place, so your map will end up inserting your stuff and then not being able to find it later.




This is because my `operator<` does not conform to Strict Weak Ordering. I was trying to squash a bug in my code for hours/days; thinking there was a bug in the library implementation of `std::map`, thinking the universe was playing a trick on me. The light bulb moment for me came when I thought about ordering words in a dictionary. Consider three words, "apple," "Advil," and "banana." Which comes first, alphabetically?




Using my `operator<` shown above, the answer might be "banana," because the second letter of "banana" ('a') comes before the second letter of "apple" or "Advil." But everyone knows that's wrong; you don't even LOOK at the second letter until AFTER it's clear that the first letters are the same. "Banana" gets discarded immediately; it's between "apple" and "Advil," which "Advil" wins because, even though the first letters are the same, the second letter 'd' comes before 'p'.




Implementing an `operator<` that conforms to Strict Weak Ordering needs to follow the same logic. Pick the member with the greatest importance (it might not really matter), and let the comparison proceed to compare other fields IF AND ONLY IF the first fields are equal (like the first letters of "apple" and "Advil").




Here's a corrected `operator<`:



    
    bool operator<(const KeyType& lhs, const KeyType& rhs) {
        if ( lhs.data < rhs.data )
            return true;
        if ( rhs.data <lhs.data )
            return false;
        if ( lhs.otherData < lhs.otherData )
            return true;
        return false;
    }




Verbose? Maybe. Correct? Finally. We look at the first `int` member of each, and compare them; we move on to comparing the `float` IF AND ONLY IF the `int`s are different.




The above can be expanded ad infinitum, for any number of data members you'd like to compare to determine equivalence of keys. You can determine which members are important, and which aren't--some might not even be included in the comparison. But you need to make sure that your `operator<` is robust enough that it complies with Strict Weak Ordering. Otherwise operations like `std::map::find()` (as well as `std::set::find()`), `std::map::at()`, and others, will seemingly randomly fail, because the map doesn't even know how to find what is inside itself, because it's all mixed up.




Sounds a bit like the human condition! This problem, however, we can definitively solve. So, do it.




* * *




_Note from like 2.5 years later when trying to sort-of-revive this blog and going back and finding this post: _




Don't write your comparison operator like the above. The C++ standard library provides a function template called `std::tie` that will create a tuple of references (avoiding big copies or moves or whatever you're afraid of), and `std::tuple`s implement lexicographical comparison right of the box:



    
    bool operator<(const KeyType& lhs, const KeyType& rhs) { 
       return std::tie(lhs.data, lhs.otherData) < std::tie(rhs.data, rhs.otherData);
    }




And you're done. Same semantics as the above, but harder to get wrong, easier to write and read, and one less thing to worry about in that short, sweet life of yours.


		
