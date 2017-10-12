## Item 10:优先考虑限域枚举而非未限域枚举
条款10:优先考虑限域枚举而非未限域枚举

通常来说，在花括号中声明一个名字会限制它的作用域在花括号之内。但这对于C++98风格的enum中声明的枚举名是不成立的。这些在enum作用域中声明的枚举名所在的作用域也包括enum本身，也就是说这些枚举名和enum所在的作用域中声明的相同名字没有什么不同
```cpp
enum Color { black, white, red };   // black, white, red 和
                                    // Color一样都在相同作用域
auto white = false;                 // 错误! white早已在这个作用
                                    // 域中存在
```
事实上这些枚举名泄漏进和它们所被定义的enum域一样的作用域有一个官方的术语：未限域(unscoped)枚举。在C++11中它们有一个相似物，限域枚举，它不会导致枚举名泄漏：
```cpp
enum class Color { black, white, red }; // black, white, red
                                        // 限制在Color域内
auto white = false;                     // 没问题，同样域内没有这个名字

Color c = white;                        //错误，这个域中没有white

Color c = Color::white;                 // 没问题
auto c = Color::white;                  // 也没问题（也符合条款5的建议）
```
因为限域枚举是通过**enum class**声明，所以它们有时候也被称为枚举类(enum classes)。
