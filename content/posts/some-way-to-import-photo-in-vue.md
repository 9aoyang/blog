+++
title = "Vue项目中引入图片资源的几种方式"
categories = ["Vue"]
tags = ["Vue"]
date = 2019-01-29
+++

## 按照常规语法引用路径

### HTML

```html
<img src="../../path/to/image.png" />
```

### CSS

```css
background-image: url('../../path/to/image.png');
```

### JavaScript

```js
// import
import image from '../../path/to/image.png';
// require
require('../../path/to/image.png');
```

这种方式是最常见的，但这种引入方式存在很高的维护成本。在`webpack`中，可以通过配置`alias`来简化路径，同时由于指定了更精确的路径，可以降低构建的文件搜索耗时，进而提高构建效率。

```js
module.exports = {
  //...
  resolve: {
    alias: {
      path: path.resolve(__dirname, '../../path')
    }
  }
};
```

这样，js 中的写法就可以变为

```js
// import
import image from 'path/to/image.png';
// require
require('path/to/image.png');
```

如果再借助`css-loader`，`css`中的写法就可以是

> To import assets from a node_modules path (include resolve.modules) and for alias, prefix it with a ~:

```css
background-image: url('~path/to/image.png');
```

进一步借助`vue-html-loader` 和 `css-loader`

> vue-html-loader and css-loader translates non-root URLs to relative paths. In order to treat it like a module path, prefix it with ~:

非根路径会被`vue-html-loader`和`css-loader`转化为相对路径，如果要想被识别为模块路径，就在路径前加上前缀`~`。

```html
<img src="~path/to/image.png" />
```

## 在 vue 中使用`v-bind`绑定图片`src`属性：

首先，在 data 中定义好图片路径

```js
path: require('path/to/image.png');
```

然后，在 template 模板里

```html
<img :src="path" />
```
