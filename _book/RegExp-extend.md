# 正则的扩展

## 1. RegExp 构造函数 *

在 ES5 中，`RegExp`构造函数的参数有两种情况。

第一种情况是，第一个参数是字符串，第二个参数是正则表达式的修饰符（flag）。

```javascript
var regex = new RegExp('xyz', 'i');
// 等价于
var regex = /xyz/i;
```

第二种情况是，只有一个参数，参数是一个正则表示式，这时会返回一个原有正则表达式的拷贝。

```javascript
var regex = new RegExp(/xyz/i);
// 等价于
var regex = /xyz/i;
```

但是，ES5 不允许此时使用第二个参数添加修饰符，否则会报错。

```javascript
var regex = new RegExp(/xyz/, 'i');
// Uncaught TypeError: Cannot supply flags when constructing one RegExp from another
```

在ES6中，第一个参数是正则表示式，第二个参数可以使用修饰符（flag），返回的正则表达式会忽略原有的修饰符，使用第二个参数给的修饰符

```javascript
new RegExp(/abc/ig, 'i').flags
// "i"
```

## 2. 字符串的正则表达式

字符串对象共有 4 个方法，可以使用正则表达式：`match()`、`replace()`、`search()`和`split()`。

ES6 将这 4 个方法，在语言内部全部调用`RegExp`的实例方法，从而做到所有与正则相关的方法，全都定义在`RegExp`对象上。

- `String.prototype.match` 调用 `RegExp.prototype[Symbol.match]`
- `String.prototype.replace` 调用 `RegExp.prototype[Symbol.replace]`
- `String.prototype.search` 调用 `RegExp.prototype[Symbol.search]`
- `String.prototype.split` 调用 `RegExp.prototype[Symbol.split]`

## 3. RegExp.prototype.flags 属性

ES6 为正则表达式新增了`flags`属性，会返回正则表达式的修饰符。

```js
// ES5 的 source 属性
// 返回正则表达式的正文
/abc/ig.source
// "abc"

// ES6 的 flags 属性
// 返回正则表达式的修饰符
/abc/sigu.flags
// 'gi'
```

## 4. 具名组匹配 *

正则表达式使用圆括号进行组匹配。使用`exec`方法，就可以将这三组匹配结果提取出来。

```javascript
const RE_DATE = /(\d{4})-(\d{2})-(\d{2})/;

const matchObj = RE_DATE.exec('1999-12-31');
const year = matchObj[1]; // 1999
const month = matchObj[2]; // 12
const day = matchObj[3]; // 31
```

组匹配的一个问题是，每一组的匹配含义不容易看出来，而且只能用数字序号（比如`matchObj[1]`）引用，要是组的顺序变了，引用的时候就必须修改序号。

ES2018 引入了[具名组匹配](https://github.com/tc39/proposal-regexp-named-groups)（Named Capture Groups），允许为每一个组匹配指定一个名字，既便于阅读代码，又便于引用。同时，数字序号（`matchObj[1]`）依然有效。

```javascript
const RE_DATE = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;

const matchObj = RE_DATE.exec('1999-12-31');
const year = matchObj.groups.year; // 1999
const month = matchObj.groups.month; // 12
const day = matchObj.groups.day; // 31
```

如果具名组没有匹配，那么对应的`groups`对象属性会是`undefined`。

```javascript
const RE_OPT_A = /^(?<as>a+)?$/;
const matchObj = RE_OPT_A.exec('');

matchObj.groups.as // undefined
'as' in matchObj.groups // true
```

## 5. 解构赋值和替换 * 

有了具名组匹配以后，可以使用解构赋值直接从匹配结果上为变量赋值。

```javascript
let {groups: {one, two}} = /^(?<one>.*):(?<two>.*)$/u.exec('foo:bar');
one  // foo
two  // bar
```

字符串替换时，使用`$<组名>`引用具名组。

```javascript
let re = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/u;

'2015-01-02'.replace(re, '$<day>/$<month>/$<year>')
// '02/01/2015'
```

上面代码中，`replace`方法的第二个参数是一个字符串，而不是正则表达式。

`replace`方法的第二个参数也可以是函数，该函数的参数序列如下。

```javascript
'2015-01-02'.replace(re, (
   matched, // 整个匹配结果 2015-01-02
   capture1, // 第一个组匹配 2015
   capture2, // 第二个组匹配 01
   capture3, // 第三个组匹配 02
   position, // 匹配开始的位置 0
   S, // 原字符串 2015-01-02
   groups // 具名组构成的一个对象 {year, month, day}
 ) => {
 let {day, month, year} = groups;
 return `${day}/${month}/${year}`;
});
```

## 6. 引用

如果要在正则表达式内部引用某个“具名组匹配”，可以使用`\k<组名>`的写法。

```javascript
const RE_TWICE = /^(?<word>[a-z]+)!\k<word>$/;
RE_TWICE.test('abc!abc') // true
RE_TWICE.test('abc!ab') // false
```

数字引用（`\1`）依然有效。

```javascript
const RE_TWICE = /^(?<word>[a-z]+)!\1$/;
RE_TWICE.test('abc!abc') // true
RE_TWICE.test('abc!ab') // false
```

这两种引用语法还可以同时使用。

```javascript
const RE_TWICE = /^(?<word>[a-z]+)!\k<word>!\1$/;
RE_TWICE.test('abc!abc!abc') // true
RE_TWICE.test('abc!abc!ab') // false
```

## 7. String.prototype.matchAll() *

[ES2020](https://github.com/tc39/proposal-string-matchall) 增加了`String.prototype.matchAll()`方法，可以一次性取出所有匹配。不过，它返回的是一个遍历器（Iterator），而不是数组。

```javascript
const string = 'test1test2test3';

// g 修饰符加不加都可以
const regex = /t(e)(st(\d?))/g;

for (const match of string.matchAll(regex)) {
  console.log(match);
}
// ["test1", "e", "st1", "1", index: 0, input: "test1test2test3"]
// ["test2", "e", "st2", "2", index: 5, input: "test1test2test3"]
// ["test3", "e", "st3", "3", index: 10, input: "test1test2test3"]
```

上面代码中，由于`string.matchAll(regex)`返回的是遍历器，所以可以用`for...of`循环取出。相对于返回数组，返回遍历器的好处在于，如果匹配结果是一个很大的数组，那么遍历器比较节省资源。

遍历器转为数组是非常简单的，使用`...`运算符和`Array.from()`方法就可以了。

```javascript
// 转为数组方法一
[...string.matchAll(regex)]

// 转为数组方法二
Array.from(string.matchAll(regex))
```