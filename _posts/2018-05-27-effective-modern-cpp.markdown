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
ParamType是T和const, \*(pointer), &/&&(reference)和volatile的各種可能組合

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

### Item #3: Understand decltype
- decltype(name)得到的type就是entity name的真實type
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

### Item #4: Know how to view deduced types
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

### Item #5: Prefer auto to explicit type declarations
使用auto的好處
- 避免uninitialized variable問題
- 較少的typing
- 可以正確表示lamda closure的型別(只有compiler才知道)
  - 轉型成std::function會增加記憶體使用量和較差的執行效率
- 避免錯誤的type宣告帶來不必要的轉型, 造成程式行為怪異和效能損失

### Item #6: Use the explicitly type initializer idiom when auto deduces undesired types
由於"invisible" proxy type的關係(ex. std::vector\<bool\>::reference), auto有可能會推導出錯誤的type,
造成預期外的結果. 解決方式是使用the explicitly type initializer idiom強制auto推導出需要的type
```c++
auto var = static_cast<T>(expr);  // T is the type you want
```

[Reference]
1. ["auto to stick" and Changing Your Style](https://www.fluentcpp.com/2018/09/28/auto-stick-changing-style/)

---

## Chapter 3 Moving to Modern C++

### Item #7: Distinguish between () and {} when creating objects
C++11新增了uniform initialization(braced initialization)的語法後, 初始化的方式可以有以下幾種組合
```c++
int x(0);    // direct initialization
int x = 0;   // copy initialization
int x{0};    // direct list initialization
int x = {0}; // copy list initialization
```
C++11出現braced initialization的原因是=和()的初始化方式在某些使用情況下有限制
- 初始化non-static data members不能使用()
  在C++11之前, 只能初始化static const data members
```c++
class Widget
{
    ...
private:
    static const int x = 0;
    int y{0};               // also can use "=" to initialize
};
```
- 在non-copyable object中就不能使用=
- most vexing parse問題
- 防止narrowing conversion
- 直接初始化container的內容

Braced initialization的缺點是, 當type的constructor有多個, 且其中一個constructor
有使用std::initializer_list當做參數的話, 不論如何, braced initialization都會以
std::initializer_list為優先對象
```c++
std::vector<int> v1(10, 20); // create 10 elements with value 20
std::vector<int> v2{10, 20}; // create 2 elements 10, 20
```
雖然作者對使用那種initialization方式沒有明確的建議, 但是braced initialization的確也帶來了一些
意想不到的混亂. 參考部份給出了其它的比較與建議.

[Reference]
1. [Tip of the Week #88: Initialization: =, (), and {}](https://abseil.io/tips/88)
2. [Initialisation in C++17 – the matrix](https://timur.audio/initialisation-in-c17-the-matrix)

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

### Item #17 Understand special member function generation
compiler產生的special member functions是public, inline, non-virtual(除非base destructor是virtual, 產生的destructor才會是virtual)

C++98的規則:
- default constructor: user沒有宣告任何constructor(包含copy constructor, constructor function template)
- copy constructor/copy assignment operator: member-wise copy non-static data members, 宣告copy constructor or assignment operator不會影響另一個的產生 

Rule of three: 如果有宣告destructor, copy constructor或是copy assginment operator其中一個, 那麼這三個member functions都要宣告

C++11的規則:
- default constructor: 同C++98(包含move constructor)
- destructor : 同C++98, 除了destructor預設是noexcept
- copy operator/copy assignment operator: 同C++98, 但因為rule of three的關係, 自動產生另二個member functions是deprecated.
如果有宣告move operations, 則變成deleted狀態
- move constructor/move assignment operator: member-wise move non-static data members. move constructor和move assignment operator如果有宣告其一,
另一個就不會自動產生. 如果有宣告copy operations和destructor也不會自動產生

Howard Hinnant整理出了一個對照表![](http://howardhinnant.github.io/smf.jpg)

[Reference]
1. [Compiler-generated Functions, Rule of Three and Rule of Five](https://www.fluentcpp.com/2019/04/19/compiler-generated-functions-rule-of-three-and-rule-of-five/)
2. [The Rule of Zero in C++](https://www.fluentcpp.com/2019/04/23/the-rule-of-zero-zero-constructor-zero-calorie/)

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

### Item #28
本篇是對universal reference的正式解釋, #24雖然不是正確的解釋, 但在概念上相對來說比較容易理解
universal reference實際上仍然是rvalue reference, 只是符合以下二種情況
- 有type deduction區分lvalues和rvalues
- 可能發生reference collapsing

reference collapsing的規則很簡單, 只要前後之一的reference是lvalue reference, 結果就是lvalue reference, 否則才會是rvalue reference

reference collapsing發生的四種contexts:
- template instantiation
- auto type generation
- typedef and alias declaration
- decltype

### Item #29
C++11引入了move operations, 可以減少copy operations增進performance, 但是move operations不是萬靈丹, 有些情況下並不會發生效用
- object不提供move operations
- 就算了用move也沒有比copy快多少, 例如std::array和SOO情況下的std::string
- 需要no exception的move operations確沒有宣告成noexcept
- source object是lvalue(隨便move lvalue不是件好事, 除非確定此lvalue在move後不再使用)

---

## Chapter 6 Lambda Expressions
lambda expression產生的object稱為closure, 其type為closure class, 每個lambda expression的closure class都是唯一的

### Item 31: Avoid default capture modes
by reference capture要注意的是reference的object可能在closure引用時失效造成的dangling reference問題

by value capture要注意的是pointer被capture(包含this pointer), 可能會有dangling pointer問題

在class的member function內capture member variable by value
- 宣告一個local variable, 將member variable copy至local variable, 再capture此local variable
- 使用C++14的generalized lambda capture

C++17新增了`[*this]`可以capture this object by copy, C++20開始default capture by value不再implicit capture this pointer

[Reference]
1. [Back to Basics: Lambdas from Scratch - Arthur O'Dwyer - CppCon 2019](https://youtu.be/3jCOwajNch0)

### Item 32: Use init capture to move objects into closures
C++14引入了init capture(又稱為generalized lambda capture)解決了C++11無法capture by move的限制
```c++
auto pw = std::make_unique<Widget>();
...
auto func = [pw = std::move(pw)]
            {...}
```
等號左邊和大括號內的pw是closure class的data member, 等號右邊的pw則是屬於lambda expression定義時的scope

C++11的做法有二種:
1. 改用functor(因為lambda expression只是functor的syntax sugar)
2. 利用std::bind, 將movable object傳至std::bind當做lambda expression的參數
```c++
// lambda by default is const, to avoid modifications we need const
// in lambda's parameter. For mutable lambda, const is no need.
std::bind([](const std::unique<Widget>& pw) {
                ...
            },
            std::make_unique<Widget>());
```

### Item 33: Use decltype on auto&& parameters to std::forward them
C++14有了generic lambda, lambda expression的參數可以使用auto, 對應的實作就是functor的operator()變成了function template

如果lambda expression的參數想要使用perfect forwarding, 參數要宣告成auto&&, 套用的std::forward則是需要decltype的幫忙
```c++
auto f =
    [](auto&& param)
    {
        return func(normalize(std::forward<decltype(param)>(param)));
    };
```

C++20可以在lambda expression使用template parameters, 所以也可以寫成
```c++
auto f =
    []<typename T>(T&& param)
    {
        return func(normalize(std::forward<T>(param)));
    };
```

### Item 34: Prefer lambdas to std::bind
std::bind的用處只剩下在C++11模擬move capture和generic lambda的功用, 其應用上的缺失和複雜, 都可以由lambda expression解決

---

## Chapter 7 The Concurrency API

### Item #35
使用task based(std::async)的方式相較於thread based(std::thread)可以把thread exhausion, oversubscription, load balancing
等問題留給standard library處理

### Item #36
std::async有二種launch policy, 預設是這二種policy的組合
1. std::launch::async: 真的create thread來執行task(**asynchronously**)
2. std::launch::deferred: 當呼叫get or wait時才執行task(**synchronously**)

在使用std::async的wait loop中要小心避免造成endless loop, 必須考慮task是否被defer execution的情形
```c++
auto fut = std::async(f);
if (fut.wait_for(0s) == std::future_status::deferred)
{
    // running synchronously
    ...
}
else
{
    // running asynchronously
    while (fut.wait_for(100ms) != std::future_status::ready)
    {
        // task is not finished
        ...
    }
    // task is finished
    ...
}
```

---

## Chapter 8 Tweaks

### Item #41: Consider pass by value for copyable parameters that are cheap to move and always copied

實作member function `addName`, 傳入的參數只有一個std::string `newName`, `newName`會被加入std::vector.
考慮以下二種使用情形, 有三種不同的方式來實作`addName`:

```c++
Widget w;
std::string name("Bart");
w.addName(name);           // call addName with lvalue
w.addName(name + "Jenne"); // call addName with rvalue
```
1. Overloading:
```c++
class Widget {
public:
    void addName(const std::string& newName)
    {
        names.push_back(newName);
    }
    void addName(std::string&& newName)
    {
        names.push_back(std::move(newName));
    }
private:
    std::vector<std::string> names;
};
```
此方式需要實作二個functions, 產生的object code也可能有二個functions(如果沒有inline).  
Cost分析:
- parameter是lvalue: 1 copy constructor(names的push_back)
- parameter是rvalue: 1 move constructor(names的push_back)
2. Using a universal reference:
```c++
class Widget {
public:
    template<typename T>
    void addName(T&& newName)
    {
        names.push_back(std::forward<T>(newName));
    }
private:
    std::vector<std::string> names;
};
```
此方式只需要實作一個function template在header file, 但在使用時會實體化成二個functions,
所以產生的object code和overloading方式一樣有code bloat問題.  
Cost分析:
- parameter是lvalue: 1 copy constructor(names的push_back)
- parameter是rvalue: 1 move constructor(names的push_back)

3. Passing by value:
```c++
class Widget {
public:
    void addName(std::string newName)
    {
        names.push_back(std::move(newName));
    }
private:
    std::vector<std::string> names;
};
```
此方式的實作重點在於std::move, 因為newName已與caller無關, 所以可以安全的將其move至names.  
Cost分析:
- parameter是lvalue: 1 copy constructor(newName) + 1 move constructor(names的push_back)
- parameter是rvalue: 1 move constructor(newName) + 1 move constructor(names的push_back)

本條目的標題有幾項重點:
1. consider: 考慮pass by value的方式需仔細評估花費.
2. copyable parameters: 參數必須是copy constructible type. 如果參數只是move-only type,
overloading的方式只需提供一種function, 因此pass by value的方式一點意義也沒有.
3. cheap to move: 相較於其它方式, pass by value多一個move constructor的cost, 因此只有在此cost影響不大時才考慮
4. always copied: 傳入的參數在function裡的流程都是有copy的動作, 如果只有某些條件下才進行copy就不符合.

如果function裡的copy是assignment copy而不是constructor copy, 對std::string而言, 當舊字串長於新字串時,
pass by value相較於pass by reference會多了memory allocation和deallocation的高花費操作.
類似的情形也會發生於有dynamic memory allocation的type, 例如std::vector.

即使pass by value只多了一個cheap move operation, 當一連串的pass by value發生時, 累積的move花費仍不可忽視.
