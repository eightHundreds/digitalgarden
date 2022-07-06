---
{"dg-publish":true,"permalink":"///babel//transform-runtime/","dgHomeLink":true,"dgPassFrontmatter":false}
---


# 简介

默认情况下，在babel编译过程中会插入一些辅助代码来实现一些语法。并且是每个文件都会插入。存在重复代码。
我们可以借助`@babel/runtime`和`@babel/plugin-transform-runtime`来减少这些代码。

- `@babel/runtime`包含了 `regenerator`(实现生成器语法)，和其他helper方法
- `@babel/plugin-transform-runtime`用在编译时的插件，将辅助代码改成从`@babel/runtime`  import helper代码

# Pollyfill

他们支持pollyfill功能（默认关闭）， 与preset-env不同的是，它们以import的方式引入pollyfill，不会出现原型链污染问题（更适合**库开发**）。


注意, 如果你打算用它的pollyfill，需要配置corejs(默认关，可选配置是`2`和`3`)，同时需要将`@babel/runtime`改成`@babel/runtime-corejs2`或`@babel/runtime-corejs3`


## 缺点

### 无脑编译

对于
``` js
const a = {
	replaceAll: () => '1',
};

a.replaceAll();
```

会编译成

``` js
import _replaceAllInstanceProperty from "@babel/runtime-corejs3/core-js/instance/replace-all";
var a = {
  replaceAll: function replaceAll() {
    return '1';
  }
};

_replaceAllInstanceProperty(a).call(a);
```

js 无法知晓a的类型，只会无脑编译。


# 配置

## corejs

默认关，可配置`2`,`3`，分别代表使用corejs2和3
注意：corejs2 不支持内建api，如`''.replaceAll()`

## helpers

是否开启helper方法导入，默认开

## version
runtime的版本。默认是`7.0.0`
一些helper的引入是有版本要求 [babel/helpers.ts](https://github.com/babel/babel/blob/b67e369e85/packages/babel-helpers/src/helpers.ts)

推荐配置成当前安装的版本, 例如你安装了`@babel/runtime-corejs2@7.7.4` , 就配置成`^7.7.4` 这样可以减少编译后应用的大小.



# 实战

简单配置，项目会享受到babel helper代码减少的优势
``` JSON
{
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "version": "^7.7.4"
      }
    ]
  ]
}
```


## pollyfill

需要安装`@babel/runtime-corejs3`，需要关闭preset-env的pollyfill。

```
{
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "version": "^7.7.4",
        "core": 3
      }
    ]
  ]
}
```

> [!note] Tip
> 特别的是，它不需要像preset-env那样要精确地设置core-js版本
> 
