---
title: 使用React实现自定义的浏览器滚动条
author: Neil Ning
date: 2025-01-17 10:22:57
tags: ["React", "scrollbar"]
categories: 学习
cover: bg.jpg
---
# React自定义滚动条

开发Web应用时，有时候我们会遇到需要自定义滚动条的情况。比如在一个长列表中或者侧边的导航栏中，我们希望滚动条的样式和颜色能够和整体风格保持一致，这时我们就需要自定义滚动条。自定义滚动条的方法大致有两种，比如通过CSS样式和JavaScript代码。

## 通过CSS自定义滚动条
首先，最简单的方案就是通过CSS设置滚动条的样式。我们可以通过`::-webkit-scrollbar`伪类来设置滚动条的样式。可以设置的部分有滚动条有水平滚动条和垂直滚动条，滚动条的轨道，滚动条的滑块等。示例代码如下：
```css
/* 设置滚动条的样式 */
.scrollbar::-webkit-scrollbar {
  width: 10px; /* 设置滚动条的宽度 */
  height: 10px; /* 设置滚动条的高度 */
}

/* 设置滚动条的轨道样式 */
.scroll::-webkit-scrollbar-track {
  background: #f1f1f1;
}

/* 设置滚动条的滑块样式 */
.scroll::-webkit-scrollbar-thumb {
  background: #888;
  border-radius: 5px;
}

/* 设置滚动条的滑块悬停样式 */
.scroll::-webkit-scrollbar-thumb:hover {
  background: #555;
}

/* 设置滚动条的滑块激活样式 */
.scroll::-webkit-scrollbar-thumb:active {
  background: #555;
}

/* 设置水平和垂直滚动条交界处的样式 */
.scroll::-webkit-scroll-corner {
  background: #f1f1f1;
}
```
以上伪类带有-webkit前缀，所以只能在webkit内核的浏览器中生效，比如Chrome、Safari等。对其他浏览器如Firefox和IE，就需要考虑其他方案了，需要JS和CSS配合实现。

## 通过JS和CSS自定义滚动条
由于CSS设置滚动条样式的兼容性较差，所以实际项目中基本无法采用上面的方式。这就需要我们通过JS和CSS配合来实现自定义滚动条。主要思路是创建一个模拟的滚条，然后通过JS来监听滚动事件，动态修改模拟滚动条滑块的位置，然后通过CSS隐藏浏览器原生滚动条。

下面通过React实现一个模拟滚动条，首先通过vite快速初始化一个React项目，我们先从最常见的垂直滚动条开始，创建一个Scroll组件，其基本结构如下：
```jsx
import React, { useEffect, useState, useRef } from 'react';

import './scrollbar.css';

export const Scrollbar = ({children, height}) => {
  const outerRef = useRef(null);
  const innerRef = useRef(null);

  const [scrollHeight, setScrollHeight] = useState(0);

  const handleScroll = (e) => {
    
  };

  return (
    <div className="scrollbar-container" style={{height}}>
      <div className="scrollbar-vertical-track">
        <div className="vertical-thumb" style={{ height: scrollHeight }} />
      </div>
      <div 
        ref={outerRef} 
        className="scrollbar-outer" 
        onScroll={handleScroll} 
      >
        <div className="scrollbar-inner" ref={innerRef}>
          {children}
        </div>
      </div>
    </div>
  );
};
```
scrollbar.css的内容如下：
```css
.scrollbar-container {
  overflow: hidden;
  position: relative;
}

.scrollbar-container:hover .scrollbar-vertical-track  {
  opacity: 1;
}

.scrollbar-vertical-track {
  position: absolute;
  right: 0;
  top: 0;
  height: 100%;
  width: 8px;
  background-color: #ff9d0036;
  opacity: 0;
  transition: opacity 0.25s;
}

.vertical-thumb {
  cursor: pointer;
  width: 8px;
  border-radius: 4px;
  background: gray;
}

.scrollbar-outer {
  height: 100%;
  overflow: auto;
  /* 在元素上滚动时避免外部容器滚动 */
  overscroll-behavior: contain;
}
.scrollbar-inner {
  height: auto;
  width: fit-content;
}
```

