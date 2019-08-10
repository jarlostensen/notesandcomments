## The big refactoring project
### Lessons learned transforming a deep OO C++ code base

#### TL;DR
Transforming a large-ish code base that had been evolving using traditional OO methods, implemented in C++, to a "c-with-c++" code base for readability, maintanability, and reason.
This is *not* an article filled with OO- or C++ -hate speech.

### The starting point
We had developed a code base to support a Windows desktop client application. Fundamentally it was a D3D based renderer and a number 
of service layers interacting with remote servers and marshalling information to and from the client PC.
The architecture lent itself readily to a class hierarchy; there was runtime polymorphism and there was layering of responsibilities.
A lot of functionality was generic and could be shared between multiple concrete implementations and the natural way to build this was 
to use a variant of the [template pattern](https://en.wikipedia.org/wiki/Template_method_pattern). This pattern allows us to write "scaffolding" code with embedded points of specialisation that we can delegate to a concrete implementation. It differs from a straight method override approach in that the subclass provides *partial* implementations that the base class injects where it needs it.

Code speaks more than a thousand words, so here's a code snippet to illustrate how this was done;

```c++
class ClientBase
{
public:
    bool Initialise()
    {
        // our initialisation consists of multiple steps, interleaved with "implementation specifics", which 
        // we delegate to using the protected virtual methods "On..."
        // this is effectively the "template pattern"
        if (BasePreInitialise())
        {
            // over to you, subclass (or just our stub which always succeeds)
            if( OnInitialise() )
            {
              ...
              return BasePostInitialise();
            }
        }
        ...
        OnInitialiseFailed();
        return false;
    }
    bool Something()
    {
        if(CertainConditionsAreMet())
        {
            SetupRequiredBaseState();
            // this bit needs a concrete implementation
            if ( DoSomething() )
            {
                StateUpdateAndOtherMisc();
                return true;
            }
        }
        ...
    }
    
protected:
    // the implementation provides these which get invoked at runtime
    // Do's *must* be implemented
    virtual bool DoSomething() = 0;
    // On's are optional
    virtual bool OnInitialise() { return true; }
    virtual void OnInitialiseFailed() {}
    ...
    // subclasses can access base state, anything non-implementation specific
    int GetSomeBaseState() const;
private:
    // these represent the concrete steps that the base class implements, into which subclass implementation is injected
    bool BasePreInitialise();
    bool BasePostInitialise();
};
...
class ClientImplementation : public ClientBase
{
public:
    ...
protected:
    bool DoSomething() override;
    bool OnInitialise() override;
    // ignoring OnInitialiseFailed, this implementation doesn't care (others might)
};
```
In the rest of the code we pick a concrete implementation, at runtime, depending on some conditions (for example what flavour of D3D the client rendering engine should be using, or some other conditions), and we hook up the endpoints. We had a lot of experimentation going on with different client implementations and cross platform scenarios at this time as well, so the flexibility was exactly what the doctor ordered. 
<br/>
In conclusion; the pattern works well for these types of scenarios, and in C++ it's easy to implement using virtual inheritance. 
Job done.
<br/>
...or is it?

### The wakeup call
While the code base is managed by one, perhaps two, people, and while you're working on it all the time everything is fine. You know how the logic flows through the code and you can easily jump to that source file where that implementation lives because you have the hierarchy, and the runtime dynamics, in your mind at all times. 
However, this bliss is short lived. 
We foresaw some of the challenges around reasoning about this type of code so we added a lot of documentation including helpful hints like "This OnSomething method gets invoked at location X in the hierarchy, source.cpp:line". What could possibly go wrong when the underlying design is so clean and simple?
<br/>
Now there are a lot of fancy words to describe it, like cognitive overload, or context switch cost, but I'll just call a spade by its name and say "it's a mess".
<br/>
The problem is the lack of locality in the code. Everything depends on "action at a distance", and a lot of it depends on what happens at runtime. The former simply means that when looking at a piece of code you can't reason completely about it because you don't have all the information you need. Some behaviour is delegated via "On's" or "Do's", which in itself wouldn't be so bad if it wasn't for the fact that you only know *who* will implement them at runtime. Code navigation tools can only suggest where you should look and you often end up firing up the debugger or mentally executing code to determine the exact flow. 
The use of C++ classes throws up another issue; they are realised as objects with their own state. Of course they are, that's the whole point, but what that means in the highly fragmented "spiderweb" of connections that you often get is that you end up carrying pointers and indirections around to access shared state or functionality.
<br/>




# In Conclusion
*Writing good, readable and maintainable, C++ code is HARD*. Refactoring often, questioning your assumptions, often, and remaining pragmatic about what solution you pick for a given job is crucial. Resist the urge to jump straight into a class hierarchy, ask yourself it it *makes sense*, and think about how runtime and static behaviour is mixed. An architecture that only becomes manifest at runtime is *very difficult to reason about*. 
<br/>
Stay safe, and good luck.

