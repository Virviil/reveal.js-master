Preemptive Scheduling Strikes Back: How to Avoid CPU-Hogging Goroutines


1. Introduction (2 minutes)
Good afternoon everyone! Today I'll be talking about a crucial improvement in Go's runtime that affects every Go developer, whether they realize it or not: preemptive scheduling.
Why is this topic important? In today's world of increasingly CPU-intensive applications, especially with the rise of AI-powered development tools, understanding how our code behaves under heavy computational load is more critical than ever.
Before Go 1.14, Go had a significant limitation: goroutines could monopolize CPU cores if they didn't yield control voluntarily. This created responsiveness issues, unpredictable latency spikes, and could even cause entire applications to appear frozen.
Today, I'll explain how Go fixed this problem, why it matters, and what's happening behind the scenes when your goroutines execute.




-----------> 2 
Now im working architect at root.io we are doing automatic vulnerabuility remediation for containers and we are Golang shop

I have 10+ years exp with multiple languages in multiple ecosystmes and i like languages internals very much

Today ill be happy to share with you interesting theme in which Golang is definitely a champion across other languages.

-------------->>> What are we going to talk about!!!!!!!  3 slide

We write concurrent programs to make software faster, more responsive, and better at doing multiple things at once. In the real world, applications often need to handle many tasks simultaneously—like loading data while staying responsive to user input, or processing thousands of network requests in parallel. Without concurrency, programs would have to wait for one task to finish before starting another, leading to poor performance and bad user experiences.

Modern programming languages help us manage this complexity by providing tools like threads, async/await, and coroutines. These features make it easier to express concurrent logic clearly and safely, without the pitfalls of manual thread handling. As software becomes more connected, interactive, and real-time, concurrency isn’t just an optimization—it’s a core part of how we design systems that feel fast and scale well.

Still, concurrency is inherently hard. Bugs like race conditions and deadlocks are easy to introduce and hard to detect. Languages try to simplify this with different models—some offer structured concurrency, some rely on explicit async syntax, while others like Go use lightweight goroutines and channels to make concurrent code feel more like writing sequential logic. While no approach eliminates all challenges, models that prioritize simplicity and readability can make concurrent programming far more approachable.




--------------> 4 How it's solved - 2 approaches




What's the problem? - concurrency - why do we write concurrerncy...
How it's solved - two approaces: stackless and stackfull
    Colored functions - stackless routines
        ----------> Limitations 
            Show how they are compiled into state machine
                    SOOOO
                Impossible to do anything except
                           !!!! Cooperative concurrency
    Go, Erlang - stackfull routines
        Also cooperative concurrency
            Another approach, with stop points in code
        
    Showcase - a code that works wrong in go 1.13 bevause of cooperative concurrency.

        -------> FROM 1.14
            Preemptive concurrency - because it's possible with stackfull routines
        
        How it works

    Showcase - now it works right

Final: use  Go and have fun



2. Stackful vs Stackless Execution Models (5 minutes)
Let's start with fundamentals: understanding execution models.

Colored functions

Definitions:

Stackful coroutines: Have their own call stack and can be suspended at any point, including deep within nested function calls
Stackless coroutines: Don't have their own stack and can only be suspended at certain points (usually at the top level of the function)

Comparison with other languages:
Stackful examples:

Go with goroutines: Each goroutine has its own stack (initially 2KB, grows dynamically)
Erlang with processes: Lightweight processes with their own stacks, very similar to goroutines
Ruby with Fibers: Have their own stacks for flexible execution control

Stackless examples:

Python (asyncio): Coroutines based on generators, only suspend on await
JavaScript (async/await): Only suspend on await keywords

Comparing Erlang and Go:
Both systems have lightweight execution threads with their own stacks, but with important differences:

Erlang historically had preemptive scheduling (since the 1980s)
Erlang's BEAM VM and Go runtime solve similar problems with different approaches
Erlang is more isolated (actor model), while Go is lower-level with shared memory
Go initially chose cooperative scheduling and only later moved to preemptive

Advantages of stackful model for Go:

Can block at any point without rewriting code
More natural multithreaded model for developers (no explicit yield points)
Supports blocking system calls without stopping the entire program
Enables implementation of preemptive scheduling (which was added in Go 1.14)

3. How Go's Scheduler Works (5 minutes)
Let's dive deeper into how Go knows what to execute and how it manages goroutines.
The G-P-M Model:

G (Goroutine): The actual function you want to run
P (Processor): Logical processor with a local queue of runnable goroutines
M (Machine): OS thread that executes goroutines

When you create a goroutine with go func(), the Go runtime:

Creates a new G structure with initial stack
Places it in a P's local queue or global queue
Ensures an M will execute it

Context Switching:
When switching between goroutines, the scheduler:

Saves the current goroutine's stack pointer and CPU registers
Loads the next goroutine's stack pointer and registers
Resumes execution at the new location

This is similar to how operating systems switch between threads, but much lighter weight.
Cooperative Scheduling (Pre-Go 1.14):
Before Go 1.14, scheduling was cooperative, meaning goroutines had to yield control willingly at specific points:

Channel operations
Network/file I/O
Time operations (time.Sleep)
Memory allocation (when garbage collection needs to run)
Function calls (only in certain cases)

