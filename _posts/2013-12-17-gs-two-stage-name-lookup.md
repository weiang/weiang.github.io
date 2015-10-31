---
layout: post
title: "G++'s two-stage name lookup"
description: ""
category: Reserved 
tags: [C++]
---
转自:<http://gcc.gnu.org/onlinedocs/gcc/Name-lookup.html>

The C++ standard prescribes that all names that are not dependent on template parameters are bound to their present definitions when parsing a template function or class. Only names that are dependent are looked up at the point of instantiation. For example, consider

       void foo(double);
     
       struct A {
         template <typename T>
         void f () {
           foo (1);        // 1
           int i = N;      // 2
           T t;
           t.bar();        // 3
           foo (t);        // 4
         }
     
         static const int N;
       };

Here, the names foo and N appear in a context that does not depend on the type of T. The compiler will thus require that they are defined in the context of use in the template, not only before the point of instantiation, and will here use ::foo(double) and A::N, respectively. In particular, it will convert the integer value to a double when passing it to ::foo(double).

Conversely, bar and the call to foo in the fourth marked line are used in contexts that do depend on the type of T, so they are only looked up at the point of instantiation, and you can provide declarations for them after declaring the template, but before instantiating it. In particular, if you instantiate A::f<int>, the last line will call an overloaded ::foo(int) if one was provided, even if after the declaration of struct A.

This distinction between lookup of dependent and non-dependent names is called two-stage (or dependent) name lookup. G++ implements it since version 3.4.

Two-stage name lookup sometimes leads to situations with behavior different from non-template codes. The most common is probably this:

       template <typename T> struct Base {
         int i;
       };
     
       template <typename T> struct Derived : public Base<T> {
         int get_i() { return i; }
       };
In get_i(), i is not used in a dependent context, so the compiler will look for a name declared at the enclosing namespace scope (which is the global scope here). It will not look into the base class, since that is dependent and you may declare specializations of Base even after declaring Derived, so the compiler can't really know what i would refer to. If there is no global variable i, then you will get an error message.

In order to make it clear that you want the member of the base class, you need to defer lookup until instantiation time, at which the base class is known. For this, you need to access i in a dependent context, by either using this->i (remember that this is of type Derived<T>*, so is obviously dependent), or using Base<T>::i. Alternatively, Base<T>::i might be brought into scope by a using-declaration.

Another, similar example involves calling member functions of a base class:

       template <typename T> struct Base {
           int f();
       };
     
       template <typename T> struct Derived : Base<T> {
           int g() { return f(); };
       };
Again, the call to f() is not dependent on template arguments (there are no arguments that depend on the type T, and it is also not otherwise specified that the call should be in a dependent context). Thus a global declaration of such a function must be available, since the one in the base class is not visible until instantiation time. The compiler will consequently produce the following error message:

       x.cc: In member function `int Derived<T>::g()':
       x.cc:6: error: there are no arguments to `f' that depend on a template
          parameter, so a declaration of `f' must be available
       x.cc:6: error: (if you use `-fpermissive', G++ will accept your code, but
          allowing the use of an undeclared name is deprecated)
To make the code valid either use this->f(), or Base<T>::f(). Using the -fpermissive flag will also let the compiler accept the code, by marking all function calls for which no declaration is visible at the time of definition of the template for later lookup at instantiation time, as if it were a dependent call. We do not recommend using -fpermissive to work around invalid code, and it will also only catch cases where functions in base classes are called, not where variables in base classes are used (as in the example above).

Note that some compilers (including G++ versions prior to 3.4) get these examples wrong and accept above code without an error. Those compilers do not implement two-stage name lookup correctly.

{% include JB/setup %}
