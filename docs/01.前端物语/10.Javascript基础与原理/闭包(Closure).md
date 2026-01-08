---
title: 闭包（Closure）
date: 2025-12-25 21:25:02
permalink: /web/javascript/closure    
categories:
  - 前端物语
tags:
  - javascript、闭包
---

## 什么是闭包（Closure）？它可能导致哪些问题？如何避免内存泄漏？
闭包是一个函数以及其创建时所在的词法环境（Lexical Environment）的组合。​简单来说，**一个函数可以“记住”并访问它被创建时所处的外部作用域，即使它在那个作用域之外被执行，这种现象就叫做闭包。**

闭包的核心由两部分组成：
- 1.**函数本身**
- 2.**一个指向其词法环境的引用**，这个环境包含了该函数创建时作用域内的所有局部变量。

> 经典示例：计数器
```javascript
// 定义创建计数器的工厂函数
function createCounter() {
  // 私有变量：仅在 createCounter 作用域内可见，外部无法直接访问
  let count = 0;

  // 返回内部函数（闭包）：能够持续访问外部函数的 count 变量
  return function() {
    count++; // 修改外部函数的私有变量
    console.log(count); // 打印当前计数器值
  };
}

// 创建第一个独立的计数器实例
const counter1 = createCounter();
// 创建第二个独立的计数器实例
const counter2 = createCounter();

// 调用第一个计数器，维护自身的 count 状态
counter1(); // 输出: 1
counter1(); // 输出: 2

// 调用第二个计数器，维护自身独立的 count 状态
counter2(); // 输出: 1 (与 counter1 的 count 互不干扰)
```

解析：
- createCounter 函数执行后，其内部的 count 变量本应随着函数执行完毕而被销毁。
- 但是，由于返回的匿名函数（我们将其赋值给了 counter1 和 counter2）仍然在引用 count，所以垃圾回收机制无法回收 count。
- counter1 和 counter2 各自拥有一个独立的闭包环境。它们各自“记住”了自己创建时环境中的那个 count 变量，因此它们的状态是相互隔离的。
- 你可以把闭包想象成一个函数背着一个“背包”，背包里装着它诞生时环境里的所有变量。无论这个函数走到哪里去执行，它都可以随时打开背包使用里面的东西。

## 闭包可能导致哪些问题？
> 虽然闭包非常强大，但如果不正确使用，也会带来一些问题。

### 1、内存泄漏（Memory Leak）

**这是最常被提及的问题。**

> **原因**：由于闭包会使其外部作用域的变量一直保存在内存中，如果这个闭包的生命周期很长（例如，被一个全局变量引用，或者作为DOM元素的事件监听器），那么这些外部变量也同样会一直占用内存，无法被垃圾回收。如果这些变量占用的内存很大，就可能导致内存泄漏。

**示例**：
```javascript
// 给页面元素绑定点击事件监听器（包含闭包场景演示）
function attachEventListener() {
  // 模拟一个体积很大的字符串数据（仅作演示，无实际业务意义）
  let largeData = new Array(1000000).join('*');
  // 通过元素ID获取页面中的目标DOM元素
  let element = document.getElementById('myElement');

  // 点击事件的回调函数是一个闭包：即使外部函数执行完毕，它仍能引用 largeData 变量
  element.addEventListener('click', function() {
    // 打印大数据字符串的长度（演示闭包对外部变量的访问）
    console.log(largeData.length);
  });
}

// 执行函数，完成事件监听器的绑定
attachEventListener();
```
- 在这个例子中，只要 element 存在并且其点击事件监听器没有被移除，largeData 变量就会一直被闭包引用，始终占据内存，即使这个元素可能永远不会被点击。

##### 循环中的陷阱

这是一个经典的闭包相关面试题，尤其是在 var 时代。
- **问题描述**：在循环中创建多个函数（例如事件监听器），而这些函数都关闭了同一个变化的循环变量
- **示例 （错误的行为）**：
```javascript
for (var i = 1; i <= 5; i++) {​  
  setTimeout(function timer() {​    
    console.log(i);​ 
  }, i * 1000);​
 }
```
- 很多人期望这段代码会每隔1秒依次输出1, 2, 3, 4, 5。但实际结果是，它会每隔1秒输出一次 6，总共输出5次。
- 原因：setTimeout 里的函数是闭包，它们都共享了同一个全局作用域（或函数作用域）下的变量 i。当 setTimeout 的回调函数开始执行时，for 循环早已结束，此时 i 的值已经变成了 6。

##### 解决方案：
- 使用 `let` 或 `const` (ES6 推荐)：let 会为循环的每次迭代创建一个新的块级作用域，所以每个闭包都会引用到一个独立的 `i`
- 使用立即执行函数表达式 (IIFE) (ES5 解决方案)：通过`IIFE`为每次循环创建一个新的函数作用域，将当前的`i` 值作为参数传入。

## 如何避免由闭包引起的内存泄漏？

理解了原因后，防范就变得有章可循了。

#### 1、​手动解除引用

在不再需要闭包时，手动将其引用的变量设置为 `null`。这样可以切断引用链，让垃圾回收机制能够正常工作。

```javascript
// 手动解除对闭包的引用
function createLeakyClosure() {​  
let largeData = { /* ... a large object ... */ };​​ 
 return function() {​   
   // ... use largeDatareturn largeData;​  
  }
 ​}​
 ​let myClosure = createLeakyClosure();​
 ​// ... 使用 myClosure ...
 //  当你确定不再需要
 ​myClosure = null; // 解除引用，largeData 就可以被回收了
```

#### 2、谨慎处理DOM引用和事件监听器

这是在前端开发中最常见的场景。
- **及时移除事件监听器**：当DOM元素被销毁或不再需要交互时，一定要用 `removeEventListener` 移除绑定的事件处理函数
- 现代前端框架（如 React, Vue）的生命周期钩子（或 `useEffect` 的清理函数）为我们提供了很好的机制来处理这类清理工作。

**React 示例：**
```javascript
// 导入 React 核心库及所需 Hooks
import React, { useEffect, useState } from 'react';

// 定义函数式组件 MyComponent
function MyComponent() {
  // 副作用钩子：处理滚动事件的绑定与销毁
  useEffect(() => {
    // 滚动事件处理函数（闭包：可访问组件作用域内的变量/方法）
    const handleScroll = () => {
      console.log('scrolling');
    };

    // 组件挂载时：为 window 绑定滚动事件监听
    window.addEventListener('scroll', handleScroll);

    // 清理函数：组件卸载时执行，用于销毁副作用
    return () => {
      // 移除滚动事件监听，防止内存泄漏
      window.removeEventListener('scroll', handleScroll);
      console.log('Event listener removed!');
    };
  }, []); // 空依赖数组：仅在组件挂载（首次渲染）和卸载时执行一次

  // 组件渲染的 DOM 结构
  return <div>My Component</div>;
}

// 导出组件，供其他模块引入使用
export default MyComponent;
```

#### 3、避免不必要的闭包
如果一个函数不需要访问外部变量，就不要把它放在会形成闭包的位置。虽然现代JavaScript引擎优化得很好，但养成良好的编码习惯总是有益的。

### 总结
- **闭包** 是一个函数和其词法环境的组合，它允许函数访问其外部作用域的变量。​
- **优点**：可以创建私有变量、数据封装、实现柯里化等高级函数式编程模式。​
- **潜在问题**：主要是**内存泄漏**（因为外部变量无法被回收）和**循环中的变量**引用问题。​
- **避免内存泄漏的方法**：在不需要时**手动解除引用 (= null)**，以及**及时移除事件监听器**。
