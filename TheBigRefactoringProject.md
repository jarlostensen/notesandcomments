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
            ...
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


