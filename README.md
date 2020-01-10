# Move semantics "best practices"

***1. Pass-by-value, std::move over, pass-by-reference difference***

[ Taken from: https://stackoverflow.com/a/51706522 ]

```cpp
/* (0) */ 
Creature(const std::string &name) : m_name{name} { }
```

- A passed lvalue binds to name, then is copied into m_name.

- A passed rvalue binds to name, then is copied into m_name.

```cpp
/* (1) */ 
Creature(std::string name) : m_name{std::move(name)} { }
```
- A passed lvalue is copied into name, then is moved into m_name.

- A passed rvalue is moved into name, then is moved into m_name.

```cpp
/* (2) */ 
Creature(const std::string &name) : m_name{name} { }
Creature(std::string &&rname) : m_name{std::move(rname)} { }
```
- A passed lvalue binds to name, then is copied into m_name.

- A passed rvalue binds to rname, then is moved into m_name.

As move operations are usually faster than copies, ***(1)*** is better than ***(0)*** if you pass a lot of temporaries. ***(2)*** is optimal in terms of copies/moves, but requires code repetition.

The code repetition can be avoided with perfect forwarding:

```cpp
/* (3) */
template <typename T,
          std::enable_if_t<
              std::is_convertible_v<std::remove_cvref_t<T>, std::string>, 
          int> = 0
         >
Creature(T&& name) : m_name{std::forward<T>(name)} { }
```


***2. Copy elision & RVO***


Omits copy and move (since C++11) constructors, resulting in zero-copy pass-by-value semantics.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;


struct Noisy {
    Noisy() { std::cout << "constructed\n"; }
    Noisy(const Noisy&) { std::cout << "copy-constructed\n"; }
    Noisy(Noisy&&) noexcept { std::cout << "move-constructed\n"; }
    ~Noisy() { std::cout << "destructed\n"; }
};

std::vector<Noisy> f() {
	std::cout << "Into f..." << '\n';

    std::vector<Noisy> v = std::vector<Noisy>(3); // copy elision when initializing v
                                                  // from a temporary (until C++17)
                                                  // from a prvalue (since C++17)
    return v; // NRVO from v to the result object (not guaranteed, even in C++17)
}             // if optimization is disabled, the move constructor is called

void g(std::vector<Noisy> arg) {
    std::cout << "arg.size() = " << arg.size() << '\n';
}

std::vector<Noisy> foo1(std::vector<Noisy> names)
{
	//names.push_back(Noisy{});
    return names;
}

std::vector<Noisy> foo2(std::vector<Noisy> const& names) // names passed by reference
{
    std::vector<Noisy> r(names);        // and explicitly copied
    return r;
}

std::vector<Noisy> foo3(std::vector<Noisy> const& names) // names passed by reference
{
    return names;
}


std::vector<Noisy> boo()
{
    return std::vector<Noisy>(3);
}


int main()
{
	    std::vector<Noisy> v = f(); // copy elision in initialization of v
	                                // from the temporary returned by f() (until C++17)
	                                // from the prvalue f() (since C++17)
	    std::cout << "Calling g..." << '\n';
	    g(f());                     // copy elision in initialization of the parameter of g()
	                                // from the temporary returned by f() (until C++17)
	                                // from the prvalue f() (since C++17)

	    std::cout << "Calling foo1..." << '\n';
	    // get_names() is an rvalue expression; we can omit the copy!
	    std::vector<Noisy> tmp1 = foo1( boo() );
	    std::cout << "Calling foo2..." << '\n';
	    std::vector<Noisy> tmp2 = foo2( boo() );
	    std::cout << "Calling foo3..." << '\n';
	    std::vector<Noisy> tmp3 = foo3( boo() );

	    std::cout << " *** Now lvalue! *** " << '\n';
	    std::vector<Noisy> vec{3};

	    std::cout << "Calling foo1..." << '\n';
	    std::vector<Noisy> tmp4 = foo1( vec );
	    std::cout << "Calling foo2..." << '\n';
	    std::vector<Noisy> tmp5 = foo2( vec );
	    std::cout << "Calling foo3..." << '\n';
	    std::vector<Noisy> tmp6 = foo3( vec );

	    std::cout << "END" << '\n';
}
```
***Posibble output:***

Into f...

constructed

constructed

constructed

Calling g...

Into f...

constructed

constructed

constructed

arg.size() = 3

destructed

destructed

destructed

Calling foo1...

constructed

constructed

constructed

Calling foo2...

constructed

constructed

constructed

copy-constructed

copy-constructed

copy-constructed

destructed

destructed

destructed

Calling foo3...

constructed

constructed

constructed

copy-constructed

copy-constructed

copy-constructed

destructed

destructed

destructed

*** Now lvalue! *** 

constructed

constructed

constructed

Calling foo1...

copy-constructed

copy-constructed

copy-constructed

Calling foo2...

copy-constructed

copy-constructed

copy-constructed

Calling foo3...

copy-constructed

copy-constructed

copy-constructed

END

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

destructed

