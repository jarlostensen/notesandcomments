## The big refactoring project

Notes are being typed as the project progresses...it will converge on something coherent, soon.

### Lessons learned transforming a deep OO C++ code base

#### TL;DR
Transforming a large-ish code base that had been evolving using traditional OO methods, implemented in C++, to a "c-with-c++" code base for readability, maintanability, and reason.
This is *not* an article filled with OO- or C++ -hate speech.

### The starting point
We had developed a code base to support a Windows desktop client application. Fundamentally it consisted of a number 
of service layers interacting with remote servers and marshalling information to and from the client PC.
The architecture had started out with a high degree of flexibility in mind with support for multiple platforms, etc, and had evolved organically. Maintainability and extensibility had clearly suffered but a mentality of "if it isn't broke, don't fix it" had kept it from being properly overhauled until I put on my head phones, rolled up my sleeves, and got to work analysing it and synthesising a cleaner version. 
<br/>

The analysis highlighted a couple of "anti patterns" which were at the root of the problems with the code base:

### Classes as Namespaces
This seems fairly common and is a result of the tendency to *default to classes* when designing code in c++. 
Remember that a ```class``` is a *type* in c++, whereas a ```namespace``` is something you use to group functionality logically. 
The former is something you can have many of, manifest as objects, the latter is just a handy way of allowing you to name things in better ways. 
<br/>
One immediate problem with this is the amount of boiler plate a class often requires which makes the code harder to read, as this simple example illustrates
```c++
class MySimpleClass
{
public:
    MySimpleClass();
    MySimpleClass(const MySimpleClass&);
    MySimpleClass(MySimpleClass&&);
    MySimpleClass& operator=(const MySimpleClass&);
    MySimpleClass& operator=(MySimpleClass&&);
    ~MySimpleClass();
    
    bool MyLittleFunction(int);
};
```

### Stateful pipelines, or high-latency programming
This reminds me of rendering pipelines in graphcis programming where you build up a large amount of state for various stages of a pipeline, then execute the pipeline. In the code case this is where classes contain large amount of state that is set with mutators and this state influences methods executed later.

### Deeply nested responsibilities 
TODO: obfuscated code in deep function hierarchies, helpful "utitlity" classes and functions generalising where it's not needed.

### Functions as programmes
TODO: APIs that hide high degrees of complexity, in particular relating to resource management such as threads.

#### TBD
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
While the code base is managed by one or two people who are working on it all the time everything is fine. They know the logic flow through the code and can make the required mental jumps around the architecture and the source with ease. 
We even added a lot of documentation including helpful hints so what could possibly go wrong?
<br/>

The problem is *lack of locality* when many things depend on action at a distance, and in time. 
Code navigation tools can at best suggest where you should look and you often end up firing up the debugger or mentally executing code to determine the exact flow. (A classic example of this, in VisualStudio, is ```ctrl-click``` on a method to go to its declaration which in the case of a virtual function only takes you to the interface, you have to find the implementation yourself.)
<br/>

Another problem is the way heavily object-oriented code can end up with emerging state and state management, further encapsulating information needed for local understanding. In pure OO terms this is not a bad thing and when used correctly it is a powerful tool, but we found that having state distributed across many classes often resulted in duplication or "parallell implementations", simply because it was so hard for new programmers to get the full view of where state lived and how it influenced different parts of the code.
<br/>

Keep in mind that this architecture started out looking very clean on paper and it was implemented in a few source files and headers. In theory this should not be a problem, and well created designs, UML diagrams, and deep documentation would of course mitigate all of the issues, in an ideal world.

<br/>
The result is exactly what you'd expect; people create shortcuts and connections where there shouldn't be any, or where a connection already exists but is hard to spot. The spider's web starts to grow and you get duplication of state, stray pointers into all corners of the hierarchy, and outright sabotage of separation of concerns. Because, ultimately, people have a job to do.

