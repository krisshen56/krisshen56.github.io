---
layout: post
title:  "Notes on Effective C++"
date:   2018-12-23 08:07:00 +0800
categories: c++
---
經典的C++書籍, 雖然現在已經是modern C++的時代了, 但還是重新閱讀一次, 看看有那些東西已經改變了, 有那些東西仍然沒有記住

---

## Chapter 1 Accustoming Yourself to C++

### Item #4 Make sure that objects are initialized before they're used
1. 一般初始化data members都是使用member initialization list, 但是如果有多個constructor, 為了避免重複冗長的member initialzation list,
可以例外的在private help function使用assignment的方式來初始化這些data members.
C++11新增了[non-static member initializer][], 我想應該是可以取代掉這個private help function的功能.

2. 每個translation unit定義的non-local static object, 其初始化的順序是undefined的, 所以直接引用可能會使用到沒有初始化的object.
解決方式是類似singleton pattern, 使用static function回傳local static object的reference

[non-static member initializer]: http://www.stroustrup.com/C++11FAQ.html#member-init

---

## Chapter 2 Constructors, Destructors, and Assignment Operators

### Item #5 Know what functions C++ silently writes and calls
如果data members的type是reference or const, compiler generated的copy constructor和copy assignment operator就會無法適用.
因為reference或是const的object, 只能在object初始化時assign. 雖然書上說可以使用者自定義copy constructor/copy assignment operator解決,
但無法在copy assignment operator初始化data members, 所以必須要先destroy舊object, 再利用constructor建立新object.

基本上, 使用reference or const type做為data members本身就是一件奇怪的事, stackoverflow上的討論是reference data members可以使用C++11的
std::reference_wrapper

### Item #6 Explicitlty disallow the use of compiler-generated functions you do not want
這item已經被Modern Effectice C++的Item #11取代, 但本身的原理還是值得理解

- 要防止compiler generated functions的方式是declare them, 同時為了防止user使用, 將declarations放在private
- 但是friend or member functions還是可以invoke, 所以不要define them (產生link time error)
- 封裝成uncopyable base class, 如boost::noncopyable, user defined class再使用private inheritance繼承此class達到uncopyable(產生compile time error)

### Item #7 Declare destructors virtual in polymorphic base classes
1. base class如果是當做polymorphic base class使用(利用pointer to base class來操作object), base class就要將destructor宣告成virtual.
如果不是當成polymorphic base class使用, 則不要將destructor宣告成virtual. (ex: boost::noncopyable, STL的input_iterator_tag)
2. C++11新增了final關鍵字, 可以讓class無法被繼承
3. 最簡單的abstract class就是只有宣告pure virtual destructor的class, 但要注意的是要提供此destructor的定義, 否則會有link error.

### Item #9 Never call virtual functions during construction or destruction
事實上, 在constructor或destructor呼叫virtual functions並不會使用derived class的版本, 而是base class的版本, 原因是在這時期derived class的object部份並不存在

### Item #12 Copy all parts of an object
如果derived class實作了copy functions(不使用compiler generated的copy constructor和copy assignment operator), 要記得呼叫base class的copy functions
```c++
class Derived : public Base {
public:
    ...

    Derived(const Dervied& rhs)
        : Base(rhs), ...
    {
        ...
    }

    Derived& operator=(const Dervied& rhs)
    {
        Base::operator=(rhs);
        ...
    }

    ...
};
```

---

## Chapter 4 Designs and Declarations

### Item #20 Prefer pass-by-reference-to-const to pass-by-value
除了build-in types, STL iterators和STL function objects的參數傳遞應使用pass-by-value外, 其它types都應該使用pass-by-reference-to-const.
然而C++11有了move semantics後, 此item不再絕對正確, 請參考Modern Effective C++的Item #41

### Item #23 Prefer non-member non-friend functions to member functions
class的convenience functions應該實作為一般的functions, 因為member or friend functions會減少封裝性(增加了access privates的interfaces),
並將convenience functions和class宣告在同一個namespace但不同header files

### Item #24 Declare non-member functions when type conversions should apply to all parameters
當class的constructor不是宣告成explicit時, 其參數的object型別如果出現在function的paramters時就可以implicitly convert成class object.
書上舉的例子是rational number的class, 在和一般的numeric literal做運算時, 需要這種轉型機制. 因為numeric literal有可能在運算元兩邊,
所以對應的operator functions應實作為non-member functions

### Item #25 Consider support for non-throwing swap
一般來說, std::swap的功能已經可以滿足大部份型別的需求(只要型別支援copy operations), 但是對pimpl的class type而言, 完整的object copy並不需要,
需要的只是pointer的swap. 因此, 可以對此class type的std::swap做full specialization, 透過呼叫class的swap member function來完成pointer的交換
(因為只有member function可以access pointer member variable). 要注意的是, function template specialization要在namespace std內.
```c++
class WidgetImpl;
class Widget {
public:
    ...
    void swap(Widget& other)
    {
        using std::swap;
        swap(pImpl, other.pImpl);
    }
private:
    WidgetImpl* pImpl;
};

namespace std {

template<>
void swap(Widget& a, Widget& b)
{
    a.swap(b);
}

}
```
如果pimpl的class是class template, 就必須使用function overloading, 因為function template不支援partial specialization.
不過overloading在std namespace等於是新增了std的內容, 書上認為這是違反規定的行為, 即使可以正常編譯使用(實際上在clang 6下的確沒問題).
解決方式是利用ADL rule(or Koenig lookup), unqualified function的查找也會將其arguments所在的namespace當做查找的範圍,
將non-member function swap放在class template的namespace下
```c++
namespace WidgetStuff {
...
template<typename T>
class Widget {
    ...
};

template<typename T>
void swap(Widget<T>& a, Widget<T>& b)
{
    a.swap(b);
}
}
```
非class template的class其實也可以不用template specialization而利用ADL的方式來實作swap function, 但是書上提到許多直接限定std::swap的錯誤使用,
讓提供template specialization的方式成為一種較佳的方式.

最後, member function的swap除了提供較佳的執行較率外, non-throwing也是一般swap無法提供的(因為copy operations可能throw exceptions),
利用built-in types(ex. pointer)不會throw exceptions的特點, 將可以達到此一要求.
