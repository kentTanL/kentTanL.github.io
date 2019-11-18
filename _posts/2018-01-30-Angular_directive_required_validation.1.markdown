---
layout:     post
title:      "[Angular Directive] 输入框禁止为空字符串与自动去除空格指令"
date:       2018-01-30 19:46:00
author:     "Kent"
header-img: "img/post-bg-2016.jpg"
catalog: true
tags:
    - Angular
    - Directive
---

{:toc}

## 一、前言

input 输入框自带了`required`属性，用以表单验证，但是只要有字符，即使全为空格也能通过`required`验证，这无法满足一些应用场景，所以需要自定义一些指令，用来满足验证全为空格的输入。

在使用自定义的 Directive 修改 input 输入框值或属性时，需要注意：

1. **请尽量使用 Angular 提供的类或方法来修改输入框的值， 以免`ngModel`无法同步；**
2. 同上，使用`FormControl`而非`ElementRef`来更新输入框的值；
3. 构造函数中使用`NgControl`获取输入框的引用，而非直接使用`FormControl`


## 二、实例

- 实例使用版本：angular 5.1.0

### 1. 验证空字符串

通过添加自定义的`Validator`，实现验证空字符串，不合法的输入返回`{'required': true}`对象，等同于原生的在 input 输入框添加 required 属性后的不合法输入的返回结果。

```js
import { Directive, ElementRef, HostListener, Input, Renderer } from '@angular/core';
import { FormGroup, FormControl, NgControl } from '@angular/forms';

@Directive({
  selector: '[input-required]'
})
export class InputRequiredDirective {

    constructor(private elementRef: ElementRef, private control : NgControl) {
        if (control && control.control) {
            control.control.setValidators((c: FormControl) => {
                let v = c.value;
                if (!v || v.trim() == '') {
                    return {'required': true};
                } 
                return null;
            });
        }
    }
    
}
```

#### 使用方式

```html
<input type="text" [(ngModel)]="name" name="name" input-required />
```

### 2. 替换左边空格

替换左边空格，其实也意味可以在字符串右侧输入空格，却无法全输入空格，但是这种方法有一个缺陷，即如果一直按住空格，然后点击表单的提交，也能绕过输入框的`required`验证，所以最好是配合上面的`input-required`指令一起使用。

同时请注意，这里使用`FormControl.setValue(...)`方法，可以触发`ngModel`的更新，如果使用 js 或 ElementRef 提供的方式，将会出现`ngModel`无法同步更新的后果。

```js
import { Directive, ElementRef, HostListener, Input, Renderer } from '@angular/core';
import { FormGroup, FormControl, NgControl } from '@angular/forms';

@Directive({
  selector: '[input-ltrim]'
})
export class InputLeftTrimDirective {

    constructor(private elementRef: ElementRef, private control : NgControl) {

    }

    @HostListener("keyup", ["$event", "$event.target"]) 
    keyupFun(evt, target) {
        if (target.value) {
            this.control.control.setValue(this.ltrim(target.value));
        } 
    }

    ltrim(str) {
        return str.replace(/(^\s*)/g,"");
    }

}
```

#### 使用方式

```html
<input type="text" [(ngModel)]="name" name="name" input-ltrim />
```

### 3. 禁止输入空格

禁止输入空格，即当用户按下空格键时便阻止输入，但是如果只是这样，那么用户仍然可能使用粘贴的方式输入空格，所以这里同时在`keyup`事件中将所有空格替换了。

```js
import { Directive, ElementRef, HostListener, Input } from '@angular/core';
import { FormGroup, FormControl, NgControl } from '@angular/forms';

@Directive({
  selector: '[input-noSpace]'
})
export class InputNoSpaceDirective {

    constructor(private elementRef: ElementRef, private control : NgControl) {

    }

    @HostListener("keydown", ["$event"]) 
    keydownFun(evt) {
        if (evt.key.trim() == '') {
            evt.preventDefault();
        }
    }
    
    @HostListener("keyup", ["$event", "$event.target"]) 
    keyupFun(evt, target) {
        if (target.value) {
            this.control.control.setValue(target.value.replace(/(\s*)/g, ""));
        } 
    }
    
}
```

#### 使用方式

```html
<input type="text" [(ngModel)]="name" name="name" input-noSpace />
```

## 二、配置

上面省略了配置，这里一并贴出。

**1. 在`directives`目录下的`index.ts`中输入如下信息：**

```js
import { InputLeftTrimDirective } from './input-trim/input-ltrim.directive'
import { InputNoSpaceDirective } from './input-trim/input-noSpace.directive'
import { InputRequiredDirective } from './input-required/input-required.directive'

export const DIRECTIVES : Array<any> =[
    InputLeftTrimDirective,
    InputNoSpaceDirective,
    InputRequiredDirective,
]
```

**2. 在`@NgModel`中配置如下（不相关部分被省略）：**

```js

import { DIRECTIVES } from './directives';

@NgModule({
    declarations: [
        ..._DIRECTIVES,
    ],
}

```

## 三、参考链接

- [https://stackoverflow.com/questions/40682717/angular-2-input-directive-modifying-form-control-value](https://stackoverflow.com/questions/40682717/angular-2-input-directive-modifying-form-control-value)

## 四、最后

在实际的开发中，因为刚开始对于 Directive 并不熟，所以对于在构造函数中传入`NgControl`并不了解，导致踩了很多坑，但是通过 debug，意外的发现了可以自定义设置效验器，以及其它的一些问题，也算是一种额外的收获吧，以上。