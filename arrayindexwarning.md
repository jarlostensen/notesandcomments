## A (very) annoying issue with std::unique_ptr operator[] 
Compiled with VisualStudio 2017, C++17, Warninglevel 4 (as errors).

I spent some time tracking this down, and it was one of those annoying issues that are only obvious once you've found it. 
It also, imoh, highlights some of the problems with C++ being a very rich, but also complex and intricate, language with many side effects and behaviours masking as others.

### The problem
The code I was compiling wasn't this, but the problem was most easily distilled down to something *like* this;

``` c++ highlight=8
template<typename K>
struct foo
{
    auto bar(const K key) -> decltype(key)
    {
        static std::unique_ptr<K[]> _table = std::unique_ptr<K[]>(new K[1001]);
        const auto tab = key % 1001;
        _table[tab] = key;  //< this is line 12 (see below)
        return tab;
    }
};
foo<uint64_t> myFoo;
int bar(uint64_t key)
{
    myFoo.bar(key);
}
```

Compiling it gives the following error;
``` c++
1>source.cpp(12): error C2220: warning treated as error - no 'object' file generated
1>source.cpp(8): note: while compiling class template member function 'void foo<uint64_t>::bar(const K)'
1>        with
1>        [
1>            K=uint64_t
1>        ]
1>strangewarning\source.cpp(30): note: see reference to function template instantiation 'void foo<uint64_t>::bar(const K)' being compiled
1>        with
1>        [
1>            K=uint64_t
1>        ]
1>source.cpp(29): note: see reference to class template instantiation 'foo<uint64_t>' being compiled
1>source.cpp(12): warning C4244: 'argument': conversion from 'const uint64_t' to 'size_t', possible loss of data
```

Not super helpful right away, and in my original code the compiler showed me the actual warning around a comparison between a ```table[tab]``` entry and a constant. 

### The solution
I was using a ```std::unique_ptr```, not a fundamental array type, so the index parameter needs to be of type ```size_t```, *explicitly*. In this case ```const auto``` just made ```tab``` the same type as ```K```, which in this case was a ```uint64_t``` and so triggered the warning.

Changing the line before the indexing to 
``` c++
const size_t tab = key % 1001;
```
fixes the problem.

Why was this not obvious? Well, for me at least, it wasn't obvious because, even though I knew I was using a ```std::unique_ptr```, I didn't immediately connect that I couldn't use ```operator[]``` the same way as always, with the compiler happily letting me use an integer, size_t, or whatever other integral type. 

It's a minor issue, but I for one believe that the authors of ```std::unique_ptr``` whould have spent the effort to make the array specialization compatible with the built in array type. 
Now I have to rememember to explicitly specify the type of a variable that could be used as an indexer, and specifically make it ```size_t```. If I refactored some code to switch from built-in to ```std::unique_ptr``` -array that would have added work and frustration that really shouldn't be neccessary. 
