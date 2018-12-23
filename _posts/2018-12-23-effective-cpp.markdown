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

[non-static member initializer]: http://www.stroustrup.com/C++11FAQ.html#member-init
2. 每個translation unit定義的non-local static object, 其初始化的順序是undefined的, 所以直接引用可能會使用到沒有初始化的object.
解決方式是類似singleton pattern, 使用static function回傳local static object的reference

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
