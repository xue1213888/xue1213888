# typescript学习笔记，一篇包含所有知识点的笔记


> 本篇笔记是阅读阮一峰老师的《TypeScript 教程》总结出来的笔记
> 
> 笔记正在更新中，预计26天完成，一天一篇。

# 1. 基本用法

`2024年04月18日凌晨三点总结`

## 1. 类型声明

声明方式：`冒号+类型`

```typescript
let foo:string;

let foo:string = 'hello world!';

function toString(num: number):string {
    return String(num);
}
```

## 2. 类型推断

类型声明不是必要的，如果没有，TypeScript会自己推断。

```typescript
let foo = 123;  // 推断为 number

function toString(num: number) { // 返回值推断为 string
    return String(num)
}
```

## 3. TypeScript的编译

浏览器和 `Node.js` 不认识 `TypeScript` 。所以提前需要把 `TypeScript` 编译为 `JavaScript`。这个过程叫做编译。所以 `TypeScript` 是编译时类型检查，编译后就不会检查了。

## 4. 值与类型

注意要分清楚 `值` 和 `类型` 。

`TypeScript` 代码只涉及类型，不涉及值，所有和值有关的其实都是 `JavaScript` 完成的。

编译时，系统也仅仅是将 `TypeScript` 也就是 `类型` 全部拿掉。

## 5. Playground

 [TypeScript Playground](http://www.typescriptlang.org/play/)

## 6. tsc 编译器

将 `.ts` 文件编译为 `.js` 的工具。

### 6.1 安装

```bash
## 下面命令是全局安装，你也可以仅在项目中进行安装 tsc 依赖模块
npm install -g typescript


## 获取当前 tsc 版本
tsc -v


## 获取帮助信息
tsc -h

## 查看完整帮助信息
tsc --all

```

### 6.2 编译

```bash
tsc app.ts

## 同时编译多个文件
tsc file1.ts file2.ts file3.ts

## outFile 多个文件编译输出到一个文件
tsc file1.ts file2.ts file3.ts --outFile app.js

## outDir 指定编译到指定目录
tsc app.ts --outDir dist

## target 默认会编译为 es3 版本，推荐指定 es2015
tsc --target es2015 app.ts
```

### 6.3 编译错误处理

如果没有错误，`tsc` 命令不会有任何显示。

编译出现错误会报错，同时也会生成相应的 `js` 文件，开发者应该自己考虑是不是符合预期。

如果你想在报错时停止编译，请使用`noEmitOnError`参数

```bash
tsc --noEmitOnError app.ts
```

还有一个只检查是否正确，无论正确与否都不进行编译 `noEmit`。

```bash
tsc --noEmit app.ts
```

### 6.4 tsconfig.json

该文件可以将 `tsc` 命令写到 `json` 文件中，这样就不用每次写 `tsc` 命令时写一长串。

例如：

- 方法1
  
  - ```bash
    tsc file1.ts file2.ts --outFile dist/app.js
    ```

- 方法2 
  
  - ```json
    {
      "files": ["file1.ts", "file2.ts"],
      "compilerOptions": {
        "outFile": "dist/app.js"
      }
    }
    ```
  
  - ```bash
    tsc
    ```
    
    

## 7. ts-node 模块

`node` 环境不支持直接运行 `ts` 文件，可以使用 `ts-node` 。

```bash
npm install -g ts-node

ts-node script.ts
```

如果不安装 `ts-node` 也可以使用 `npx` 来调用

```bash
npx ts-node script.ts
```

也可以像 `node` 一样，起一个交互窗口进行互动。

```bash
$ ts-node
> const twice = (x:string) => x + x;
> twice('abc')
'abcabc'
> 
```

退出使用 `Ctrl + d`。或者输入 `.exit` 。



# 2. any类型，unknown类型，never类型

`2024年4月18日中午1点总结`

## 1. any 类型

### 1.1 基本含义

`any` 是顶层类型，是包含了所有类型的父集。

`any` 类型可以存入任何值。如果某个变量设置为 `any` 。 `TypeScript` 会关闭该变量的类型检查。

```typescript
let x: any;

// 当 x 是 any 的时候，下面语句都不报错
x = 1;
x = 'foo';
x = true;
x(2);
x.foo = 100;
```

尽量避免使用 `any` 类型。

### 1.2 类型推断

只要没明确写类型，`TypeScript` 又推断不了的类型，全部都是 `any` 。

使用 `noImplicitAny` 选项，可以让编译器在推断为 `any` 类型时报错。

- **注意：明确写 any 的将不受影响**

- **注意：使用 `let` 和 `var` 声明变量但不赋值也不明确写变量类型，这种情况开启 `noImplicitAny` 也不会报错**

### 1.3 污染问题

`any` 类型可以合理的赋值给其他任意类型而不会触发编译检查。所以可能会污染已经明确类型的变量，导致部分错误只能等运行阶段才能得到发现。

```typescript
let x:any = 'hello';
let y:number;

y = x;

y*123;

y.toFixed()
```

## 2. unknown 类型

也是顶层类型，可以存入任意类型的值。

- 不能赋值给其他类型的变量(除了`any` 类型和 `unknown` 类型)，解决了 `any` 类型污染的问题。

- 不能直接调用 `unknown` 类型变量的方法和属性。

- 运算限制，只能进行比较运算 `==`、`===`、`!=`、`!==`、`||`、`&&`、`?`、`!`、`typeof`、`instanceof`这几种。

`unknown` 使用类型缩小到明确的类型后才能使用该真实类型变量的方法和属性，以及对其进行运算操作。

```typescript
let a:unknown = 1;

a += 1; // 报错

if (typeof a === 'number') { // 不报错
    let r = a + 10;
}

// -----------------------
let s:unknown = 'hello';

if (typeof s == 'string') { // 正确
    s.length;
}
```

## 3. never 类型

类型为空，不包含任何值，即不可能存在的值。对其赋值会报错。

`never` 类型主要是保证类型运算的完整性。

当我们写联合类型，例如 `string|number` 在对其进行类型缩小时，如果不满足 `string` 和 `number` 类型，那么就是 `never` 类型。

```typescript
function fn(x:string|number) {
  if (typeof x === 'string') {
    // ...
  } else if (typeof x === 'number') {
    // ...
  } else {
    x; // never 类型
  }
}
```

`never` 可以赋值给任意其他类型。

`any` 和 `unknown` 是顶层类型，包含了所有类型，`any` 可以赋值给任意类型，`unknown` 可以复制给 `unknown` 和 `any` 类型。

`never` 是底层类型，被所有类型包含着。所以 `never` 类型可以赋值给任意类型。

# 3. TypeScripe 的类型系统

## 1. 基本类型

### 1.1 概述

### 1.2 boolean 类型

### 1.3 string 类型

### 1.4 number 类型

### 1.5 bigint 类型

### 1.6 symbol 类型

### 1.7 object 类型

### 1.8 undefined 类型、null 类型

## 2. 包装对象类型

### 2.1 包装对象的概念

### 2.2 包装对象类型与字面量类型

## 3. Object 类型与 object 类型

### 3.1 Object 类型

### 3.2 object 类型

## 4. undefined 和 null 的特殊性

## 5. 值类型

## 6. 联合类型

## 7. 交叉类型

## 8. type 命令

## 9. typeof 运算符

## 10. 块级类型声明

## 11. 类型的兼容



