# The remedy
The first thing to do when refactoring is to take a step back. Read the code, run it through a debugger to fully understand the flow, then step back again. In our case this exercise uncovered a few key points:
* Runtime polymorphism wasn't really needed; we didn't have implementations being swapped in and out, they were decided on very early on, or even at compile time. I suspect that's more often the case than not.
* The many paths up and down the virtual inheritance hierarchy made the code flow very hard to understand on first contact. We had organic and incongruent growth of patches of code addressing immediate needs as a result.
* Classes and data isolation didn't really give us much as we ended up with pointers to objects everywhere. Heavy use of the pimpl idiom that had helped isolate headers from implementation early on started to become a problem with state leaking out into the header space of classes (which is **bad** but people make mistakes.)
* A collection of different functional modules emerged, for example to deal with telemetry gathering and accumulation, which had dependencies on shared state between other modules using it. This further exacerbated the spider web of pointers. 
* Deep call stacks and heavily branching code that calls off to many functions make code even harder to read and debug. In many cases the depth and jumps are the result of invalid assumptions; stowing code away in a function might seem like good practice, but if the code is only ever used in *one* place then why? We found that this old and tested cleanliness practice tended to cascade and result in function 1 wrapping code that calls function 2 wrapping code etc making the code harder and harder to reason about statically.

The realisation that runtime polymorphism was uneeded is probably the most important from a pure code perspective since it implies that *objects are no longer needed*. When you architecture no longer requires objects being passed around the amount of complexity coming from considering each of them as having their own unique state no longer applies. 

Consider the following (contrived) example:
```c++

struct SomeSharedStateWeEndedUpWithForReasons
{
    ...
    std::vector<float>  _telemetry_samples;
    ...
    void AddSample(float sample);
};
...
class SomeClassWithState
{
    ...
    int GetValue() const;
    void SetValue(int val);
    void DoSomethingWithValue();
    ...
    SomeSharedStateWeEndedUpWithForReasons* _shared_state;
};
...
class SomeOtherClass
{
    ...
    void DoSomething(SomeClassWithState* that)
    {
        ...
        auto myval = that->GetValue();
        ...
        that->SetValue(42);
        ...
        that->DoSomethingWithValue();
        ...
        _shared_state->AddSample(aFloatValueWeGenerated);
};
```
<br/>
> Keep in mind that this is the story of a real-world implementation in a real-world team with day jobs and a million things to do. You may argue that "don't do it this way from the start" is a solution but that's often not a luxury we have in the real world.

## Summary, take II, after the refactor
* The class hierarchy is completely gone as part of the API. Classes are used internally but not publicly exposed.
* The code is broken out into a variant of a [service oriented architecture](https://en.wikipedia.org/wiki/Service-oriented_architecture), i.e. presented as self contained, black box, that as much as possible is *stateless*. 
* An emphasis on locality; instead of setting state somewhere in a class object which is then implicitly assumed by its functionality we explicitly provide the state at the call site when using that functionality. *Oh look, it's C, I hear you say...yes, yes, alright then*.
* In addition to removal of class hierarchies we have overall reduced the *depth* of implementations, i.e. the number of function calls you have to step into to comprehend the path the code takes. We're now trying to keep it as shallow as possible.
* The template method pattern worked really well because it isolated large amounts of boiler plate and flow control code and avoided copy-and-paste in implementations, ultimately this *lowered* maintenance cost, and we kept this pattern in the refactored version.
* Where template methods would have been implemented using virtual inheritance we use ```std::function```, or just plain function pointers, again emphasising the locality of state and functionality. Furthermore we use C++ lambdas...a lot...and this encourages locality *even more*. 
* We adopted a principle of *minimizing the nesting depth of code*, tearing code out of their little functions where they were much better kept right there at the call site.
* **Peer code reviews!** We use GitHub and enforcing pull requests means that more eyes can help spot at least some of the issues arising from misunderstanding of code intent. 

# In Conclusion
*Writing good, readable and maintainable, C++ code is HARD*. Refactoring often, questioning your assumptions, often, and remaining pragmatic about what solution you pick for a given job is crucial. Resist the urge to jump straight into a class hierarchy, ask yourself it it *makes sense*, and think about how runtime and static behaviour is mixed. *An architecture that only becomes manifest at runtime is very difficult to reason about*. 
<br/>
Stay safe, and good luck.
