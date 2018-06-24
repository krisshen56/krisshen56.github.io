---
layout: post
title:  "Notes on Effective Modern C++"
date:   2018-05-27 17:00:00 +0800
categories: c++11
---
這本書買了也快二年了, 書的內容大概也看過一半了, 但是對於內容的熟悉度還是相當貧乏. 主要原因是工作上完全用不上C++, 加上自己閒暇時也沒有在寫C++程式, 所以總是看完後沒多久記憶力就歸零了.

本篇文章是個人的讀書筆記, 讓自己可以快速抓回重點, 畢竟重看原文書的內容還是會花一點時間.

---

## Chapter 1 Deducing Types

### Item #1
template type deduction的三種型式, 以下面的function template來做討論.
```c++
template<typename T>
void f(ParamType param);

f(expr)
```
ParamType是T和const, *(pointer), &/&&(reference)和volatile的各種可能組合

- Case 1: ParamType是pointer或reference
  expr如果有const, const會被保留, 否則傳進去f後, 會被修改而失去了const的意義
  expr如果有&, &不保留, 因為&只是alias
- Case 2: ParamType是universal reference
  expr如果是l-value, T就會被推導成有&的type(根據reference collaption)
  expr如果是r-value, 規則同Case 1 
- Case 3: ParamType不是pointer也不是reference
  pass by value的情況, 所以expr的最外層const可以被去掉 
  expr如果有&, &不保留, 因為&只是alias

套用以上rule後再做expr和ParamType的pattern matching推導出T

當expr是array或function時, 會deacy成pointer, 除了在ParamType有reference的情況例外

### Item #2
auto type deduction可以與template type deduction做對應, auto對應到T, param前的type宣告就是ParamType
```c++
auto param = expr;        // ParamType = auto = T
const auto& param = expr; // ParamType = const auto& = const T&
```
例外的情況是expr是braced initializer
```c++
auto x = 27;   // the type of x is int
auto y = {27}; // the type of y is std::initializer_list<int>

template<typename T>
void f(T param);
f({27});       // error, cannot deduce
```

要注意的是, auto如果出現在function的return type或lamdba表達式中的參數, 也是只支援template type deduction,
而非一般的auto type deduction

### Item #3
- decltype(expr)得到的type就是expr的真實type
- decltype在C++11是和auto搭配成為trailing return type語法
- C++11只允許lambda表達式中單一statement的return type deduction, C++14放寬到function並允許multiple statements
- decltype(auto)除了可以使用在function return type外, 也可以使用在一般的type宣告
```c++
// cw is const Widget&
auto w1 = cw;           // w1 is Widget
decltype(auto) w2 = cw; // w2 is const Widget&
```
- decltype不能取得真正的type例外情況: expr不是單純的variable name, 而是expr時, 會強制變成T&
```c++
int x = 0;
decltype(x) y;   // y is int
decltype((x)) z; // z is int&
```

### Item #4
如何得知推導出來的type:
- Compile time可以使用undefined template function, 從compiler吐出來的error訊息得知
```c++
template<typename T>
class TD;
TD<decltype(x)> xType;
```
- Runtime則必須使用boost::typeindex, 因為std::type_info對cvr處理的限制

---

## Chapter 2 auto

### Item #5
使用auto的好處
- 避免uninitialized variable問題
- 較少的typing
- 較佳的執行效率和記憶體使用量(以lambda closure轉換成std::function為例)
- 避免錯誤的type宣告帶來不必要的轉型, 造成程式行為怪異和效能損失

### Item #6
由於"invisible" proxy type的關係, auto有可能會推導出錯誤的type, 造成意料外的結果

解決方式是使用the explicitly type initializer idiom強制auto推導出需要的type
```c++
auto var = static_cast<T>(expr);  // T is the type you want
```

---

## Chapter 3 Moving to Modern C++

### Item #7
Braced initialization的好處
- 在任何地方都可以用來初始化物件, =和()的方式在某些地方就有些限制

  例如:
  - 在class non-static data member的預設值就不能使用()
  - 在non-copyable object中就不能使用=
- 防止narrowing conversion和vexing parse
- 可以直接初始化container的內容