然后创建一个demo组件，使用Scrollbar组件，如下：
```jsx  
import { Scrollbar } from './component/Scrollbar';

function App() {
  return (
    <div className='app'>
      <Scrollbar
        height={200}
      >
        <div style={{ width: 1200, height: 1200, background: 'linear-gradient(135deg, #646cff, #f6ff64)' }}></div>
      </Scrollbar>
    </div>
  )
}

export default App
```

## 设置滑块的高度
首先我们要根据容器和内容的高度计算出滑块的高度，在组件中添加以下代码：
```jsx
export const Scrollbar = ({children, height}) => {
  // 其他代码省略...

  const outerRect = useRef({ height: 0, width: 0 });
  const innerRect = useRef({ height: 0, width: 0 });
  const scaleY = useRef(0); // 存储比例，供后面使用
  const [scrollHeight, setScrollHeight] = useState(0);

  useEffect(() => {
    // 1. 计算垂直滑块的高度
    const outer = outerRef.current;
    const inner = innerRef.current;
    if (outer && inner) {
        outerRect.current = { height: outer.clientHeight, width: outer.clientWidth };
        innerRect.current = { height: inner.clientHeight, width: inner.clientWidth };
      
      // 使用百分比计算滚动条的宽度和高度
      setScrollHeight(outer.clientHeight / inner.clientHeight * 100 + '%');
      const thumbHeight = outer.clientHeight / inner.clientHeight * outer.clientHeight;
      scaleY.current = (innerRect.current.height - outerRect.current.height) / (outerRect.current.height - thumbHeight);
    }
  }, [outerRef.current, innerRef.current]);

  return (
    <div className="scrollbar-container" style={{height}}>
      <div className="scrollbar-vertical-track">
        <div className="vertical-thumb" style={{ height: scrollHeight }} />
      </div>
      <div 
        ref={outerRef} 
        className="scrollbar-outer" 
        onScroll={handleScroll} 
      >
        <div className="scrollbar-inner" ref={innerRef}>
          {children}
        </div>
      </div>
    </div>
  );
};
```

## 设置滑块的位置
紧接着要处理滚动事件，在滚动事件中设置模拟滚动条滑块的位置，计算滑块的位置主要通过scrollTop和百分比计算，然后通过translateY来修改滑块的位置，代码如下：
```jsx
export const Scrollbar = ({children, height}) => {
  // 其他代码省略...
  const [translateY, setTranslateY] = useState(0);

  const handleScroll = (e) => {
    // 2. 滚动时计算滑块的位置
    const {height} = outerRect.current;
    // 通过百分比的计算方式最为简单
    const y = e.currentTarget.scrollTop / height * 100 + '%';
    setTranslateY(y);
  };

  return (
    <div className="scrollbar-container" style={{height}}>
      <div className="scrollbar-vertical-track">
        <div className="vertical-thumb" style={{ height: scrollHeight, transform: `translateY(${translateY})` }} />
      </div>
      <div 
        ref={outerRef} 
        className="scrollbar-outer" 
        onScroll={handleScroll} 
      >
        <div className="scrollbar-inner" ref={innerRef}>
          {children}
        </div>
      </div>
    </div>
  );
};
```

