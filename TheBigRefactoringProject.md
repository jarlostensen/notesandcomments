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
    virtual bool DoSomething() { return true; }
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
};
```

