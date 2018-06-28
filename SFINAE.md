# SFINAE

## Substitution Failure Is Not An Error

What does this mean?  

The loosest possible definition of SFINAE is 'template type hacks'.  
In general people understand that it's some kind of advanced C++ wizardry practised only by voodoo priests, Nth level arcane warlocks of the dark-realm and the daedra-worshipping cult of Hermaeus Mora.  
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
	
	Vector operator-(void) const
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

void function(void)
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
Specifically, the compiler failed to substitute `int` for the type variable `T` - **that is the 'substitute failure' part**.  
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
