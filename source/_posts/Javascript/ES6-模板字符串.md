---
title: ES6-模板字符串
date: 2019-02-26 16:06:08
tags: [编程,语言,WEB]
categories: JS
---
`模板字面量`是允许嵌入表达式的字符串字面量。你可以使用多行字符串和字符串插值功能。它们在`ES2015`规范的先前版本中被称为“模板字符串”。

<!-- more -->

# ECMAScript6 - 模板字符串

## 语法

```js
`string text`

`string text line 1
 string text line 2`

`string text ${expression} string text`

tag `string text ${expression} string text`
```



---

## 描述

模板字符串使用反引号(\`\`)来代替普通字符串中的用双引号和单引号。模板字符串可以包含特定语法（`${expression}`）的占位符。占位符中的表达式和周围的文本会一起传递给一个默认函数，该函数负责将所有的部分连接起来，如果一个模板字符串由表达式开头，则该字符串被称为带标签的模板字符串，该表达式通常是一个函数，它会在模板字符串处理后被调用，在输出最终结果前，你都可以通过该函数来对模板字符串进行操作处理。在模版字符串内使用反引号（`）时，需要在它前面加转义符（\）。

```js
`\`` === "`" // --> true
```

### 多行字符串

在新行中插入的任何字符都是模板字符串中的一部分，使用普通字符串，你可以通过以下的方式获得多行字符串：

```js
console.log('string text line 1\n' +
'string text line 2');
// "string text line 1
// string text line 2"
```

要获得同样效果的多行字符串，只需使用如下代码：

```js
console.log(`string text line 1
string text line 2`);
// "string text line 1
// string text line 2"
```

### 插入表达式

在普通字符串中嵌入表达式，必须使用如下语法：

 

```js
var a = 5;
var b = 10;
console.log('Fifteen is ' + (a + b) + ' and\nnot ' + (2 * a + b) + '.');
// "Fifteen is 15 and
// not 20."
```

 

现在通过模板字符串，我们可以使用一种更优雅的方式来表示：

 

```js
var a = 5;
var b = 10;
console.log(`Fifteen is ${a + b} and
not ${2 * a + b}.`);
// "Fifteen is 15 and
// not 20."
```

 

### 嵌套模板(表达式)

在某些时候，嵌套模板是具有可配置字符串的最简单也是更可读的方法。 在模板中，只需在模板内的占位符 `${ }` 内使用它们，就可以轻松地使用内部反引号。 例如，如果条件 a 是真的，那么返回这个模板化的文字。

ES5:

```js
var classes = 'header'
classes += (isLargeScreen() ?
   '' : item.isCollapsed ?
     ' icon-expander' : ' icon-collapser');
```

在ES2015中使用模板文字而没有嵌套：

```js
const classes = `header ${ isLargeScreen() ? '' :
    (item.isCollapsed ? 'icon-expander' : 'icon-collapser') }`;
```

 

在ES2015的嵌套模板字面量中：

 

```js
const classes = `header ${ isLargeScreen() ? '' :
 `icon-${item.isCollapsed ? 'expander' : 'collapser'}` }`;
```

 

### 带标签的模板字符串

更高级的形式的模板字符串是带标签的模板字符串。标签使您可以用函数解析模板字符串。标签函数的第一个参数包含一个字符串值的数组。其余的参数与表达式相关。最后，你的函数可以返回处理好的的字符串（或者它可以返回完全不同的东西 , 如下个例子所述）。用于该标签的函数的名称可以被命名为任何名字。

```js
var person = 'Mike';
var age = 28;

function myTag(strings, personExp, ageExp) {

  var str0 = strings[0]; // "that "
  var str1 = strings[1]; // " is a "

  // There is technically a string after
  // the final expression (in our example),
  // but it is empty (""), so disregard.
  // var str2 = strings[2];

  var ageStr;
  if (ageExp > 99){
    ageStr = 'centenarian';
  } else {
    ageStr = 'youngster';
  }

  return str0 + personExp + str1 + ageStr;

}

var output = myTag`that ${ person } is a ${ age }`;

console.log(output);
// that Mike is a youngster
```

正如下面例子所展示的，标签函数并不一定需要返回一个字符串。

```js
function template(strings, ...keys) {
  return (function(...values) {
    var dict = values[values.length - 1] || {};
    var result = [strings[0]];
    keys.forEach(function(key, i) {
      var value = Number.isInteger(key) ? values[key] : dict[key];
      result.push(value, strings[i + 1]);
    });
    return result.join('');
  });
}

var t1Closure = template`${0}${1}${0}!`;
t1Closure('Y', 'A');  // "YAY!" 
var t2Closure = template`${0} ${'foo'}!`;
t2Closure('Hello', {foo: 'World'});  // "Hello World!"
```

### 原始字符串

在标签函数的第一个参数中，存在一个特殊的属性`raw` ，我们可以通过它来访问模板字符串的原始字符串，而不经过特殊字符的替换。

```js
function tag(strings) {
  console.log(strings.raw[0]);
}

tag`string text line 1 \n string text line 2`;
// logs "string text line 1 \n string text line 2" ,
// including the two characters '\' and 'n'
```

另外，使用[`String.raw()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/String/raw) 方法创建原始字符串和使用默认模板函数和字符串连接创建是一样的。

```js
var str = String.raw`Hi\n${2+3}!`;
// "Hi\n5!"

str.length;
// 6

str.split('').join(',');
// "H,i,\,n,5,!"
```

### 带标签的模版字面量及转义序列[节](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/template_strings#%E5%B8%A6%E6%A0%87%E7%AD%BE%E7%9A%84%E6%A8%A1%E7%89%88%E5%AD%97%E9%9D%A2%E9%87%8F%E5%8F%8A%E8%BD%AC%E4%B9%89%E5%BA%8F%E5%88%97)

自ES2016起，带标签的模版字面量遵守以下转义序列的规则：

- Unicode字符以"\u"开头，例如`\u00A9`
- Unicode码位用"\u{}"表示，例如`\u{2F804}`
- 十六进制以"\x"开头，例如`\xA9`
- 八进制以"\"和数字开头，例如`\251`

这表示类似下面这种带标签的模版是有问题的，因为对于每一个ECMAScript语法，解析器都会去查找有效的转义序列，但是只能得到这是一个形式错误的语法：

```js
latex`\unicode`
// 在较老的ECMAScript版本中报错（ES2016及更早）
// SyntaxError: malformed Unicode character escape sequence
```

#### ES2018关于非法转义序列的修订

带标签的模版字符串应该允许嵌套支持常见转义序列的语言（例如[DSLs](https://en.wikipedia.org/wiki/Domain-specific_language)、[LaTeX](https://en.wikipedia.org/wiki/LaTeX)）。ECMAScript提议[模版字面量修订](https://tc39.github.io/proposal-template-literal-revision/)(第4阶段，将要集成到ECMAScript 2018标准) 移除对ECMAScript在带标签的模版字符串中转义序列的语法限制。

不过，非法转义序列在"cooked"当中仍然会体现出来。它们将以 [`undefined`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/undefined) 元素的形式存在于"cooked"之中：

```js
function latex(str) { 
 return { "cooked": str[0], "raw": str.raw[0] }
} 

latex`\unicode`

// { cooked: undefined, raw: "\\unicode" }
```

值得注意的是，这一转义序列限制只对带标签的模板字面量移除，而不包括不带标签的模板字面量：

```js
let bad = `bad escape sequence: \unicode`;
```

---