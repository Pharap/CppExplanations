# SFINAE

## Substitution Failure Is Not An Error

What does this mean?  

The loosest possible definition of SFINAE is 'template type hacks'.  
In general people understand that it's some kind of advanced C++ wizardry practised only by voodoo priests,  
Nth level arcane warlocks of the shadow realm and the daedra-worshipping cult of Hermaeus Mora.  
Even a lot of the people who do use it don't quite understand what it means.  

But it's actually not that difficult if you've got a bit of basic template knowledge.  

## Definition by words

A substitution failure is when the compiler is unable to instantiate a template because the resulting instantiation would be invalid.  
The SFINAE rule states that if a substitution failure occurs when performing overload resolution for a function, the compiler should not signal an error.  
Instead the compiler should discard the instantiated function and remove it from the list of potential overloads.  

## Definition by example

This is quite a long example, but bear with me.  

Consider the following function, `negate`:  
```cpp
template < typename T >
typename T::result_type negate(const T & value)
{
	return -value;
}
```

It expects the type `T` to have a member type alias `result_type` and an implementation of `operator -` (that accepts a value or reference to T as a parameter and is either a member function or a free function).  

Such a class might look like this:  
```cpp
class Vector
{
public:
	using result_type = Vector;
	
public:
	int x;
	int y;
	
public:
	Vector(int x, int y) : x(x), y(y) {}
	
	Vector operator-() const
	{
		return Vector(-this->x, -this->y);
	}
};
```

`negate` can happily be used on this class because it meets the requirements of `T`:  
```cpp
Vector vector = Vector(1, 1);
auto result = negate(vector); // Equivalent to Vector result = -vector;
```

Next consider this:
```cpp
int integer = 1;
auto result = negate(integer);
```

This fails. If you're a wizard you'll have realised why.  
Essentially the compiler attempts to create this:  
```cpp
typename int::result_type negate(const int & value)
{
	return -value;
}
```

This code fails because there's no such thing as `int::result_type`.  
So let's go on to create another function that can handle `int`.  
```cpp
int negate(int value)
{
	return -value;
}
```

Now obviously, the code works:  
```cpp
int integer = 1;
auto result = negate(integer);
```

Now what if I told you that you had just witnessed SFINAE in action?  

Here's the whole code:

```cpp
template < typename T >
typename T::result_type negate(const T & value)
{
	return -value;
}

int negate(int value)
{
	return -value;
}

void function()
{
	int integer = 1;
	auto result = negate(integer);
}
```

Stop and think about that for a moment.  
What happened to the compiler-instantiated function that doesn't work?  

It's easy to dismiss this as just "the compiler decided that this was a better match",
but the compiler still hat to attempt to instantiate the other function to determine that.  
That means that the function didn't complain about the failure to compile the other function.  

The fact it didn't complain - that is SFINAE.  

The compiler attempted to instantiate the templated `negate` function.  
The instantiation failed, but the compiler still found a suitable match so it decided not to complain - **that is the SFINAE rule**.  
Specifically, the compiler failed to substitute `int` for the type variable `T` - **that is the 'substitution failure' part**.  
Following that the compiler didn't raise an error because it found something better - **that is the 'not an error' part**.  

## When does SFINAE occur?  

The cases where type substitution may occur in a template function include:  

* The return type
* The types of any function parameters
* The types of any local variables
* The types of any local type aliases
* The types used in the declaration of template parameters (e.g. default template arguments)
* All expressions used in determining the function type
* All expressions used in the template parameter declaration

## Which errors are substitution errors?  

* Attempting to create
	* an array of `void`
	* an array of references
	* an array of functions
	* an array of abstract class types
	* an array of negative size
	* an array of zero size
	* a pointer to a reference
	* a reference to `void`
	* a pointer to a member of `T`, when `T` is not a class type
	* a function that returns an array
	* a function that returns a function
	* a function with a parameter of type `void`
	* a function with a parameter that is an abstract class
	* a function with a return type that is an abstract class
* Attempting to use a member of a type where
	* the member does not exist
	* the member is not a type, but a type is required
	* the member is not a template, but a template is required
	* the member is not a variable, but a variable is required
	* the member is not a function, but a function is required
* Attempting to use an invalid expression in a `decltype` context in
	* a trailing return type
	* a template parameter type
