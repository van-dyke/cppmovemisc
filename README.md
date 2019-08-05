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

