本项目是  _**Scott Meyers**_ 所写的 **_Effective Modern C++_** 的中文简体翻译。  
>This is a Chinese translation of the **_Effective Modern C++_** written by _**Scott Meyers**_.  

当然，这个翻译对于一些读者来说是不完美的和不标准的，因为存在着很多主观理解。我会持续改进的，如果大家  
有什么建议的话，请给我发邮件，这是我的邮箱 <ljqvincent@163.com>。  
>Of course, the translation is not perfect and not standard for some people, because there is some >subjective  
>understandings. I will continue to improve it, so email me <ljqvincent@163.com> if you have any >suggestions.

下面是我不知道如何解决的问题，如果大家知道怎么解决的话，请给我发邮件。  
* 如何让 _Markdown_ 中的 _Preview_ 中的每一行都一样长。
* 同一个 _md_ 中有相同标题时，如何进行设置。
>The following is some issues that i don't know how to fix. Please email me if anybody knows how to >fix the issues. 
>* How to make each line the same length in the Preview of Mardown.
>* When there are same headlines in the same md, how to set up.

翻译惯例

* case 场景
* situation 情景 情况
* example 例子
* scenario 情况
* for example 例如
* eg 比如
* specification 规范
* general 通用的 普遍的
* suppose 假设 假如 假定
*  eligibl 适合
*  copy 副本
* scope 作用域
* container 容器
* assignment 赋值
* _conversion_ 转换
* null pointer 空指针
* transformation 转换
*  _braced initializer_ 花括号初始化表达式
* _initializer_ 表达式
* _enums_ 枚举
* braces {}
* parentheses ()
* _lamba expression_ 
* _iterator_
* _undefined behavior_
* _move operations_
* _copy operations_
* _exception safe_
* _basic guarantee_
* _strong guarntee_
* _callable_
*  _signature_
*  _universal reference_
* _constness_
* _non-reference_
* _uniform initialization_
* _trailing return type_
* _container_
* _proxy_
* _alias declartion_ 
* _braced initialization_
* _uncopyable_
* _narrowing conversions_
* _legacy_
* _numeric_ 
* _integral_
* _mutex_
* _alias declarations_
* _allocator_
* _dependent type_
* _type traits_
* _tutorial_
* _Standardization Committee_
* _scoped enums_
* _unscoped enums_
* _namespace_
* _strongly typed_
> 比如 container 可以翻译为容器，也可以使用 _container_，大部分都使用的是 _container_。

译者注  
* 在 _C++_ 中，基本类型是：_int_ 类型、_bool_ 类型和 _unsigned_ 等，复合类型是：引用类型、指针类型、数组类型  
和枚举类型等，而 **_cv_** 则为限定符，即为：_const_ 和 _volatile_ 为限定符，既不属于基本类型也不属于复合类型。  
为了方便表述，在本翻译中将基本类型、复合类型和 **_cv_** 限定符都称作了类型，这也是复合原文的。

* 当本翻译中提及类型时，有时候指的是此类型，而有时候指的是此类型所对应的对象或变量；反之也成立，当  
本翻译中提及对象或变量时，有时候指的是此对象或变量，而有时候指的是此对象或变量所对应的类型。比  
如：**_universal reference_** 有时候指的是 _universal reference_ 类型，而有时候指的是 _universal reference_ 类型所  
对应的对象或变量；比如**引用**有时候指的是引用类型，而有时候指的是引用类型的变量；进行了一些**省略**，并  
没有拘泥于严格的定义，因为那样会很冗长繁琐，本文只会在重要的地方重点说明下，根据上下文环境理解即  
可，这也是复合原文的。