The key problem: if a goroutine was running tight computation loops with no system calls or memory allocations, it would never yield the processor, causing other goroutines to starve.


4. The CPU-Hogging Problem (Pre-Go 1.14) (3 minutes)
Let's see a practical example of the problem before Go 1.14:
gofunc CPUHog() {
    for {
        // CPU-intensive work with no yields
        heavyComputation()
    }
}

func main() {
    // Start 4 CPU hogs
    for i := 0; i < 4; i++ {
        go CPUHog()
    }
    
    // This goroutine may never get scheduled!
    go func() {
        time.Sleep(time.Second)
        fmt.Println("Can I please run?")
    }()
    
    // Wait forever
    select {}
}
In this example, the CPUHog goroutines would completely monopolize the CPU cores, potentially preventing other goroutines from running. If you had as many CPUHog goroutines as CPU cores, your application could become unresponsive.
Real-world examples where this caused problems:

JSON parsing of large documents
Complex math/cryptographic operations
Tight loops processing large datasets
Machine learning inference

This issue wasn't just theoretical - it affected production systems and was particularly problematic in applications that mixed CPU-bound and I/O-bound work.




5. Preemptive Scheduling in Go 1.14+ (4 minutes)
Go 1.14 introduced a solution: asynchronous preemption.
How it works:

The Go runtime periodically sends SIGURG signals to threads running goroutines
The signal handler checks if preemption is needed
If so, it safely pauses the goroutine at specific safe points
The goroutine is placed back in the scheduler queue
Other goroutines get a chance to run

Let's look at the same example running on Go 1.14+:
gofunc CPUHog() {
    for {
        // Even with no yields, preemption will occur
        heavyComputation()
    }
}

func main() {
    // Start 4 CPU hogs
    for i := 0; i < 4; i++ {
        go CPUHog()
    }
    
    // This goroutine WILL get scheduled now!
    go func() {
        time.Sleep(time.Second)
        fmt.Println("I can run now!")
    }()
    
    // Wait forever
    select {}
}
Under Go 1.14+, even with CPU-hogging goroutines, all goroutines get fair access to CPU time.
Under the hood:

Uses POSIX signals on Unix systems (SIGURG)
Uses Windows mechanism on Windows
Safely interrupts at function prologues and specifically marked safe points
Minimal performance overhead in most cases
Falls back to cooperative scheduling when signals can't be used

6. Conclusion (1 minute)
Preemptive scheduling in Go represents an important evolution in the language's concurrency model, making it more robust for modern computing requirements:

You don't need to change any existing code to benefit
Your applications become more responsive, even under heavy CPU load
System-wide latency improves with fair CPU time allocation
Complex applications mixing I/O and CPU work behave more predictably

As we increasingly work with AI tools, data processing, and computation-heavy applications, these improvements become even more important.
The beauty of Go's approach is that these improvements happen "for free" when you upgrade to Go 1.14 or later versions. The Go team continues to refine the scheduler to make your applications better without requiring changes to your code.
I recommend checking the Go blog for detailed explanations of these mechanisms, and the Go runtime source code for those interested in the implementation details.
Thank you! I'm ready to take your questions.
Potential Questions and Answers
1. Are there any limitations to the new preemptive approach?
Yes, there are a few limitations:

Assembly code can still avoid preemption
FFI calls to C code (cgo) can block preemption
Some very specific edge cases with certain low-level operations
There's a small performance overhead from the signal handling

However, for the vast majority of Go applications, these limitations don't matter in practice.
2. Does preemptive scheduling affect performance?
The overhead is minimal in most cases - typically less than 1%. The Go team did extensive benchmarking to ensure the performance impact was negligible. The benefits of more predictable scheduling and responsiveness far outweigh the small overhead.
3. What other scheduler improvements have been added in recent Go versions?
Recent Go versions have introduced:

Improved goroutine stealing for better work distribution
Enhanced netpoller efficiency for network-heavy applications
Better integration with the garbage collector
Improvements to goroutine stack management
NUMA-aware scheduling in some cases

4. How can developers help the scheduler in their code?
While preemption helps a lot, you can still optimize:

For CPU-bound work, consider using worker pools with a controlled number of goroutines
Break up very long-running computations when possible
Be mindful of sync.Mutex hold times
Use buffered channels appropriately
Consider using runtime.Gosched() in extreme cases where you want to explicitly yield

5. How does Go's approach compare to other languages?

Erlang/Elixir: Has had preemptive scheduling from the beginning, but with different primitives
Java/JVM: Uses OS threads with preemption, but they're heavier weight than goroutines
Node.js: Uses event loop with cooperative scheduling
Python (asyncio): Cooperative only, must explicitly await
Rust: Offers both OS threads and async/await primitives

Go's approach strikes a nice balance between programmer ergonomics (you don't need to mark async functions) and efficiency (goroutines are lightweight).





https://dzone.com/articles/go-runtime-goroutine-preemption

https://winder.ai/cpu-hogging-in-golang/ - HERE is example of cpu hogging



stackguard - trap - и так мы всегда проверяем стек потому что он может быть оверфлоу. Чтобы не проверять два раза
g.preempt - только потом проверяем, это дорого