## 滑轨点击事件
接下来要处理的另外一个效果是滑轨点击事件，即点击滑轨的某个位置时，能够直接滚动到相应的位置。但是为了能完全还原原生滚动条的效果，我们不使用click事件，而是使用mouseDown事件，即鼠标按下时就执行滚动操作。代码如下：
```jsx
export const Scrollbar = ({children, height}) => {
  // 其他代码省略...

  // 3. 滑轨点击事件
  const handleVerticalTrackMouseDown = (e) => {
    e.stopPropagation();
    const { height: outerH } = outerRect.current;
    const { height: innerH } = innerRect.current;
    // 获取点击时，在滚动条上的位置
    const offset = e.clientY - e.currentTarget.getBoundingClientRect().top;
    const scrollTop = offset / outerH * innerH - outerH / 2;
    outerRef.current.scrollTop = scrollTop;
  };

  // 其他代码省略...

  return (
    <div className="scrollbar-container" style={{height}}>
      <div className="scrollbar-vertical-track" onMouseDown={handleVerticalTrackMouseDown}>
        <div className="vertical-thumb" style={{ height: scrollHeight, transform: `translateY(${translateY})` }} />
      </div>
      <div 
        ref={outerRef} 
        className="scrollbar-outer" 
        onScroll={handleScroll} 
      >
        <div className="scrollbar-inner" ref={innerRef}>
          {children}
        </div>
      </div>
    </div>
  );
};
```

## 滑块拖拽事件
最后要模拟的行为是拖动滑块，执行滚动操作，这里则需要用到mouseDown、mouseMove和mouseUp事件，代码如下：
```jsx
export const Scrollbar = ({children, height}) => {
  // 其他代码省略...

  // 判断是否按下鼠标
  const verticalMouseDown = useRef(false);
  // 记录按下鼠标时指针的位置
  const startY = useRef(0);
  // 记录按下鼠标时，已经滚动的距离
  const startScrollTop = useRef(0);

  // 4.滑块拖拽事件
  const handleVerticalThumbMouseDown = (e) => {
    // 阻止默认行为, 防止选中文本
    e.preventDefault();
    // 阻止事件冒泡, 防止触发滑轨点击事件
    e.stopPropagation();
    verticalMouseDown.current = true;
    startY.current = e.clientY;
    startScrollTop.current = outerRef.current.scrollTop;
    // 在window上增加move事件，鼠标移动出滑轨的区域仍然有效
    window.addEventListener('mousemove', handleMouseMove);
    window.addEventListener('mouseup', handleMouseUp);
  };

  const handleMouseMove = (e) => {
    // 阻止默认行为, 防止选中文本
    e.preventDefault();
    if (verticalMouseDown.current) {
      const offsetY = e.clientY - startY.current;
      outerRef.current.scrollTop = startScrollTop.current + offsetY * scaleY.current;
    }
  };

  const handleMouseUp = (e) => {
    if (verticalMouseDown.current) {
      verticalMouseDown.current = false;
      startY.current = 0;
      // 结束后移除事件监听，防止其他bug产生
      window.removeEventListener('mousemove', handleMouseMove);
      window.removeEventListener('mouseup', handleMouseUp);
    }
  };

  // 其他代码省略...

  return (
    <div className="scrollbar-container" style={{height}}>
      <div className="scrollbar-vertical-track" onMouseDown={handleVerticalTrackMouseDown}>
        <div className="vertical-thumb" onMouseDown={handleVerticalThumbMouseDown} style={{ height: scrollHeight, transform: `translateY(${translateY})` }} />
      </div>
      <div 
        ref={outerRef} 
        className="scrollbar-outer" 
        onScroll={handleScroll} 
      >
        <div className="scrollbar-inner" ref={innerRef}>
          {children}
        </div>
      </div>
    </div>
  );
};
```

