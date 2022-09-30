# 使用mobx之后, 页面性能居然提升了10倍

## 问题描述
我写了一个JSON渲染器, 用来将JSON渲染成一个页面。页面渲染逻辑如下
![image](https://user-images.githubusercontent.com/18495604/84284743-2e04af80-ab6f-11ea-84ea-d86a82c25e27.png)
1. `local/remote`变量替换
2. 解析`remote`, 发送http请求，从服务端获取数据，存到数据中心`pagestate`
3. 递归调用`React.createElement`方法, 创建`ReactNode Tree`
4. 更新`DOM Tree`
5. `pagestate`的变化, 将会重复第三、四步

在渲染器上层，还有一个可视化编辑器(不久将开源)，用于在线编辑页面。

一个简单的增、删、改、查页面，使用可视化编辑器配置之后，再由JSON渲染器实时渲染出来。配置了1000多个页面，也没有用户报性能问题。直到将页面使用微前端嵌入到其他项目时，有一个用户给我报了个性能问题，如下图所示，页面非常卡顿

![image](https://user-images.githubusercontent.com/18495604/84284364-a7e86900-ab6e-11ea-86b5-a739d287f5d2.png)

通过chrome Performance调试工具，很容易发现，CPU内存空间被占满了，出现了渲染阻塞。

## 原因分析
渲染器源码大概是这样的
```jsx
class JSONRender extends React.Component {

  /**
   * 解析props上的ReactNode配置, 渲染为ReactNode节点
   *
   * @param { any } v 待解析的属性
   * @param { String } dataPagestatePath 属性相对于pagestate的路径, 例如 'elementConfig.children[0].props.extra'
   * @returns { any } 解析后的值, 如果为ReactNode格式({ type: "**", props: {}, children: [] }), 则调用creatElement方法渲染为ReactNode节点
   *
   * @example const v = {
   * @example             props: {
   * @example               extra: { type: "Button", props: {}, children: ['新建页面'] }
   * @example             }
   * @example           }
   * @example const dataPagestatePath = 'elementConfg.children[0].extra'
   * @example const result = parseProps(v, dataPagestatePath)
   * @example //result => { props: { extra: <Button data-pageconfig-path="elementConfig.children[0].props.extra" pagestate={pagestate}/> }}
   */
  parseProps = (v: any, dataPagestatePath: string): any => {
    try {
      if (isPlainObject(v)) {
        // 如果为一个reactNode配置, 则渲染为一个ReactNode节点
        if (isElementConfig(v)) {
          return this.createElement( v,  { key: v.props.key || dataPagestatePath }, dataPagestatePath)
        }

        let formatV = {}
        for (let key in v) {
          if (Object.prototype.hasOwnProperty.call(v, key)) {
            formatV[key] = this.parseProps( v[key], `${dataPagestatePath}.${key}`)
          }
        }
        return formatV
      }

      if (isArray(v)) {
        return v.map((val: any, i: number) => {
          return this.parseProps(val, `${dataPagestatePath}[${i}]`)
        })
      }

      return v
    } catch (e) {
      setTimeout(() => {
        throw e
      })
    }
  }

  /**
   * 创建一个ReactNode节点
   *
   * @param { Object } elementConfig ReactNode配置
   * @param { Object } defaultProps 默认属性
   * @param { String } dataPagestatePath 相对于PageState的路径, 例如 pagestate = { elementConfig: { props: 'a' }}, 'a'相对于pagestate 的路径为 `elementConfig.props.a`
   * @returns { ReactNode } ReactNode节点, 具有data-pageconfig-path和pagestate属性
   *
   * @example const elementConfig = { type: "Button", props: {}, "children": ["提交"] }
   * @example const Node = createElement(elementConfig, {}, 'elementConfig.props.button')
   * @example // Node => <Button data-pageconfig-path="elementConfig.props.button" pagestate={pagestate}>提交</Button>
   */
  createElement = ( elementConfig, defaultProps, dataPagestatePath = '' ) => {
    // 空字符串, null, undefined, 0, false
    if (!elementConfig) return elementConfig

    if (!isElementConfig(elementConfig)) {
      // string/number/boolean
      if ( isString(elementConfig) ||  isNumber(elementConfig) || isBoolean(elementConfig)) {
        return elementConfig
      }

      // ReactNode
      if (ReactIs.isElement(elementConfig))  return elementConfig
      
      // array/function/object
      throw new TypeError('element格式错误:' + JSON.stringify(elementConfig))
    }

    const pagestate = this.getPageState()

    const { type, props = {}, children = [], } = elementConfig 

    let _props = this.parseProps(props, `${dataPagestatePath}.props`)

    const mergeProps = { ..._props, ...defaultProps, pagestate, key: dataPagestatePath }

    let _children
    if (isArray(children) && children.length) {
      _children = children.map((child, index) => {
        if (isPlainObject(child)) {
          return this.createElement(child, {},`${dataPagestatePath}.children[${index}]`)
        }
        return child
      })
    }
    const { Component } = this.getComponent(type)

    return React.createElement( Component, mergeProps,  _children )
  }

  /**
   * 解析element配置, 渲染为ReactNode节点
   * @param { Object } elementConfig 页面节点配置
   * @param { Object } pagestate 作用域, 数据中心
   * @returns { React.ReactNode } ReactNode
   */
  renderElement = (elementConfig, pagestate) => {
    // 解析JSON模版, 变量替换
     const _elementConfig = parseJsonTemplate(elementConfig, pagestate)
     return this.createElement(_elementConfig, {}, 'elementConfig')
  }

  render(){
    const { pagestate, elementConfig } = this.state
    return this.renderElement(elementConfig, pagestate)
  }

}

```

React 的更新策略, 在不设置`shouldComponentUpdate`的情况下, 只要props或者state变化, 就会调用`render`方法, 更新组件。

在我的渲染器(JSONRender)里面, `render`方法调用了一个计算量特别大的`renderElement`方法, 用来生成`ReactNode Tree`. 

在上面的情况中，用户每次按下键盘按键时，都会触发pagestate更新，每次更新都会调用`renderElement`方法，很快CPU就被打满了，页面表现得非常卡顿。

## 解决办法
思路: 如果每次pagestate更新, 只重新计算变化的部分，将大大减少计算量。

很快，我找到了`mobx`([详情](https://cn.mobx.js.org/)) 它的特性让人眼前一亮:
* 能够监听数据的变化（与vue一致的响应式更新原理）
* 使用方便，API简洁 （与redux相比）
* 性能极高 

`mobx-react` 在`mobx`和`React`的基础上进行了封装, 提供了高阶函数`Observer`, 可以作为装饰器或普通函数使用。使用Observer 装饰的组件, 会自动收集依赖数据，自动根据所依赖数据的变化，来更新视图.

引入mobx-react, 对代码做了修改:
* 将数据中心从根组件的state中, 移到了组件外部, 使用一个局部变量store存储数据
* 使用observer装饰器装饰渲染器JSONRender
* 对每次递归创建的子组件都用observer包一层

```
import { action, autorun, computed, observable, runInAction, toJS } from 'mobx'`
import { observer } from 'mobx-react'

/**
 * 数据中心
 * 以页面路由为维度, 存储数据
 */
 const store = observable({})

@observer
class JSONRender extends React.Component {

  parseProps = (v, dataPagestatePath) => { ...  }

  createElement = ( elementConfig, defaultProps, dataPagestatePath = '' ) => { 
     ...
     // return React.createElement( Component, mergeProps,  _children )
    return React.createElement(observer(Component), mergeProps,  _children )
  }

  renderElement = (elementConfig, pagestate) => {
    // 解析JSON模版, 变量替换
     const _elementConfig = parseJsonTemplate(elementConfig, pagestate)
     return this.createElement(_elementConfig, {}, 'elementConfig')
  }

  render(){
      const { elementConfig, pagestate } = store
      return this.resolveElementConfig(elementConfig, pagestate)
  }

}
```

优化之后，页面的卡顿没有了，输入框值时，只有使用了输入框值的组件会更新。通过chrome Performance工具可以发现，输入响应时间变成了25ms, 页面性能至少提升10倍不止。

## 总结

mobx是一个非常好的状态管理工具，使用简洁，性能优异，社区评价很高，值得一试。
