
>"Poppy smart. Poppy good. Poppy help master find bug." said Poppy, contemplating its existence.

Poppy is a stack tracing and performance measurement library for C++. Here is when it is useful:

- You send in your bug-free ("Honest!") application for beta-testing and the testers start reporting hard-to-reproduce crashes. Where in the code did those crashes occur?

- You need to know the parameter values for the functions on the stack trace leading to the crash, not just the function names. You also need to know their local variable values, loop counters, etc.

- You publish your application to end users, without debug symbols. You need to log the exact stack traces of any crashes for later analysis.

- You need to measure the performance of the individual functions in your code and find the bottlenecks.

- Your development environment doesn't support a visual debugger or debug symbols. During an unhandled exception or a segfault crash you need to know where the crash happened.

##Integration

Just include the PoppyDebugTools.h header in your source files. In your global exception catcher or signal handler function you can query the stack trace that lead to the crash like so:

```
std::string stackTrace = Stack::GetTraceString();
```

(Read below for examples on exception handling and signal catching.)

Then sprinkle the STACK~ macros over your code. Here are the macros themselves.

##STACK macro

The `STACK` macro is the simplest of the lot. Just place it at the start of each function you want stack-traced. For example:

```
void func1(){
    STACK
    func2();
    func3();
}

void func2(){
    STACK

    //induce a segfault crash
    int* x = (int*) -1;
    int y = *x;
}

void func3(){
    STACK
    throw new std::runtime_error("error");
}
```

For the above example you will get something like the following for the func2 crash:

```
function func1 at FileOfFunc1.cpp:123

while INSIDE the scope of: function func2 at FileOfFunc2.cpp:456
```

with the filenames and line numbers of those functions shown as appropriate. Once you fix the `func2` crash, you will get the following for the `func3` crash:

```
function func1 at FileOfFunc1.cpp:123

while INSIDE the scope of: function func3 at FileOfFunc3.cpp:789
```

We are talking about functions here, but those can just as well be class methods too - the same principles apply.

You don't have to mark every single function with a `STACK` macro. If you skip some function it will simply be missing in the resulting stack trace - you are free to decide if that suits your needs. For example:

```
void func1(){
    STACK
    func2();
}

void func2(){
    //no STACK macro? It is perfectly OK!
    func3();
}

void func3(){
    STACK
    throw new std::runtime_error("error");
}
```

The stack trace will be:

```
function func1 at FileOfFunc1.cpp:123

while INSIDE the scope of: function func3 at FileOfFunc3.cpp:789
```

##STACK_BLOCK macro

The `STACK_BLOCK` macro can be used to give names to the control blocks within your functions, like loops and "if" blocks. For example:

```
void func1(){
    STACK

    int r = rand();

    if(r%2){
        STACK_BLOCK(crashy if clause)

        //induce a segfault crash
        int* x = (int*) -1;
        int y = *x;
    }

    for(int i = 0; i < 10 ; ++i){
        STACK_BLOCK(crashy loop)

        if(r%3){
            STACK_BLOCK(nested if)

            throw new std::runtime_error("error");

        }
    }
}
```

The stack trace for the first crash will be:

```
function func1 at SomeFile.cpp:123

while INSIDE the scope of: block "crashy if clause"
```

If that passes, the trace for the second crash will be:

```
function func1 at SomeFile.cpp:123

block "crashy loop"

while INSIDE the scope of: block "nested if"
```

Note how the `STACK_BLOCK` acts like an additional named stack frame, used to further isolate the location of the crash within the function. A `STACK_BLOCK` is valid until the next closing } brace.

You are not required to have a `STACK` macro in the function where a `STACK_BLOCK` is placed - but it is generally a good idea, as it will show you the function name in the stack trace and not just the block name.

You can also have several blocks inside the same scope, like this:

```
void func1(){

    STACK

    //code...

    STACK_BLOCK(first block)

    //more code...

    STACK_BLOCK(second block)

    throw new std::runtime_error("error");

}
```

yielding the following trace:

```
function func1 at SomeFile.cpp:123

block "first block"

while INSIDE the scope of: block "second block"
```

##STACK_SECTION macro

The above `STACK_BLOCK` example was nice, but you didn't really need to see that you are in "first block" - it is obviously above the crash location and it isn't bringing any useful information. So to delimit long functions you might use the `STACK_SECTION` macro instead. It is similar to `STACK_BLOCK`, with the difference that it pops/destroys any previous `STACK_SECTION`s up to the last `STACK` or `STACK_BLOCK`. Let's rewrite the above example:

```
void func(){

    STACK

    //code...

    STACK_SECTION(first)

    //more code...

    STACK_SECTION(second)

    throw new std::runtime_error("error");

}
```

The stack trace will be:

```
function func at SomeFile.cpp:123

while INSIDE the scope of: section "second"
```