* Attempting to perform an invalid conversion
* Attempting to reference a type member alias on a non-class, non-enumeration type

## `std::void_t`

### What is `std::void_t`

`std::void_t` is a type alias defined in `<type_traits>` as of C++17.

`std::void_t` is typically defined as thus:
```cpp
template< typename ... >
using void_t = void;
```
(Due to a defect in the standard, some earlier compliers require a level of indirection, but that detail doesn't affect usage. If you wish to know more, see [here](https://en.cppreference.com/w/cpp/types/void_t#Notes).)

It is one of many tools in SFINAE, and is particularly used in conjunction with `decltype` for testing whether or not an expression is valid.
It is useful because it effectively consumes the type detected by `decltype` without that type having to actually be used anywhere, and because it can be used in template specialisations.

It is easiest to understand by looking at some examples...

### Example 1

Here's a simple example in which the presence of a member variable `foo` is detected:
```cpp
// Base case
template< typename T, typename = std::void_t<> >
struct has_foo:
	std::false_type
{
};

// Specialisation
template< typename T >
struct has_foo< T, std::void_t< decltype(T::foo) > > :
	std::true_type
{
};
```

And some code to demonstrate the use of this 'type trait':
```cpp
struct A
{
};

struct B
{
	int foo;
};

struct C
{
	int foo()
	{
		return 0;
	}
};

struct D
{
	using foo = int;
};

// Fails for no member
static_assert(!has_foo<A>::value, "A does not have a foo member");

// Succeeds for member variable
static_assert(has_foo<B>::value, "B has a foo member");

// Fails for function
static_assert(!has_foo<C>::value, "C does not have a foo member");

// Fails for type alias
static_assert(!has_foo<D>::value, "D does not have a foo member");
```

In this case, `std::void_t` is used to detect the presence of a member named `foo`.
Note that all this particular version does is determine that the member variable `foo` exists in `T`.
This particular version does not detect functions or type aliases, which require different expressions to be used in place of `T::foo`.

Essentially this demonstrates what `std::void_t` is intended for - for use in conjunction with `decltype` in determining whether or not a particular expression is a valid expression with a discernible type.

### Example 2

Here's another more useful example which detects whether or not `T() + T()` is a valid expression, which in turn allows the user to identify whether or not there is an accessible `+` operator capable of operating on two objects of type `T`:
```cpp
// Base case
template< typename T, typename = std::void_t<> >
struct has_add :
	std::false_type
{
};

// Specialisation
template< typename T >
struct has_add< T, std::void_t<decltype(T() + T()) > > :
	std::true_type
{
};
```

This can be used as thus:
```cpp
struct A
{
};

struct B
{
	int value;
};

B operator+(B left, B right)
{
	return { left.value + right.value };
}

struct C
{
	int value;

	C operator+(C other) const
	{
		return { this->value + other.value };
	}
};

// Fails for no operator
static_assert(!has_add<A>::value, "A does not have a suitable + operator");

// Succeeds for free function operator
static_assert(has_add<B>::value, "B has a suitable + operator");

// Succeeds for member function operator
static_assert(has_add<C>::value, "C has a suitable + operator");

// Succeeds for int
static_assert(has_add<int>::value, "int has a suitable + operator");
```

Knowing whether or not two types can be added is a very useful property.
This (perhaps in conjunction with the other techniques discussed here) could be used to selectively enable a `+` operator for a type that 'wraps' other types, thus allowing a type to still be used even if it doesn't implement the full range of operations.
I.e. it allows the `+` operator to be optional rather than a requirement.

Additionally, being able to detect whether `+` is available can provide more specific error messages to an end user when combined with `static_assert`.
E.g. `static_assert(has_add<T>, "Provided type does not have a suitable + operator");`.

## `std::declval`

### What is `std::declval`?

`std::void_t` is a type alias defined in `<type_traits>` as of C++17.

`std::void_t` is typically defined as thus:
```cpp
template<class T>
typename std::add_rvalue_reference<T>::type declval() noexcept;
```
Where `std::add_rvalue_reference<T>` follows the following rules:
* If `T` is `T` then `::type` is `T &&`
* If `T` is `T &` then `::type` is `T &`
* If `T` is `T &&` then `::type` is `T &&`

`declval` is declared as a function that returns an lvalue reference (`T &`) or an rvalue reference (`T &&`).
However it is never actually defined.

This means it can be used in 'unevaluated contexts', such as `declval` (which merely infers the type of an expression without actually trying to evaluate the expression), but not in 'evaluated contexts', i.e. any context where the `declval` would be expected to produce a value, such as the initialiser for a (possibly `constexpr`) variable or the argument to a (possibly `constexpr`) function.

This is useful because it can be used in cases where a type is needed but an object of that type may not be actually constructible for various reasons (e.g. because no constructor exists for the object, or because all constructors for the object are `private` or `protected`).

### Example 1

Taking the earlier `has_add` example as a base, it can be proved that the previous definition is inadequate like so:
```cpp
struct B
{
	int value;

	// B can be constructed with an `int` argument,
	// which disables B's default constructor
	B(int value) :
		value { value }
	{
	}
};

B operator+(B left, B right)
{
	return { left.value + right.value };
}

// Fails because B has no public default constructor
static_assert(!has_add<B>::value, "B does have a suitable + operator, but has_add's defintion is inadequate");
```

This is because the expression being tested is `T() + T()`,
and when `T` is substituted with `B` that gives `B() + B()` which isn't a valid expression because `B()` attempts to call `B`'s default constructor,
which doesn't exist because the definition of the constructor `B(int)` causes `B` to no longer have an implicitly defined default constructor.

Naturally it's impossible to know ahead of time whether or not a type will have a default constructor,
and it's unreasonable to expect all types to have a default constructor because sometimes it doesn't make sense to have a default constructor.

This is precisely the problem that `std::declval` was created to solve.
The expression `std::declval<T>()` can act as a stand-in for any `T` object in an unevaluated context,
allowing an object of `T` to be introduced without any assumptions about what constructors (if any) the type `T` does or does not have.

Thus to solve this conundrum, `has_add` simply has to change the expression it tests to `std::declval<T>() + std::declval<T>()`:
```cpp
// Base case
template<typename T, typename = std::void_t<>>
struct has_add :
	std::false_type
{
};

// Specialisation
template<typename T>
// Note the change to the tested expression here:
struct has_add< T, std::void_t< decltype(std::declval<T>() + std::declval<T>()) > > :
	std::true_type
{
};
```

Here is the earlier code with all these changed incorporated:
```cpp
struct A
{
};

struct B
{
	int value;

	// B can be constructed with an `int` argument,
	// which disables B's default constructor
	B(int value) :
		value { value }
	{
	}
};

B operator+(B left, B right)
{
	return { left.value + right.value };
}

struct C
{
	int value;

	C operator+(C other) const
	{
		return { this->value + other.value };
	}
};

// Fails for no operator
static_assert(!has_add<A>::value, "A does not have a suitable + operator");

// Succeeds for free function operator, regardless of B's lack of a default constructor
static_assert(!has_add<B>::value, "B has a suitable + operator");

// Succeeds for member function operator
static_assert(has_add<C>::value, "C has a suitable + operator");

// Succeeds for int
static_assert(has_add<int>::value, "int has a suitable + operator");
```

### Example 2

Here's an example to prove that `std::declval<T>()` can work even when no public constructor is defined:
```cpp
// Base case
template< typename T, typename = std::void_t<> >
struct has_value :
	std::false_type
{
};

// Specialisation
template< typename T >
struct has_value< T, std::void_t< decltype(std::declval<T>().value) > > :
	std::true_type
{
};
// Base case
template< typename T, typename = std::void_t<> >
struct is_default_constructible :
	std::false_type
{
};

// Specialisation
template< typename T >
struct is_default_constructible< T, std::void_t< decltype(T()) > > :
	std::true_type
{
};

struct A
{
	int value;
};

struct B
{
	int value;

	// Delete B's only constructor so it can't be constructed at all
	B() = delete;
};

// Succeeds as expected
static_assert(has_value<A>::value, "A has an assignable 'value' member");
static_assert(is_default_constructible<A>::value, "A is default constructible");

// Succeeds despite B not having any kind of constructor
static_assert(has_value<B>::value, "B has an assignable 'value' member");
static_assert(!is_default_constructible<B>::value, "B is not default constructible");
```

Thus demonstrating the benefit of `std::declval`.

## `std::enable_if`

### What is `std::enable_if`

TBC...

### Example 1

TBC...

## Tag Dispatch

### What is 'tag dispatch'?

TBC...

### Example 1

TBC...