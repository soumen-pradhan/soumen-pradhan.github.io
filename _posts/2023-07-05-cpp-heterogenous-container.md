## Heterogenous Container in C++

Dynamic languages like Python have no type constraints, hence containers like
these are quite common and idiomatic.

```py
container = []

container.append(2) # number
container.append("str") # string
container.append([5, 6, "str"]) # another container
```

### Is the same possible in C++?
tl;dr Having arbitrary types is not feasible. The Compiler needs to know some common functionality for the members. If needed, use Abstract Base Class (interface) and pointers to store the objects.

```cpp
class Hittable {
public:
    virtual bool hit() const = 0;
};

struct Sphere : public Hittable {
    virtual bool hit() const override;
};

struct Cube : public Hittable {
    virtual bool hit() const override;
};

class HittableList : public Hittable {
    std::vector<std::shared_ptr<Hittable>> objects;

public:
    HittableList& add(std::shared_ptr<Hittable> obj) {
        objects.push_back(obj);
        return *this;
    }

    virtual bool hit() const override {
        for (const auto& obj : objects) {
            // ...
        }
    }
};

// Usage
HittableList world;

Sphere s;
Cube c;

world.add(std::make_shared<Sphere>(s))
     .add(std::make_shared<Cube>(c));

```

(BTW, these are from examples in the excellent [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html))

---

However, I have been hearing about this 'concepts' thing that was released in C++20. So, I wonder if we can somehow utilize that to create a compile-time data structure. This will remove the virtual pointer to the base class (and hopefully, make it faster).

First, we need a sum type `std::variant` that contains all the classes and a way to pattern match on it `std::visit`. However, this is not very ergonomic as `std::visit` needs a Visitor object. We will come to that later.

Let's see how concepts can be used.
```cpp
template <typename Obj>
concept Hittable = requires(const Obj& obj) {
    { obj.hit() } -> std::same_as<bool>;
};
```
It says class `Obj`` must have a const method `hit` on it that produces something like bool. This is equivalent to our Base Class. Now, the derived classes can be replaced with any class that has a method of this signature, no override is needed.

```cpp
struct Sphere {
    bool hit() const;
};

struct Cube {
    bool hit() const;
};
```

Mark the container as a template.
```cpp
template <class T>
struct HittableList;
```

Now, here is a template-template. Note the use of the concept `Hittable`. This tells the compiler that instantiates a template when `HittableList` has a template parameter of type `T<A, B, C ...>` (higher kinded types?) and `A, B, C ...` all match `Hittable`.
```cpp
template <template <class...> class T, Hittable... Args1>
struct HittableList<T<Args1...>> {

    using Geometry = T<Args1...>;
    std::vector<std::shared_ptr<Geometry>> objects;

    HittableList& add(std::shared_ptr<Geometry> obj) {
        objects.push_back(obj);
        return *this;
    }

    bool hit() const {
        for (const auto& obj : objects) {
            match(*obj, [&]<Hittable H>(H& h) { /**/ });
        }
    }
};
```

Unfortunately, this does not work which would have been much clearer.
```cpp
std::vector<std::shared_ptr<Hittable>> // Incomplete type
```

Wait, how does the `match` function work?
```cpp
// https://en.cppreference.com/w/cpp/utility/variant/visit
template <class... Ts>
struct overloaded : Ts... {
    using Ts::operator()...;
};

// explicit deduction guide (not needed as of C++20)
template <class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;

template <typename Val, typename... Ts>
auto match(Val& val, Ts... ts) {
    return std::visit(overloaded { ts... }, val);
}
```
A templated struct `overloaded` is described that inherits from all template parameter types, we are especially interested in the `operator()`. Now, `std::visit` expects a Visitor that has `operator()` corresponding to all types enclosed in the `variant` object.

The function `match` constructs the overloaded object using types `ts...` (usually lambdas) in brace initialization. Note that, even though we have used a single lambda, it is generic and a new concrete lambda will be generated for each `Hittable` type enclosed in `Hittable::Geometry`.

```cpp
// Usage
using Geom = std::variant<Sphere, Cube, Cone>;

HittableList<Geom> world;

Sphere s;
Cube c;
Cone k;

world.add(std::make_shared<Geom>(s))
     .add(std::make_shared<Geom>(c))
     .add(std::make_shared<Geom>(k));

auto h = world.hit();
```
The downside is, once the container is created, we cannot add any object of a different type than those in `Geom`. The promise of better performance also seems lofty, since `std::visit` recreates a virtual pointer table.

---
So, what is the verdict?

Just use Virtual Base Classes. C++ allows multiple inheritance and that helps us with cross-cutting concerns, unlike other OOP languages. Concepts are wonderful and should be used whenever possible. This situation is however not one of them.