Note how the first section was popped and is not present in the stack trace. This is useful for delimiting long functions.

Keep in mind that sections are meant to be placed in flat stretches of code, not in nested scopes. If you for example place a `STACK_SECTION` in a loop inside your function, without a `STACK_BLOCK` for the loop, every loop iteration will destroy the previously generated section and start a new one, thus invalidating the usefulness of the macro.

##STACK_VAL macro

Last but not least is the `STACK_VAL` macro. It is used to place values inside the generated stack trace, like parameter values, loop counter values or expression results. For example:

```
void func(string s){
    STACK
    STACK_VAL(argument, s)

    for(int i = 0; i < 10; ++i){
        STACK_BLOCK(for loop)

        //toString is some function converting int to string
        STACK_VAL(index, toString(i))

        if(i == 5){
            throw new std::runtime_error("error");

        }
    }
}
```

The crash stack trace after calling `func("epic fail")` will be:

```
function func at SomeFile.cpp:123

argument = epic fail

block "for loop"

while INSIDE the scope of: index = 5
```

The first parameter to `STACK_VAL` is the name to give to the output value. The second parameter is an `std::string` expression encoding the value itself. You can thus output any object type, as long as you convert it to a string somehow.
As you can see from the output above, `STACK_VAL` has similar scoping rules as `STACK_BLOCK` - it is alive until the next closing } bracket.
When popping older sections, `STACK_SECTION` leaves any `STACK_VAL`s above it intact.

`STACK_VAL` is a powerful way to log the exact data values that caused the crash, and not just the crash location. Debug symbols would have been useless in this.

##Exit markers

So you sprinkled your code with STACK~ macros, but missed a few spots. Or there are parts of the code that you cannot modify, like third-party libraries. What the stack trace will look like when one of those pieces of code crashes? Take a look at this example:

```
void func1(){
    STACK
    func2();
}

void func2(){
    //no STACK macro here!
    func3();

    //induce a segfault crash
    int* x = (int*) -1;
    int y = *x;
}


void func3(){
    STACK

    //some harmless code here
}
```

The stack trace will be:

```
function func1 at FileOfFunc1.cpp:123

after EXITING the scope of: function func3 at FileOfFunc3.cpp:456
```

Note how Poppy can't know where exactly the crash happened because func2 is not STACK-marked. But it knows what was the last piece of code that worked for sure - func3. And it says the crash happened after that.

Here is another example:

```
void func(){
    STACK //properly marked

    for(int i = 0; i < 10; ++i){
        STACK_BLOCK(for loop)

        //some harmless code here
    }

    //induce a segfault crash
    int* x = (int*) -1;
    int y = *x;
}
```

The stack trace will be:

```
function func at SomeFile.cpp:123

after EXITING the scope of: block "for loop"
```

Note how the stack trace is further narrowed down to point to the area of the code after the last scope that exited correctly.

##A note on handling exceptions and segfault crashes

If your application is a stand-alone executable, with a "main" function, you just need to surround the main with a try/catch block and get the stack trace in the catch:

```
void main(){
    try{
        //application code goes here
    }
    catch(...){//catch everything
        std::string stackTrace = Stack:: GetTraceString();
        //you can print or log the stack trace here
    }
}
```

If your code has multiple entry points, enclose each of them in try/catch blocks like above. Examples of this are dynamically linked libraries, Android native code invoked via JNI or iOS C++ code invoked by Objective-C.

You would probably also want to handle signals, in order to debug crashes from null dereferencing, dangling pointer accesses, buffer overflows, etc. This is done like so:

```
#include <signal.h>

void signal_handler(int signal, siginfo_t *info, void *reserved){
    std::string stackTrace = Stack:: GetTraceString();
    //you can print or log the stack trace here
}

struct sigaction handler;
handler.sa_mask = 0;
handler.sa_restorer = 0;
handler.sa_sigaction = signal_handler;
handler.sa_flags = SA_RESETHAND;
#define CATCHSIG(X) sigaction(X, &handler, &old_sa[X])
CATCHSIG(SIGILL);
CATCHSIG(SIGABRT);
CATCHSIG(SIGBUS);
CATCHSIG(SIGFPE);
CATCHSIG(SIGSEGV);
CATCHSIG(SIGSTKFLT);
CATCHSIG(SIGPIPE);
```

##Performance measurement

Once you've placed the STACK~ macros, they can also be used for measuring the run times of the functions, blocks and sections. 
First, set the `PERFORMANCE_COUNTING_ENABLED` macro in PoppyDebugTools.h to 1 to enable performance measurement. Now run the application and call one of the following static functions regularly. They both have similar output but for a different time frame.

1)
```
std::string perfReportForLastInterval = CallTree::GetPerformanceReportForLastInterval();
```

This function extracts a report for the performance of the call trees during the previous interval. By default the interval is 5 seconds. You can show the report on-screen to see the current performance bottlenecks. This is useful for performance-tuning real-time applications such as games.

