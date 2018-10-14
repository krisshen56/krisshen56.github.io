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
看不太懂的一個item, 作者寫了很多內容, 但看完了還是不知道到底要不要宣告function為noexcept

### Item #15
constexpr修飾variable時, 表示其值是const且在compilation time已知

constexpr修飾function時, 如果其傳入的參數是constexpr object, 則其回傳的運算結果也是constexpr;如果傳入的參數的值是
runtime才能決定, 則視為一般function

C++11在constexpr function有些限制, C++14則解除了這些限制
1. return值不能是void(因為不是literal types)
2. 只允許單一return statement(但可以利用三元運算子?:達到if else的效果)
3. member function隱含const的含義

### Item #16
重點就如同標題所寫的, 除非確保const member function不會被multiple threads同時引用, 請確保其具有thread safety

其它談到的重點有
- 引入mutex在data members會讓class object失去copyable, 因為mutex是moveable only type
- mutex的成本可能比atomic貴, 所以如果只有單一variable或memory需要保護, 可以改用atomic

### Item #17
compiler產生的special member functions是inline, public, non-virtual(除非base destructor是virtual, 產生的destructor才會是virtual)

C++98的規則:
- constructor: user沒有宣告constructor
- copy constructor/copy assignment operator: member-wise copy, 宣告copy constructor or assignment operator不會影響另一個的產生 

Big three rules: 如果有宣告destructor, copy constructor或是copy assginment operator其中一個, 那麼三個都要重新定義 

C++11的規則:
- constructor: 同C++98
- destructor : 同C++98, destructor預設是noexcept
- copy operator/copy assignment operator: 同C++98, 但因為big three rules的關係, 自動產生另二個functions是deprecated
如果有定義move operations, 則變成deleted狀態
- move constructor/mov assignment operator: member-wise move, 但如果不支援實際上可能會呼叫到copy operations. move constructor和move
assignment operator如果有定義其一, 另一個就不會自動產生. 如果有copy operations和destructor也不會自動產生
- member function templates不影響上面的規則

如果compiler產生的special member function符合需求, 可以手動宣告再加上=default

---

## Chapter 5 Rvalue References, Move Semantics, and Perfect Forwarding
lvalue和rvalue的區別方式可以用能否對expression取address來決定, 只有lvalue expression才可以對其取address

因此, 所有的parameters都是lvalue, 即使其type是rvalue reference
```c++
void f(Widget&& w); // the type of w is rvalue reference, but it is lvalue 
```

### Item #23
1. 從function回傳的rvalue reference是rvalue
2. std::move是無條件的將其argument cast成rvalue(利用1)
3. std::forward是有條件的將其argument case成rvalue(argument是rvalue reference時)
4. 根據3和4, std::move和std::forward只是cast, runtime不產生額外的executable code
5. movable object不可以宣告成const, 否則會轉換成copy operations(因為move operations的參數沒有const, 只有copy operations才有)

### Item #24
universal references因為都和std::forward搭配, C++標準稱之為forwarding references.

T&&成為universal references的條件
1. 有type deduction的過程
```c++
void f(Widget&& param); // no type deduction, rvalue reference
```
2. 參數格式必須為T&&或auto&&
```c++
template <typename T>
void f(std::vector<T>&& param); // not format T&&, rvalue reference
template <typename T>
void f(const T&& param); // format with const, rvalue reference
```
否則都是rvalue references

### Item #25
std::move搭配rvalue reference使用, std::forward搭配universal reference使用

使用universal reference的好處除了可能可以減少不必要的object construction/destruction外, 也可以減少多個parameters時對各種不同
lvalue/rvalue組合提供overloading functions的困擾

在return by value的function中, 在最後不使用rvalue reference/universal reference的object時, 才做std::move/std::forward,
即使object不支援move operations也沒關係, 因為rvalue reference也可以使用copy operations

RVO(return value optimization)的條件:
1. local object的type和return type相同
2. 直接return local object(不加上額外的std::move, function call之類的東西)
compiler會對此做copy elision, 無法做copy elison則會將local object等同std::move後做為rvalue回傳

### Item #26
本篇是在講解overloading有使用universal reference的function可能會造成的一些問題
由於universal reference是一個function template, 加上universal reference是可以完美match任何參數型態, 造成許多意想不到
的結果
1. 一般function: 呼叫的引數是非int的integral type, 根據overloading rule, 選擇的是universal reference版本, 與預期不合
```c++
template <typename T>
void logAndAdd(T&& name);
void logAndAdd(int idx);
```
2. constructor: 使用在constructor上更糟糕, 使用non-const lvalue構建object和derived class呼叫base class constructor都會使用universal reference版本,
而不是compiler generated的copy/move constructor

### Item #27
本篇是探討#26的解決方案
1. 放棄使用overloading on universal reference:

   a. 將有universal reference的function和其它function用不同的name區別開來

   b. 退回C++98的方式, 使用pass by const T&, 但會失去performance最佳化

   c. 使用pass by value, 相較於universal reference只有多一個move operation, 但在string literal to string的case則會多一個temporary object的產生和銷毀
2. 仍舊使用overloading on universal reference:

   a. 利用tag dispatch的技巧, 使用type traits根據type導至不同的implementation function
```c++
template <typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(std::forward<T>(name), std::is_intergal<std::remove_reference_t<T>::type>());
}
template <typename T>
void logAndAddImpl(T&& name, std::false_type);
void logAndAddImpl(int idx, std::true_type);
```
   b. 使用tag dispatch無法解決constructor的case, 因為compiler generated function會因為normal function優先於template instanced function而繞過了
      tag dispatch的機制, 所以必須使用std::enable_if來限定universal reference template可以instanciate的type

---
