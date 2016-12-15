---
layout: post
title:  "Transactional operations made easy through lists of scope guards"
date:   2016-11-01 10:00:00 +0100
comments: true
---

The scope guard idiom is a popular idiom in C++. It provides a way to execute a function at the exit of a scope with the possibility of deactivation.

A lot of descriptions of the idiom and some good implementations are floating around so I will not go into details. You can find a good presentation of Andrei Alexandrescu [here](https://channel9.msdn.com/Shows/Going+Deep/C-and-Beyond-2012-Andrei-Alexandrescu-Systematic-Error-Handling-in-C).

But for the sake of what will follow, I will still show you a basic implementation.

```c++
template<typename Funct>
class scope_guard {
public:
  static_assert(std::is_nothrow_move_constructible<Funct>::value,
                "Funct template should be nothrow move constructible.");
 
  explicit scope_guard(Funct function) noexcept :
                                    m_function(std::move(function)),
                                    m_execute(true)
  {
    static_assert(noexcept(function()), "function should be noexcept.");
  }
 
  scope_guard(const scope_guard&) = delete;
  scope_guard& operator=(const scope_guard&) = delete;
  scope_guard&& operator=(scope_guard&&) = delete;
 
  scope_guard(scope_guard&& other) noexcept :
                  m_function(std::move(other.m_function)),
                  m_execute(other.m_execute)
  {
    other.dismiss();
  }
 
  ~scope_guard() {
    if(m_execute) {
      m_function();
    }
  }
 
  void dismiss() noexcept {
    m_execute = false;
  }
 
private:
  Funct m_function;
  bool m_execute;
};

template<typename Funct>
scope_guard<Funct> make_sg(Funct function) {
  return scope_guard<Funct>(std::move(function));
}
```

The scope guard just takes a function in parameter, stores it and executes it on its destruction, except if `dismiss()` was called.

To demonstrate its usage, we will create a register of employees where each employee has an id and a name.

```c++
struct employee {
  int id;
  std::string name;
};
```

We want to have a fast access to each employee's data either with its id or its name. To do so we will use two maps, one `std::unordered_map<int, employee>` and one `std::map<std::string, employee>`. We could use a vector if the IDs are contigous and we should avoid to store two copies of the employee object and use some references. Some copies operations could also be avoided in the next few snippets, but let's keep it simple for the sake of our example.

With these two maps, we need to keep an invariant for the class. If an employee is in the first map, it should also be in the second map.

When we add an employee to the register, we insert the employee in the first map. If everything goes well, we create a rollback operation that would remove the employee from the map. So now, if the insert into the second map throws an exception, the employee will be automatically removed from the first map, keeping our invariant safe. If everything goes well, we deactivate the scope guard as we don't want the scope guard to call the rollback operation.

```c++
class employee_register {
public:
  void add(employee empl) {
    auto it = m_id_map.insert({empl.id, empl}).first;
    auto sg = make_sg([&]() noexcept {m_id_map.erase(it);});

    m_name_map.insert({empl.name, empl});

    sg.dismiss();
  }
 
private:
  std::unordered_map<int, employee> m_id_map;
  std::map<std::string, employee> m_name_map;
};
```

All this work fine but let's say that for some reasons we would like to be able to insert a batch of emloyees to the register and it should be an atomic operation. Either they are all added to the register or none of them are.

A simple non-atomic way to do so will look like this.

```c++
void add_batch(std::initializer_list<employee> employees) {
  for(const auto& empl: employees) {
    m_id_map.insert({empl.id, empl});
    m_name_map.insert({empl.name, empl});
  }
}
```

How could we use the concept of scope guard to make this atomic? We will need a list of operations to do the rollback, a list of scope guards.

To do so we will use some type-erasure to store the rollback operations with `std::function<void()>` and store each of them in a vector. On destruction of our `scope_guard_list`, we will execute each stored function.

```c++
class scope_guard_list {
public:
  scope_guard_list() noexcept : m_execute(true) {
  }
    
  explicit scope_guard_list_impl(std::size_t size) {
    m_functions.reserve(size);
  }
 
  scope_guard_list(const scope_guard_list&) = delete;
  scope_guard_list& operator=(const scope_guard_list&) = delete;
  scope_guard_list& operator=(scope_guard_list&&) = delete;
 
  scope_guard_list(scope_guard_list&& other) noexcept :
                    m_functions(std::move(other.m_functions)),
                    m_execute(other.m_execute)
  {
    other.dismiss();
    other.m_functions.clear();
  }
 
  ~scope_guard_list() {
    if(m_execute) {
      for(auto it = m_functions.rbegin();
          it != m_functions.rend();
          ++it)
      {
        (*it)();
      }
      
      dismiss();
    }
  }
 
  void dismiss() noexcept {
    m_execute = false;
  }
 
private:
  std::vector<std::function<void()>> m_functions;
  bool m_execute;
};
```

We are still missing a way to add a function to our `scope_guard_list`. We could use a simple add function like the following.

```c++
template<typename Funct>
void add(Funct function) {
    static_assert(noexcept(function()), "function should be noexcept.");
    
    m_functions.push_back(std::move(function));
}
```

But we have two problems. The `m_functions.push_back` operation can throw an exception if it needs to do some allocation (in case we didn't reserve enough space in the vector). Also, the construction of `std::function<void()>` could do the same if the lambda is big enough to hinder the small size optimization of `std::function` or if the implementation doesn't do this optimization. In these cases, the rollback function will not be in the list even if the operation it should rollback has already been executed.

Instead we will use another way which ties together the function we want to execute and its rollback.

```c++
template<typename ExecFunct, typename RollbackFunct>
auto exec(ExecFunct execute_function, RollbackFunct rollback_function)
    -> decltype(execute_function())
{
  static_assert(noexcept(rollback_function()),
                "rollback_function should be noexcept.");

  m_functions.push_back(std::move(rollback_function));

  try {
    return execute_function();
  }
  catch(...) {
    m_functions.pop_back();
    throw;
  }
}
```

We first create the `std::function<void()>` and add it to the vector. If anything goes wrong, `execute_function` was still not executed, so we are safe.

If the `execute_function` fails, we don't want to execute the associated rollback operation, so we remove it from the vector. Otherwise, we are sure the function was well executed and that the rollback operation is in the vector.

This way we also force the programmer to tie each operation and its rollback together which may prevent some mistakes and offer some implicit documentation.

With all that, let's revise our `add_batch` of our `employee_register`.


```c++
void add_batch(std::initializer_list<employee> employees) {
  scope_guard_list guards;
 
  for(const auto& empl: employees) {
    guards.exec([&](){m_id_map.insert({empl.id, empl});},
                [&]() noexcept {m_id_map.erase(empl.id);});
                
    guards.exec([&](){m_name_map.insert({empl.name, empl});},
                [&]() noexcept {m_name_map.erase(empl.name);});
  }

  guards.dismiss();
}
```

With some macro we could simplify this to the following.

```c++
#define sgl_exec(sgl, execute_function, rollback_function) \
                 sgl.exec([&]()execute_function, \
                          [&]() noexcept rollback_function)
                 
void add_batch(std::initializer_list<employee> employees) {
  scope_guard_list guards;

  for(const auto& empl: employees) {
    sgl_exec(guards, {m_id_map.insert({empl.id, empl});},
                     {m_id_map.erase(empl.id);});
    sgl_exec(guards, {m_name_map.insert({empl.name, empl});},
                     {m_name_map.erase(empl.name);});
  }

  guards.dismiss();
}
```

But take care, as the macro hides the capture of the reference and may thus not be wise to use depending on your preferences.

In the end, the `scope_guard_list` allows us to easily put transactional operations together without worrying about the exit path our code will take.

A more complete implementation of `scope_guard` and `scope_guard_list` can be found on [Github](https://github.com/Tessil/utils) with two other classes: `exception_guard` and `exception_guard_list`. For these two other classes, the rollback functions are executed only if we exit the scope through an exception so we don't have to call the `dismiss()` function during a normal return.

