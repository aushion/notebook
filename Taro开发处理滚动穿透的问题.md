## Taro开发处理滚动穿透的问题

### 1.问题

在taro开发小程序中经常会用到蒙层，模态框之类的交互，而用到模态框这种交互的时候通常都回遇到滚动穿透这个问题。本文就是针对这个问题的探讨，以及解决方案。



### 2.方案

那么如何解决问题。众所周知，在H5环境下我们可以在模块框弹出来的时候，可以给body加一个overflow:hidden的样式。当浮层关闭时再把这个样式去掉。在Taro开发中，如果你是用Taro开发H5那么这个方法依然有效。但是如果是Taro开发微信小程序，由于小程序无法直接获取到dom，那么就得换个方法给组件外层加样式了。我们翻看Taro官方文档发现在滚动穿透这一点上给了解决方案：[阻止滚动穿透](https://taro-docs.jd.com/taro/docs/react-overall/#%E9%98%BB%E6%AD%A2%E6%BB%9A%E5%8A%A8%E7%A9%BF%E9%80%8F)

如果我们直接给外层组件加上overflow:hidden的样式的话，而且页面高度超过一屏的话，虽然滚动穿透解决了，但是加了样式之后，页面会回到顶部。所以我们需要再给页面加样式的同时，记下此时页面滚动的距离。当我们关闭蒙层的时候，我们取消overflow的样式同时，也让页面滚动回到我们刚才记下的距离。虽然不那么优雅，体验不那么好，但是总比回不到原始位置要体验好一点吧。。。

### 3.代码

Taro官方文档上有说给view加上 catchMove属性，用来阻止滚动穿透， 但是我试了根本没用，也不知道是不是我用的姿势不对。下面来说具体代码

需要用到的Taro API



### [selectViewport](https://taro-docs.jd.com/taro/docs/apis/wxml/SelectorQuery#selectviewport)

选择显示区域。可用于获取显示区域的尺寸、滚动位置等信息。

> [参考文档](https://developers.weixin.qq.com/miniprogram/dev/api/wxml/SelectorQuery.selectViewport.html)





```tsx
Taro.createSelectorQuery().selectViewport().scrollOffset(function (res) {
  res.id      // 节点的ID
  res.dataset // 节点的dataset
  res.scrollLeft // 节点的水平滚动位置
  res.scrollTop  // 节点的竖直滚动位置
}).exec()
```



**[eventCenter](https://taro-docs.jd.com/taro/docs/apis/about/events)**

同时 Taro 还提供了一个全局消息中心 `Taro.eventCenter` 以供使用，它是 `Taro.Events` 的实例

```jsx
import Taro from '@tarojs/taro'
// EventCenter
Taro.eventCenter.on
Taro.eventCenter.trigger
Taro.eventCenter.off
```

具体做法是，在需要阻止滚动穿透的页面添加监听阻止滚动穿透事件。 我们在打开浮层时，trigger触发消息，并通过第二个参数把需要的滚动距离以及是否阻止滚动穿透的状态，通过消息传到目标页面。目标页面通过参数判断是否需要添加样式，取消样式并回到初始滚动位置。

首先给页面添加一个阻止滚动穿透的样式

```css
.preventScroll {
  overflow: hidden;
  height: 100vh;
}
```

随后编写自定义hooks



```javascript
export const usePreventScroll = () => {
  const [isPreventScroll, setIsPreventScroll] = useState(false)
  // 这个值用于记录页面滚动的距离
  const pageScrollTop = useRef(0)
  useDidShow(() => {
    // 监听浮层弹出发送的preventScroll消息，然后对响应的状态处理
    Taro.eventCenter.on('preventScroll', prevent => {
      // 此消息接受参数，包括滚动状态，滚动的距离
      const { pageScroll, scrollTop } = prevent
			// 如果pageScroll为true说明此时浮层弹出，那么把页面的滚动距离记录下来
      if (pageScroll) pageScrollTop.current = scrollTop
      // 如果pageScroll为false说明浮层关闭，此事后需要让页面的滚动距离从顶部回到初始位置
      if (pageScroll === false) {
        setTimeout(() => {
          Taro.pageScrollTo({
            scrollTop: pageScrollTop.current,
          })
        }, 100)
      }
      // 更新isPreventScroll用于页面添加取消阻止滚动的样式
      setIsPreventScroll(pageScroll)
    })
  })
  useDidHide(() => {
          Taro.eventCenter.off('preventScroll')
  })
// 返回一个bool值用于页面判断是否需要添加阻止滚动的样式
  return isPreventScroll
}
```



组件里浮层弹出时发送消息

```javascript
Taro.createSelectorQuery()
              .selectViewport()
              .scrollOffset(function (res) {
                Taro.eventCenter.trigger('preventScroll', {
                  pageScroll: true,
                  scrollTop: res.scrollTop,
                })
              })
              .exec()   
          
        
```

浮层关闭时发送取消消息

``` javascript
Taro.eventCenter.trigger('preventScroll', { pageScroll: false })
```



页面上使用

```react
import { usePreventScroll } from '@utils/utils/hook'
const cx = classNames.bind(styles)

const Index = () => {
  const isPreventScroll = usePreventScroll()
  return <View className={cx({prventScroll: isPrventScroll})}>
    					...
        </View>
  
}
```

#### 效果

![Dec-13-2021 11-08-31](/Users/ifeng/Desktop/Dec-13-2021 11-08-31.gif)