Braced initialization的缺點是, 當type的constructor有多個, 且其中一個constructor
有使用std::initializer_list當做參數的話, 不論如何, braced initialization都會以
std::initializer_list為優先對象
```c++
std::vector<int> v1(10, 20); // create 10 elements with value 20
std::vector<int> v2{10, 20}; // create 2 elements 10, 20
```
總結來說, 似乎是使用**{}**來初始化物件是最佳解(雖然作者對那一派沒有特別偏好). 只要注意
物件的constructor是否有std::initializer_list參數類型的constructor, 必要時用**()**來避免
上述的情形發生

### Item #8
0和NULL常常都會當做null pointer來使用, 但實際上0的type是int, NULL的type依實作可能是int或是
long. 如果遇到以下的case, 就可能會有ambiguous的情況產生
```c++
void f(int);
void f(bool);
void f(void*);

f(0);       // calls f(int)
f(NULL);    // might be ambigiuos if NULL is long
f(nullptr); // calls f(void*)
```
nullptr的type是std::nullptr_t, 可以轉換成任意的pointer type. 除了可以避免上述問題外, 也可以
讓function template在type deduction時可以推導出pointer type而不是integral type(參見書上的例子)

### Item #9
alias declaration可以完全取代typedef, 而且可以和template搭配形成alias template, 好處是可以在TMP
時省去許多::type, typename的使用
```c++
// same as typedef
using UPtrMapSS = std::unique_ptr<std::unordered_map<std::string, std::string>>; 
// seems to be better than typedef
using FP = void (*)(int, const std::string);
// alias template for C++11 to mimic C++14 type traits
template<typename T>
using remove_const_t = typename std::remove_const<T>::type;
```

### Item #10
C++98的unscoped enum定義的name,其scope是和Color同一層
```c++
enum Color { black, white, red }; // unscoped enum
auto white = false;               // error
```
C++11的scoped enum則是如同class一樣,局限在class scope
```c++
enum class Color { black, white, red }; // scoped enum
auto white = false;                     // fine
```

scoped enum是strongly typed,不像unscoped enum可以implicitly轉換成intergral type(接著再轉成float...)

scoped enum預設是可以forward declaration的, unscoped enum在C++11要達成forward declaration需要指定
underlying type
```c++
enum class Status;         // default underlying type is int
enum Color: std::uint8_t;  // underlying type is std::uint8_t
```

unscoped enum比較有用的地方在於std::tuple中指定特定的field(因為implicit conversion的好處)
```c++
using UserInfo = std::tuple<std::string, std::string, std::size_t>;
UserInfo uInfo;
enum UserInfoFields { uiName, uiEmail, uiReputation };
auto val = std::get<uiEmail>(uInfo);
```

### Item #11
C++98避免class copyable的做法是declare copy constructor和copy assignment operator在private
並且不提供實作. 但是如果有member function或是friend class/function不小心呼叫到, 在link time才會有
error產生.

C++11的做法是在public標記copy constructor和copy assignment operator為delete, deleted functions在public的
原因是C++檢查function的accessibility在delete status之前.
```c++
class NonCopyable {
public:
    ...
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
    ...
};
```
delete也可以使用在non-member function和function template instantiation來禁止某些overloaded function的使用

### Item #12
overriding是derived class實作base class的virtual function以達成polymorphism的方式, 但是常常會發生沒有正確
override的情況, 而且compiler無法去偵測到這樣的錯誤, 因為通常這種錯誤是符合語法的.

C++11提供了override關鍵字, 可以使用在derived class要override的function declaration最後, compiler會幫忙抓出
無法正確override的error.
```c++
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};

class Derived : public Base {
    virtual void mf1() override;               // error: constness different
    virtual void mf2(unsigned int x) override; // error: parameter different
    virtual void mf3() && override;            // error: reference qualifier different
    void mf4() const override;                 // error: no virtual specified in base class
};
```

C++11的member function可以指定reference qualifier以適用不同的*this object
```c++
class Widget {
public:
    ...
    void doWork() &;  // for *this as l-value
    void doWork() &&; // for *this as r-value
    ...
};
```

### Item #13
C++98在const_iterator的支援不夠完全, 在有些情況的使用上仍需使用iterator. 這部份在C++11對container class
提供了cbegin, cend等的member function已經修正. 此外, C++14也提供了non member版本的cbegin, cend等的function
讓實作template function更方便.

### Item #14