## 隐藏真实滚动条
至此模拟垂直滚动条的功能已经全部实现了，但是之前为了观察模拟滚动条的效果，还一直保留有真实滚动条，所以我们要隐藏真实滚动条，滚动条的宽度在各个系统中有差异，所以还需要一个辅助函数计算当前系统滚动条的宽度。
```jsx
const getScrollBarWidth = () => {
  const outer = document.createElement('div');
  outer.style.visibility = 'hidden';
  outer.style.width = '100px';
  outer.style.position = 'absolute';
  outer.style.top = '-9999px';
  document.body.appendChild(outer);

  const widthNoScroll = outer.offsetWidth;
  outer.style.overflow = 'scroll';

  const inner = document.createElement('div');
  inner.style.width = '100%';
  outer.appendChild(inner);

  const widthWithScroll = inner.offsetWidth;
  outer.parentNode.removeChild(outer);

  return widthNoScroll - widthWithScroll;
};
```
计算滚动条宽度的代码原理也很简单，通过向浏览器插入一个真实滚动的容器来计算滚动条的宽度，有了滚动条宽度之后，可以通过将滚动容器的marginRight设置为负值，来达到隐藏垂直滚动条的目的，代码如下：
```jsx
export const Scrollbar = ({children, height}) => {
  // 其他代码省略...

  // 是否需要显示垂直滚动条
  const [shouldShowVerticval, setShouldShowVerticval] = useState(false);
  const [marginX, setMarginX] = useState(0);
  const scrollBarWidth = useRef(0);
 
  useEffect(() => {
    const inner = innerRef.current;
    if (inner) {
      if (height === undefined) {
        setShouldShowVerticval(false);
      } else if (inner.clientHeight <= height) {
        setShouldShowVerticval(false);
      } else {
        setShouldShowVerticval(true);
      }
    }
  }, [innerRef.current, height]);

  useEffect(() => {
    scrollBarWidth.current = getScrollBarWidth();
    setMarginX(scrollBarWidth.current);
  }, []);

  // 其他代码省略

  return (
    <div className="scrollbar-container" style={{height}}>
      {
        shouldShowVerticval && (
          <div className="scrollbar-vertical-track" onMouseDown={handleVerticalTrackMouseDown}>
            <div className="vertical-thumb" onMouseDown={handleVerticalThumbMouseDown} style={{ height: scrollHeight, transform: `translateY(${translateY})` }} />
          </div>
        )
      }
      <div 
        ref={outerRef} 
        className="scrollbar-outer" 
        onScroll={handleScroll} 
        style={{ marginRight: shouldShowVerticval ? `-${marginX}px` : 0 }}
      >
        <div className="scrollbar-inner" ref={innerRef}>
          {children}
        </div>
      </div>
    </div>
  );
};
```

## 水平滚动条
水平滚动条的原理和垂直滚动条的原理基本一致，只是把在Y轴上的计算换成X轴，首先添加模拟的滚动条元素和样式：

