title: 实现React国际化
date: 2018-04-26 15:00:00
categories: jiangxiang
tags:
    - React
    - 国际化
---

React最早由Facebook的软件工程师Jordan Walke创建，它在2011年首次部署在Facebook的新闻源中，由于其性能优势明显，很快获得了广泛关注和大规模的使用，如今发展已经非常成熟。
基于React的渲染原理可以实现很多有意思的功能，例如实现一个React的国际化工具。(React v16.3)

<!--more-->

## 一、现状
### 1.1 目前React国际化常用的解决方案有以下三种

1.打包时翻译
2.动态翻译
3.生成多个版本

比较流行的为webpack团队基于打包时翻译的webpack-i18-plugin和yahoo团队基于动态翻译的React-intl

### 1.2 对比

1.打包时翻译
优点:方案比较成熟，已有成功案例
缺点:翻译表一旦更新，需要重新打包发布，本地要维护大量的翻译表文件，过于繁琐
2.动态翻译
优点:灵活，翻译表放进cdn一句sql可以更新，可操作性强
缺点:兼容性有待考察，对于不同的项目结构要设置特有的配置
3.多版本
优点:产生的打包文件最小，无需配置
缺点:修改过程复杂，应用场景不广泛

## 二、大概的设想

### 3.1 翻译语法

语法分为批量翻译、单句翻译
通过Provider包裹或通过date-translate属性实现两种形式的翻译
```
// 批量翻译
<Provider value={{
    languageMap, // 语言包
    language, // 语言类型
}}>
    ... (要翻译的代码块)
</Provider>
// 单句翻译
<div data-translate>...(要翻译的字符串)</div>
```

### 3.2 方便获取在线的语言包

可以通过请求获得在线语言包，在下载完成后进行更新

## 三、核心思路

实现多语言通俗来说就是：
1.先找到要翻译的文字
2.请求在线语言包
3.再对要翻译的文字进行翻译
三步

基于以上基本思路再思考，我们都知道，JSX是react的语法糖，而使用高阶函数React.createElement可以重新定义组件的渲染，我们只需要将要翻译的文字在该方法中处理即可。

以下两种写法经babel转义，本质上没有区别！

```
class Hello extends React.Component {
  render() {
    return React.createElement('div', null, `Hello ${this.props.toWhat}`);
  }
}

ReactDOM.render(
  React.createElement(Hello, {toWhat: 'World'}, null),
  document.getElementById('root')
);
```
基于这个想法，通过设置data-translate属性，<Provider/>，并在createElement中判断，就知道哪里的文字需要翻译，也知道要翻译成哪种语言了（第一步）！
(第二步比较简单，暂时忽略...)
再定义一个translate函数，接受文字，语言包和翻译表，返回翻译后的文字（第三步）

下面的流程图展示了整体的思路：

![](/images/process.png)


## 四、实现

毕竟是要做一个工具库，首先新建一个工程，在core文件目录下创建具体文件

### 4.1 定义一个languageMap

languageMap一定得是单例模式，方便全局调用
代码放在./core/languageMap.js

### 4.2 再写一个translate函数

```
function translate(content, languageMap, language){}
```
任何需要翻译的地方，都将会使用这个函数
它接受三个参数 content:要翻译的文字
             languageMap:语言包
             language:语言类型

代码如下：
```
const translate = (content, languageMap, language) => {
  if (!content) {
    return "";
  }
  if (!language) {
    throw new Error("you should define specific language type!");
  }
  // translate words
  if (language && languageMap && languageMap[language]) {
    languageMap[language][content] &&
      (content = languageMap[language][content]);
  }
  return content;
};
```
代码放在./core/translate.js中以备使用

### 4.3 再创建translateClass

为了使用Context API，我们需要一个辅助类来接受Provider传过来的languageMap和language等字段

先创建一个./core/context.js
```
export const { Provider, Consumer } = React.createContext();
```
在创建一个./core/translateClass
```
class Translate extends React.Component {
  render() {
    return (
      <Consumer>
        {context =>
          translate(this.props.children, context.languageMap, context.language)
        }
      </Consumer>
    );
  }
}
```
代码放在./core/translateClass.jsx中

### 4.4 改写createElement