2) 
```
std::string perfReportSinceLaunch = CallTree::GetPerformanceReportSinceLaunch();
```

This function extracts a report for the performance of the call trees since the launch of the application. It is useful for non-real-time processes like photorealistic 3D rendering and I/O-heavy operations.

Here is what the performance reports look like:

```
------Call Tree #1------------
function Update at Game.cpp:456: 100%, 25ms
    function UpdatePhysics at Game.cpp:678: 60%, 15ms
    function UpdateAI at Game.cpp:567: 40%, 10ms
------Call Tree #2------------
function Render at Game.cpp:123: 100%, 15ms
   section "color pass": 60%, 9ms
      function PerformGLRender at Game.cpp:345: 90%, 8.1ms
      function SetGLState at Game.cpp:234: 10%, 0.9ms
   section "shadow pass": 40%, 6ms
      function PerformGLRender at Game.cpp:345: 80%, 4.8ms
      function SetGLState at Game.cpp:234: 20%, 1.2ms
```

A call tree is a tree of STACK~ marked nested stack frames - functions, blocks and sections. The root is the earliest STACK~ marked frame of this tree - usually an entry point into your code. Its children are the STACK~ marked frames that it calls and so on. Each call tree's root frame takes 100% of its execution time. Child calls split this percentage between themselves, which are further split by their own children and so on down the tree. In the report child calls are shown with indentation relative to their parent. 
In the above example's 1st call tree, the Update function took 25 milliseconds, which is 100% of the execution time of this call tree. Its UpdatePhysics child function took 15ms from that, which is 60% of the total for this tree. The UpdateAI child function took 10ms, which is 40% of the total. For the 2nd call tree, the Render function took 15ms to execute, which is 100% of this call tree. This is further subdivided among its sections, which are themselves subdivided too.

Note that child frames are sorted by execution time within their parent, not by their order of execution. The call trees are also sorted by execution time. So the performance bottlenecks are generally at the top of the report.

A single function can potentially be called by multiple parents. It will then appear several times in the reports, listed by its execution time within each of its parents. This is demonstrated by the PerformGLRender and SetGLState functions above.

##Configuration

You can use the following macros inside PoppyDebugTools.h to configure Poppy's behaviour.

The `STACK_TRACING_ENABLED` macro is a global on/off switch for Poppy. If set to 0, Poppy will not insert any additional instructions into your executable and all of the STACK~ macros are made empty. Both stack tracing and performance measurement are disabled. Use this if you don't intend to use Poppy any more. The default value is 1 (i.e. enabled).

The `PERFORMANCE_COUNTING_ENABLED` macro switches only performance measurement on or off. The default is 0, meaning performance measurement is disabled. This is because it adds some extra overhead to the execution times of the STACK~ macros. Set this to 1 only when measuring performance and set it back to 0 when releasing your application.

The `PERFORMANCE_COUNTING_INTERVAL_MS` macro sets the length in milliseconds for the performance measurement interval. When using the `CallTree::GetPerformanceReportForLastInterval()` method, the execution times are shown for the previous such interval. The default value is 5000 milliseconds (5 seconds).


##Footprint

Poppy has been explicitly optimized for minimal performance footprint. Depending on the application, it may be suitable to keep it switched on even in release builds in order to gather crash stack traces in production.
Never-the-less, if has a larger overhead when used for performance measurement. It is recommended that you set `PERFORMANCE_COUNTING_ENABLED` to 0 for release builds - measurement is not needed there anyway.


##Thread safety

Poppy is currently NOT thread-safe. If the functions you mark with the STACK~ macros will be called from several threads, make sure the threads enter the marked code trees in turns, not simultaneously. No two threads should execute STACK~ marked code at the same time. The reporting methods `Stack::GetTraceString()` and `CallTree::GetPerformanceReport~()` are not thread-safe with the macros as well. As a workaround, you can use macros that turn on and off Poppy in source files depending on which thread the file is executed from (if only one thread executes it). 

##Platform support

Poppy has been tested and works on iOS and Android. It should work on any Unix-like system. It doesn't yet support Windows and hasn't been tested on the Microsoft compilers. Porting to Windows would require implementing the function CurrentTime to return the number of milliseconds passed since some fixed point in time, like the start of the epoch (1970) or the application start time. The GCC-specific preprocessor macros \_\_FUNCTION__, \_\_LINE__, \_\_FILE__ and \_\_COUNTER__ may also need porting. Code contributions are welcome!

##Future work

Windows support. 

Thread-safety would be very nice to have, but without sacrificing the portability, ease of integration and performance of the library.

Reverse call trees are not supported yet. That is, you can't list the parents that called a given function in the performance reports. Right now if a function is called from multiple parents, it might be hard to recognize it as a bottleneck as its execution times are split among its parents. 



