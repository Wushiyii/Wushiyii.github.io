---
title: TypeScript入门语法
date: 2018-08-16 10:00:57
tags: [编程,语言,WEB]
categories: WEB 
---
##### TypeScript 简明语法

`TypeScript`是微软在`JavaScript`基础上进行扩充的超集，扩展了语法以及相应的解决了许多`JS`中奇怪的BUG。并且`TypeScript`也遵循最新的ES6标准。

<!-- more -->

## 基础类型
### 布尔类型、数字、字符串
```typescript
let flag: boolean = false;

let num: number = 1;
let num: number = 0xffff;

let str: string = 'James';
let name: string = `name is ${name}`;//采用模版字符串,可以传值
```
### 数组、元组、枚举
```typescript
let arr: number[] = [1, 2, 3];
let arr: Array<number> = [1, 2, 3]; //采用数组泛型

let x: [string, boolean];//定义元组
x = ['str', true];//赋值

enum Gender{Male, Female};
let g: Gender = Gender.Female;

enum Gender{Male = 2, Female = 5};//可以自定义编号
let g: Gender = Gender.Male;
```
### 任意值、空值、NEVER、断言
```typescript
let anyType: any = 4;//可以为任意类型
anyType = 'Hello world';//可以重置为别的类型
let list: any[] ;//定义任意值数组

//void类型只能定义undefined与null以及方法的返回值
let a: void = undefined;
let a: void = null;
//never类型表示的是那些永不存在的值的类型
function error(message: string): never {
    throw new Error(message);
}
//用于让编译器某个值已经被验证过
let a: 'a sentence';
let length = (<string>a).length;
let length = (a as string).length;//第二种写法
```
### 联合类型
可以定义多种类型，即可以为变量赋值三种类型之一
```typescript
let a: number | string | null;
```

### Const、Let、Var
- `const`可以定义变量，但是变量的值不可以再改变了。
- `let`与`var`类型，但是在作用域，语义规则等方面比`var`完善。
- `var`即为`js`中定义变量的语法一致。

---
## 面向对象
### 接口
`typescript`规定了一种示范，要使用继承接口的变量必须包含有接口的特性，但也可以不用全部包含
```typescript
interface Animal{
	name: string;
	color?: string;//?表示为可选
}
function fun(animal: Animal){
	console.log(animal.name);
}
let dog = {name: 'dog', age: 3};
fun(dog);
interface Animal{
	readonly name: string;//readonly规定了继承了接口，赋值给此属性后就不可以改变了
}
```
### 类
类与`java`、`C#`差不多，作用也类似。
```typescript
class Greeter {

	static a: string = 'a';//静态属性
	static b: number = 1;
	
    greeting: string;
    constructor(message: string) {
        this.greeting = message;
    }
    greet() {
        return "Hello, " + this.greeting;
    }
    get greeting(): string {//Getter
	    return this.greeting;
    }
    set greeting(greeting: string): void {//Setter
		this.greeting = greeting;
	}
}

let greeter = new Greeter("world");
```
在`TypeScript`中，类中的属性都默认为`public`，但是也可以明确写出来。`private`声明后的属性，只可以在本类中访问。`protected`属性可以在派生类中使用。

---
### 泛型
```typescript
var animal: Array<Animal> = [];//animal数组只可以存放指定类型对象

animal[0] = new Animal("Pig");
animal[1] = new Cake("cake");//error
```
### 模块
模块可以防止命名空间冲突，此外不同的模块更容易维护
```typescript
import {FunnyComponent }from "./funny";//import类
module MyModule{
	export class Animal{//export设置属性对外暴露
		private name: string;

        get name(): string { 
            return this.name;
        }

        set name(name: string) {
            this.name = name;
        }

        constructor(name: string) {
            this.name = name;
        }

        sayHello(): void {
            alert("hello animal:" + this.name);
        }
	}
	export class Cat extends Animal{
		sayHello(): void {
            alert("hello cat:" + this.name);
        }
	}

}
```
## 一些语法
### 箭头表达式
箭头表达式即是匿名函数function(){}的简写，可以解决`JS`中`this`指针的问题，并且表达也更加简洁。
```typescript
(function(){return 1;});//JS
() => 1;//TS

(function(a){return a;});//JS
(a) => a;//TS

(function(a){a;});//JS
(a) => {a};//TS

(function(a){return abc:ABC});//JS
(a) => {return (a)};//TS
```
在函数中可以替代this
```typescript
//JS写法
function a(age: number){
	this.age = age;
	setTimeOut(function(){
		console.log(`A'age : ${this.age}`);
	},0);
}
//TS写法
function a(age: number){
	this.age = age;
	setTimeOut(()=>{console.log(`A's age : ${this.age}`)});
}
```
此外还有一些乱七八糟的用法
```typescript
//对数组中所有元素进行求和操作
var result = [1, 2, 3]
  .reduce((total, current) => total + current, 0);
 
console.log(result);//结果：6

//获取数组中所有偶数
var even = [3, 1, 56, 7].filter(el => !(el % 2));
console.log(even);//结果：[56]

//根据price和total属性对数组元素进行升序排列
var data = [{"price":3,"total":3},{"price":2,"total":2},{"price":1,"total":1}];
var sorted = data.sort((a, b) => {
  var diff = a.price - b.price;
  if (diff !== 0) {
    return diff;
  }
  return a.total - b.total;
});
console.log(sorted);
//结果：[{price: 1, total: 1},
		{price: 2, total: 2},
		{price: 3, total: 3}]

```
### foreach、for in、for of三种循环
```typescript
var arr= [10,20,30,40,50,60];
myArray.forEach(value => console.log(value));

for(var n in myArray)//for in循环
    console.log(n); 


for(var n of myArray){//for of循环可以使用break
    if（n <4）
       break;
    console.log(n);
}
```
### 注解
在Angular中有着大量的注解，而注解作为代码的元素，在`TypeScript`中可以起着说明的作用，可以让框架扫描到对应注解。
```typescript
//app.component.ts
import {Component} from '@angular/core';

@Component({//Angular自带注解，可以让框架识别到其中的属性，并使用
    selector: 'app-root',
    templateUrl: './app.component.html',
    styleUrls: ['./component.css']
})

export class AppComponent{
    title = 'app works';
}
```