有了之前的铺垫，可以很快的实现我们自己的createElement
```
import React, { Component } from "react";
import { translate } from "./translate";
import Translate from "./translateClass";
import { Consumer } from "./context";

const createElement = React.createElement;

React.createElement = (...args) => {
  console.log(args);
  let children = args.slice(2);

  children = children.map(child => {
    if (typeof args[0] === "string" && typeof child === "string") {
      return <Translate>{child}</Translate>;
    }
    return child;
  });

  return createElement(args[0], args[1], ...children);
};
```

可以看到一个ReactDom，包含了$$typeof,props,ref等属性，将其children获取到，进行如下判断：
1.如果类型是Object就不做任何操作
2.如果child类型是字符串就进行翻译

通过改变Provider的位置，可以设置不同区块的context，这样可以改变不同区块的翻译参数

代码放在./core/translateTag.jsx
如上，translate工具类基本完成

## 五、写一个页面进行测试

```
import React, { Component } from "react";
import ReactDom from "react-dom";
import languageMap from "./core/languageMap";
import { translate, middleware } from "./core/translate";
import { Provider } from "./core/context";
import "./core/translateTag.jsx";

const styles = {
  fontFamily: "sans-serif",
  textAlign: "center"
};

class App extends Component {
  constructor(...args) {
    super(...args);
  }

  componentWillMount() {
    this.fetchTranslateList();
  }

  fetchTranslateList() {
    languageMap["en"] = {
      你好: "Hello",
      国际化: "intl"
    };
    languageMap["france"] = {
      你好: "Bonjour",
      国际化: "intl"
    };
  }

  render() {
    const enConfig = { language: "en", languageMap };
    const franceConfig = { language: "france", languageMap };
    return (
      <div style={styles}>
        <Provider value={enConfig}>
          <div>
            <h1>你好</h1>
            <span>
              你好
              <Provider value={franceConfig}>
                <h1>你好</h1>
                <span>国际化</span>
              </Provider>
            </span>
          </div>
        </Provider>
      </div>
    );
  }
}

ReactDom.render(
  <App />,
  document.body.appendChild(document.createElement("div"))
);
```
以上代码，我们创建了一个入口文件，分别将中文翻译成了英文和法文，最终代码运行没有问题，证明工具可以使用！

## 六、兼容性

写这个工具的初衷当然是无缝兼容各种react项目，但事实上并不简单
举例来说，由于目前出现了很多前端流行的组件库：
1.ant-design/ant-mobile（蚂蚁金服团队的前端UI组件库，链接：https://ant.design/index-cn）
2..Element of react（饿了么团队的前端UI组件库react版本，链接：https://eleme.github.io/element-react/）
等等...
实际操作中发现了很多不兼容的问题，想必要做到开箱即用是不可能了~~~
寄希望于无缝兼容 不如提供中间件接口来让使用者自行配置 o(╥﹏╥)o

### 6.1 ant-design
例如，在ant-design组件库中，input的placeholder属性，如不进行检测，将不会被翻译

```
import {filter} from './core/translate.js';

filter.forAntDesign = (props,language) => {
    let props = Object.assign({}, props);
            if (language && props.placeholder) {
                if (languageMap[language]) {
                    if (/[^\u4e00-\u9fa5]/g.test(props.placeholder)) {
                        props.placeholder = props.placeholder.replace(/([\u4e00-\u9fa5]+)/g, (match) => {
                            return languageMap[language][match] ? languageMap[language][match] : match
                        })
                    } else {
                        languageMap[language][props.placeholder] && (props.placeholder = languageMap[language][props.placeholder]);
                    }
                }
            }
            return <input {...props} ref="input" data-translated onInput={e => {this.value = e.target.value; this.props.onInput && this.props.onInput(e);}}>{this.props.children}</input>
}
```
ant-design中 Select组件的选中时对比，约定value===children.text时且key=== 所以如果翻译的话 也要将value一起翻译，这些都可以通过增加中间件，修改props.children的过滤规则处理

### 6.2 也有一些框架或者自行编写的组件，如.Element for react，一些dom文字并不会放在text中，这就需要额外的编写特殊的filter了

## 七、总结
到这里，我们的React国际化工具已经实现了，这里是一个简单的在线demo
<iframe src="https://codesandbox.io/embed/64zx97v9qr" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>
完整代码参考 github: https://github.com/ranrantu/react-i18

总之，这可以说是react动态翻译的一种思路，这种实现至今还是有不足的地方，还请各位拍砖指正，感谢！