# C++ 常用工具箱（做题向）

> 这份不是教程，是**清单**。做题时遇到不熟的就来查，查完在对应条目上打个 `✓`——比通读一遍教程有用得多。

## 一、迭代器：先把这个模型建对

迭代器是**类型**，不是地址。它提供的是「指针式的一小组操作」，具体能做什么由**迭代器类别**决定：

| 类别 | 例子 | 能做 |
| --- | --- | --- |
| 随机访问 | `vector` / `deque` / 数组指针 | `+ - < >`、`it[i]`、`it2 - it1`（O(1)） |
| 双向 | `list` / `set` / `map` | 只能 `++ --`，不能相减、不能 `+n` |
| 单向 | `forward_list` / `unordered_*` | 只能 `++` |

**最常考的三件事**：

```cpp
auto it = lower_bound(v.begin(), v.end(), x);   // 迭代器
int  idx = (int)(it - v.begin());               // 下标：相减，别再除 sizeof
int  d   = (int)(it2 - it1);                    // 距离：相减就是元素个数（ptrdiff_t）
T    val = *it;                                 // 取值：解引用
```

- `it - begin()` 的结果是 `ptrdiff_t`（有符号），赋给 `int` 要强转，否则 `-Wall` 会警告。
- 非随机访问容器（`list`/`set`）**不能相减**，要 `std::distance(it1, it2)`，而且是 O(n)。
- 区间一律**左闭右开** `[begin, end)`：`end()` 指向「最后一个元素的下一个位置」，**不能解引用**。
- `end()` 不是「最后一个元素」！最后一个元素是 `prev(v.end())` 或 `v.back()`。

**失效规则**（做题时最容易踩）：

| 操作 | 失效的迭代器 |
| --- | --- |
| `vector::push_back` 触发扩容 | 全部（包括 `end()`） |
| `vector::erase(it)` | 被删的及之后的全部 → 用 `it = v.erase(it)` |
| `map/set::erase(it)` | 只有被删的 → 同样 `it = s.erase(it)` |
| `unordered_map` 任何插入 | 可能 rehash，全部失效 |

## 二、STL 算法清单（按做题频率排）

```cpp
sort(v.begin(), v.end());                        // 升序；自定义比较器只能写严格弱序 <
sort(v.rbegin(), v.rend());                      // 降序，比写 greater<> 更短
stable_sort(...);                                 // 需要保持相等元素原顺序时
reverse(v.begin(), v.end());

lower_bound(v.begin(), v.end(), x);              // 第一个 >= x 的迭代器
upper_bound(v.begin(), v.end(), x);              // 第一个 >  x 的迭代器
binary_search(v.begin(), v.end(), x);            // 只回答「在不在」

accumulate(v.begin(), v.end(), 0LL);             // 求和，第三参决定类型（0 会溢出！）
accumulate(v.begin(), v.end(), 0LL, plus<long long>());
max_element / min_element(v.begin(), v.end());   // 返回迭代器，取值要 *it
count / count_if(v.begin(), v.end(), x);         // 计数
find / find_if(v.begin(), v.end(), x);           // 找不到返回 end()

unique(v.begin(), v.end());                      // 只把重复挪到后面，必须配 erase
v.erase(unique(v.begin(), v.end()), v.end());    // 标准去重（要求先排序）

nth_element(v.begin(), v.begin() + k, v.end());  // 第 k 小就位，O(n) 均摊
partial_sum(v.begin(), v.end(), out.begin());    // 前缀和
iota(v.begin(), v.end(), 0);                     // 填充 0,1,2,...
fill(v.begin(), v.end(), x);

next_permutation(v.begin(), v.end());            // 下一个排列；返回 false 表示已到最后一个
prev_permutation(...);

min(a, b); max(a, b); clamp(x, lo, hi);          // clamp 是 C++17
gcd(a, b); lcm(a, b);                            // <numeric>，C++17
swap(a, b);
```

小工具：

```cpp
__builtin_popcount(x);        // int 的 1 的个数
__builtin_popcountll(x);      // long long
__builtin_ctz(x) / clz(x);    // 末尾/开头 0 的个数（x != 0）
INT_MAX / INT_MIN / LLONG_MAX
to_string(x); stoi(s); stoll(s);
```

### lower_bound / upper_bound 的官方定义（值得背下来）

