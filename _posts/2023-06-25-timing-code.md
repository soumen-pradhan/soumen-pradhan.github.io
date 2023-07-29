## Timing Code Chunks

Sometimes we need to measure the time taken by some block of code to get a
sense of the performance chokepoints. But, adding an entire library could
be too cumbersome for small cases.

From this [video by The Cherno](https://youtu.be/oEx5vGNFrLk), here is
an ingenious way to use RAII to run some code automatically at the end of
the scope.

---

#### Cpp

The Destructor runs at the end of the scope. It could be a function body
or a local scope. A function is to be supplied that handles the elapsed
time (print it or something else).

```cpp
#include <chrono>
#include <functional>

template <typename time_unit = std::chrono::milliseconds>
struct Timer {
    using clock = std::chrono::steady_clock;
    using handler = std::function<void(time_unit)>;

   private:
    clock::time_point start, end;
    handler handle;

   public:
    Timer(const handler& func) : start(clock::now()), handle(func) {}

    ~Timer() {
        end = clock::now();
        auto duration = std::chrono::duration_cast<time_unit>(end - start);
        handle(duration);
    }
};

// Usage
{
    Timer<> timer([](auto d) { fmt::print("{} ms\n", d.count()); });

    std::this_thread::sleep_for(5ms);
}
```

#### Python

Here is the same technique for Python, which utilizes Context Managers.
If you have opened a file, you may have used it.

Additionally, we can use decorators for functions.

```py
from contextlib import contextmanager
import time


@contextmanager
def timer_block():
    t1 = time.perf_counter_ns()
    yield
    t2 = time.perf_counter_ns()

    print(f"Elapsed {(t2 - t1) // 1_000_000} ms")

# Usage block
with timer_block():
    time.sleep(5)


def timer_func(fun):
    def wrapper(*args, **kwargs):
        with timer_block():
            fun(*args, **kwargs)

    return wrapper

# Usage fucntion
@timer_func
def foo(n):
    time.sleep(n)


foo(5)
```