```jsx
export const Scrollbar = ({children, height, width}) => {
  // 其他代码省略...

  const scaleX = useRef(0); // 存储比例，供后面使用
  const [translateX, setTranslateX] = useState(0);
  const [scrollWidth, setScrollWidth] = useState(0);

  // 其他代码省略...

  return (
    <div className="scrollbar-container" style={{height, width}}>
      {
        shouldShowVerticval && (
          <div className="scrollbar-vertical-track" onMouseDown={handleVerticalTrackMouseDown}>
            <div className="vertical-thumb" onMouseDown={handleVerticalThumbMouseDown} style={{ height: scrollHeight, transform: `translateY(${translateY})` }} />
          </div>
        )
      }

      <div className="scrollbar-horizontal-track" onMouseDown={handleHorizontalTrackMouseDown}>
        <div className="horizontal-thumb" onMouseDown={handleHorizontalThumbMouseDown} style={{ width: scrollWidth, transform: `translateX(${translateX})` }} />
      </div>
      
      <div 
        ref={outerRef} 
        className="scrollbar-outer" 
        onScroll={handleScroll} 
        style={{ marginRight: shouldShowVerticval ? `-${marginX}px` : 0 }}
      >
        <div className="scrollbar-inner" ref={innerRef}>
          {children}
        </div>
      </div>
    </div>
  );
};
```
水平滚动条样式代码如下：
```css
.scrollbar-container:hover .scrollbar-horizontal-track  {
  opacity: 1;
}

.scrollbar-horizontal-track {
  position: absolute;
  left: 0;
  bottom: 0;
  height: 8px;
  width: 100%;
  background-color: #ff9d0036;
  opacity: 0;
  transition: opacity 0.25s;
}

.horizontal-thumb {
  cursor: pointer;
  height: 8px;
  border-radius: 4px;
  background: gray;
}
```
下面的分别计算滑块的位置，和相应的事件，完整代码如下：
```jsx
export const Scrollbar = ({children, height, width}) => {
  const outerRef = useRef(null);
  const innerRef = useRef(null);
  const outerRect = useRef({ height: 0, width: 0 });
  const innerRect = useRef({ height: 0, width: 0 });

  const scaleY = useRef(0); // 存储比例，供后面使用
  const [translateY, setTranslateY] = useState(0);

  const [scrollHeight, setScrollHeight] = useState(0);

  // 判断是否按下鼠标
  const verticalMouseDown = useRef(false);
  // 记录按下鼠标时指针的位置
  const startY = useRef(0);
  // 记录按下鼠标时，已经滚动的距离
  const startScrollTop = useRef(0);

  // 是否需要显示垂直滚动条
  const [shouldShowVerticval, setShouldShowVerticval] = useState(false);
  const [marginX, setMarginX] = useState(0);
  const scrollBarWidth = useRef(0);

  const scaleX = useRef(0); 
  const [translateX, setTranslateX] = useState(0);
  const [scrollWidth, setScrollWidth] = useState(0);
  const horizontalMouseDown = useRef(false);
  const startX = useRef(0);
  const startScrollLeft = useRef(0);
  const [shouldShowHorizontal, setShouldShowHorizontal] = useState(false);

  const handleScroll = (e) => {
    // 2. 滚动时计算滑块的位置
    const {height} = outerRect.current;
    // 通过百分比的计算方式最为简单
    const y = e.currentTarget.scrollTop / height * 100 + '%';
    setTranslateY(y);

    const x = e.currentTarget.scrollLeft / width * 100 + '%';
    setTranslateX(x);
  };

  // 3. 滑轨点击事件
  const handleVerticalTrackMouseDown = (e) => {
    e.stopPropagation();
    const { height: outerH } = outerRect.current;
    const { height: innerH } = innerRect.current;
    // 获取点击时，在滚动条上的位置
    const offset = e.clientY - e.currentTarget.getBoundingClientRect().top;
    const scrollTop = offset / outerH * innerH - outerH / 2;
    outerRef.current.scrollTop = scrollTop;
  };

  // 4.滑块拖拽事件
  const handleVerticalThumbMouseDown = (e) => {
    // 阻止默认行为, 防止选中文本
    e.preventDefault();
    // 阻止事件冒泡, 防止触发滑轨点击事件
    e.stopPropagation();
    verticalMouseDown.current = true;
    startY.current = e.clientY;
    startScrollTop.current = outerRef.current.scrollTop;
    // 在window上增加move事件，鼠标移动出滑轨的区域仍然有效
    window.addEventListener('mousemove', handleMouseMove);
    window.addEventListener('mouseup', handleMouseUp);
  };

  const handleHorizontalTrackMouseDown = (e) => {
    e.stopPropagation();
    const { width: outerW } = outerRect.current;
    const { width: innerW } = innerRect.current;
    const offset = e.clientX - e.currentTarget.getBoundingClientRect().left;
    const scrollLeft = offset / outerW * innerW - outerW / 2;
    outerRef.current.scrollLeft = scrollLeft;
  };

  const handleHorizontalThumbMouseDown = (e) => {
    // 阻止默认行为, 防止选中文本
    e.preventDefault();
    // 阻止事件冒泡, 防止触发滑轨点击事件
    e.stopPropagation();
    horizontalMouseDown.current = true;
    startX.current = e.clientX;
    startScrollLeft.current = outerRef.current.scrollLeft;
    window.addEventListener('mousemove', handleMouseMove);
    window.addEventListener('mouseup', handleMouseUp);
  };

  const handleMouseMove = (e) => {
    // 阻止默认行为, 防止选中文本
    e.preventDefault();
    if (verticalMouseDown.current) {
      const offsetY = e.clientY - startY.current;
      outerRef.current.scrollTop = startScrollTop.current + offsetY * scaleY.current;
    }
  };

  const handleMouseUp = (e) => {
    if (verticalMouseDown.current) {
      verticalMouseDown.current = false;
      startY.current = 0;
      // 结束后移除事件监听，防止其他bug产生
      window.removeEventListener('mousemove', handleMouseMove);
      window.removeEventListener('mouseup', handleMouseUp);
    }
  };

  useEffect(() => {
    // 1. 计算垂直滑块的高度
    const outer = outerRef.current;
    const inner = innerRef.current;
    if (outer && inner) {
        outerRect.current = { height: outer.clientHeight, width: outer.clientWidth };
        innerRect.current = { height: inner.clientHeight, width: inner.clientWidth };
      
      // 使用百分比计算滚动条的宽度和高度
      setScrollHeight(outer.clientHeight / inner.clientHeight * 100 + '%');
      const thumbHeight = outer.clientHeight / inner.clientHeight * outer.clientHeight;
      scaleY.current = (innerRect.current.height - outerRect.current.height) / (outerRect.current.height - thumbHeight);

      setScrollWidth(outer.clientWidth / inner.clientWidth * 100 + '%');
      const thumbWidth = outer.clientWidth / inner.clientWidth * outer.clientWidth;
      scaleX.current = (innerRect.current.width - outerRect.current.width) / (outerRect.current.width - thumbWidth);
    }
  }, [outerRef.current, innerRef.current]);
 
  useEffect(() => {
    const inner = innerRef.current;
    if (inner) {
      if (height === undefined) {
        setShouldShowVerticval(false);
      } else if (inner.clientHeight <= height) {
        setShouldShowVerticval(false);
      } else {
        setShouldShowVerticval(true);
      }

      if (width === undefined) {
        setShouldShowHorizontal(false);
      } else if (inner.clientWidth <= width) {
        setShouldShowHorizontal(false);
      } else {
        setShouldShowHorizontal(true);
      }
    }
  }, [innerRef.current, height]);

  useEffect(() => {
    scrollBarWidth.current = getScrollBarWidth();
    setMarginX(scrollBarWidth.current);
  }, []);

  return (
    <div className="scrollbar-container" style={{height, width}}>
      {
        shouldShowVerticval && (
          <div className="scrollbar-vertical-track" onMouseDown={handleVerticalTrackMouseDown}>
            <div className="vertical-thumb" onMouseDown={handleVerticalThumbMouseDown} style={{ height: scrollHeight, transform: `translateY(${translateY})` }} />
          </div>
        )
      }

      {
        shouldShowHorizontal && (
          <div className="scrollbar-horizontal-track" onMouseDown={handleHorizontalTrackMouseDown}>
            <div className="horizontal-thumb" onMouseDown={handleHorizontalThumbMouseDown} style={{ width: scrollWidth, transform: `translateX(${translateX})` }} />
          </div>
        )
      }
      
      <div 
        ref={outerRef} 
        className="scrollbar-outer" 
        onScroll={handleScroll} 
        style={{ 
          marginRight: shouldShowVerticval ? `-${marginX}px` : 0,
          height: shouldShowHorizontal ? `${height + marginX}px` : height,
        }}
      >
        <div className="scrollbar-inner" ref={innerRef}>
          {children}
        </div>
      </div>
    </div>
  );
};
```
水平滚动条实现方式和垂直滚动条的方式几乎一致，唯一不同的是隐藏水平滚动条的方式，隐藏水平滚动条需要设置容器的高度。

## 总结
以上就是实现模拟滚动条的全部代码，用到的事件只有mouseDown、mouseMove、mouseUp和scroll等事件，通过这些事件来模拟滚动条的拖拽和点击事件。通过这种方式，我们可以自定义滚动条的样式，实现更加灵活的滚动条效果。不过上面的代码还不支持阿拉伯语的右对齐样式，如果需要支持阿拉伯语，还需要对滑块的位置进行调整。但对于不需要考虑阿拉伯语的项目，上面的代码已经足够了。

## 参考资料
- https://joyran.github.io/yi-blog/blog/scrollbar.html