标准里的定义只有一句，**不是**「二分查找目标值」：

| 函数 | 返回 | 用 `<` 说人话 |
| --- | --- | --- |
| `lower_bound(f, l, v)` | 第一个**不小于** `v`（`>= v`）的元素 | 第一个使 `!(elem < v)` 的位置 |
| `upper_bound(f, l, v)` | 第一个**大于** `v`（`> v`）的元素 | 第一个使 `v < elem` 的位置 |

两者都**不保证元素等于 `v`**，找不到就返回 `last`（`end()`）。所以它们本质是「**插入位置**」：

- `lower_bound` = 插入 `v` 后、`v` 排在**已有相等元素前面**的位置；
- `upper_bound` = 插入 `v` 后、`v` 排在**已有相等元素后面**的位置；
- `upper_bound - lower_bound` = 等于 `v` 的元素个数。

```
pref = [0, 1, 3, 3, 7]        v = 3
        ↑        ↑
     lower=2  upper=4     → 下标 2、3 都是 3，共 4-2=2 个
```

**前提不是「已排序」，而是「已按 `<` 分好区」**：所有 `elem <= v` 的元素必须排在所有 `elem > v` 的元素前面（`upper_bound` 的情形）。有序数组当然满足，所以平时不用区分；但在「先降后升」这类局部有序的数组上，用 `upper_bound` 前要自己确认分区成立。

`2389` 就是标准用法：`pref` 升序，`upper_bound(pref, q)` 得到第一个 `> q` 的下标，减一就是「最后一个 `<= q`」。

自定义比较器时注意参数顺序：`upper_bound(f, l, v, comp)` 内部调的是 **`comp(v, *it)`**（值在前、元素在后），写反了语义就变成 `lower_bound` 了。

## 三、string 的高频成员

```cpp
s.substr(pos, len);          // 第二参是长度！不是结束下标
s.find(t) / s.rfind(t);      // 返回 size_t，找不到是 string::npos
s.find(t, pos);              // 从 pos 开始找
s.push_back(c); s.pop_back();
s.replace(pos, len, t);
s.compare(...);
s.c_str();                   // 传给 C 风格 API
stoi / stol / stod;          // 要和 try-catch 或先校验配合（会抛异常）
s.begin() + i;               // string 也支持迭代器
```

坑：`s.size()` 是 `size_t`（无符号），`s.size() - 1` 在空串时是天文数字；`<cctype>` 的 `isalpha` 传负的 `char` 是 UB，先转 `unsigned char`。

## 四、`<ranges>` 要不要学

**结论：现在不用系统学，会 4 个就够。**

```cpp
ranges::sort(v);                                  // 省掉 .begin(), .end()
ranges::find(v, x);                               // 返回迭代器
ranges::count(v, x);
for (int i : views::iota(0, n)) { ... }           // 代替 for (int i = 0; i < n; ++i)
ranges::max(v);                                   // 返回值，不是迭代器（和 max_element 不同！）
```

为什么不用深入：

- 做题时 `views | filter | transform` 的链式管道**会隐藏复杂度和临时对象**，一眼看不出是 O(n) 还是 O(n²)，debug 也难；
- 面试/笔试通常要求手写，考官看的是你能不能写出循环；
- 真正需要的时候（业务代码、批量处理）再学，成本不高。

**优先级**：`for i in 0..n` 的循环写法 → 手写二分/滑窗 → `sort`/`lower_bound`/`accumulate` → 其余按需查。

## 五、什么时候该手写而不是调库

| 场景 | 该做什么 |
| --- | --- |
| 纯「找第一个 >= x」 | 用 `lower_bound`，快且不会错 |
| 二分答案（答案不在数组里） | **手写**双闭模板 |
| 需要在二分过程中维护额外状态（如 2300 的阈值、旋转数组判半边） | **手写** |
| 要求返回下标且容器是 `set` | 用成员函数 `s.lower_bound(x)` |
| 排序比较器 | 必须严格弱序（`<`），写 `<=` 在 `std::sort` 里是 UB，可能 RE |

## 六、我的理解演进

- 2026-10-30：建这份清单。起因是 2300 把 `lower_bound` 的返回值当成地址、还想 `/ sizeof(int)` 换下标——不是算法不懂，是**迭代器的模型没建对**。以后按清单逐条打勾，不通读教程。
