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

---

## Chapter 5 Implementations

### Item #26 Postpone variable definitions as long as possible

遵守這個item的守則讓人很直覺的把迴圈裡的變數定義在迴圈的大括號區塊內, 但書上提到了如果變數的assignment operator的cost比constructor +
destructor少很多且又有效能上的考量的話, 就可以把此變數放在迴圈之外.

### Item #27 Minimize casting

雖然C-style的casting仍然在C++有支援, 但coding時還是以C++的casting語法為主:
- const_cast: 將const object轉成non-const object
- dynamic_cast: 唯一C-style casting不支援的轉型, 將pointer to base class object轉成pointer to derived class object. 對效能有極度影響.
- reinterpret_case: low-level轉型, 例如pointer轉成int, 具不可移植性.
- static_cast: 其它強制轉型, 例如int轉double, non-const object轉const object.
  static_cast也可用於pointer-to-base到pointer-to-derived的轉型, 和dynamic_cast相較, dynamic_cast轉失敗會回傳nullptr, 但static_cast不會.

書上提到, pb可能不等於&d, 會根據C++編譯器實作而有所不同, 但在Linux下的GCC和Clang試了結果都一樣, 即使加了virtual functions在class裡頭.
```c++
class Base { ... };
class Derived : public Base { ... };
Derived d;
Base *pb = &d;
```
Stackoverflow上也有篇[文章](https://stackoverflow.com/questions/14776469/more-than-1-address-for-derived-class-object)詢問這個問題

### Item #29 Strive for exception-safe code

Exception-safe的條件有二:
1. Leak no resources: 遵守RAII, 將resource以object封裝, 利用constructor和destructor來確保沒有resource leak的問題.
2. Don't allow data structures to become corrupted: 當發生exceptions時, object內部的data仍是可正常使用的狀態.

只要能提供以下其中一種gurantee的function都可以稱之為exception safe function:
1. the basic guarantee: 當exceptions發生時, 只保證object內的data是正常可使用的狀態, 至於是什麼狀態, 則無法直接得知
2. the strong guarantee: object內的data狀態只會有二種可能. 有exceptions發生時, 狀態回到calling function前, 沒有exceptions發生則是成功執行function後的狀態.
   通常使用copy-and-swap的技巧來達成.
3. the nothrow guarantee: 是最難達到的一種exception safe, 只使用built-in types和其它nothrow operations來達成no exception.

exception specification的語法在C++11有變化, 請參考Modern Effective C++的Item#14

### Item #30 Understand the ins and outs of inlining

inline只是向compiler提出的一種要求, 至於compiler要不要將function inline則是取決於:
1. function的複雜度: 有迴圈或遞迴的就不會inline
2. 是不是virtual function: runtime時才能決定要呼叫那個virtual function, 所以無法在compile time時在call site inline function

當使用pointer to function的方式呼叫function時, 該function也不會在call site做inline, 原因也是runtime time才能決定實際要呼叫的function
```c++
inline void f() { ... }
void (*pf)() = f; // take address of function, compiler will generate
                  // outlined function body
...
pf();             // will not inline f here
```

### Item #31 Minimize compilaton dependencies between files

要減少compilation dependencies的準則就是盡可能的把dependencies on definitions改成dependencies on declarations
- 使用handle class: 也就是所謂的pimpl pattern
- 使用interface class: 利用pure virtual class定義出提供的member function interface讓client使用,
client利用factory function來取得concrete class的object pointer或reference

一般簡單的應用準則就是使用forward declaration, 利用pointer or reference to object來操作object提供的operations.
特別的是, function declaration的paramters or return type是class type時, 即使使用forward declaration,
並無限制此class type要是pointer or reference.

---

## Chapter 6 Inheritance and Object-Oriented Design

### Item #33 Avoid hiding inherited names

1. base class如果有overloaded functions, derived class即使只有redefine其中的virtual function, 其它同名的functions一樣會被hiding,
解決方式是在derived class的定義中使用using declarations來解決visibility問題.
2. 書上提到對於public inheritance要特別小心這種hiding, 因為會破壞is-a relationship的原則

### Item #34 Differentiate between inheritance of interface and inheritance of implementation

使用public inheritance基本上就是繼承了base class所定義的function interface, 但是function implementation的繼承與否則和function
的宣告有關:
- pure virtual function: 只繼承了function interface
- simple virtual function: 繼承了function interface, 並提供了default implementation.
  為了避免之後新增的derived class發生不該繼承default implementation的非預期行為, 應將base class的inteface function宣告為pure virtual function
- non-virtual function: 同時繼承了function interface和implementation

如果base class有pure virtual function, 那麼derived class就必須在定義中宣告此virtual function並提供實作, 即使base class的pure virtual function
有提供實作.

