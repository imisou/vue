# 过滤器

#### 作用 ：

用于一些常见的文本格式化

#### 使用方式：

过滤器可以用在两个地方：双花括号插值和 v-bind 表达式 (后者从 2.1.0+ 开始支持)。过滤器应该被添加在 JavaScript 表达式的尾部，由“管道”符号指示：

```html
<!-- 在双花括号中 -->
{{ message | capitalize }}

<!-- 在 `v-bind` 中 -->
<div v-bind:id="rawId | formatId"></div>
```

## 创建过滤器的方式

1. Vue.filter('id',function(){})  全局过滤器定义
2. 组件中 filters : { 'id' : function(){} } 组件内部过滤器

## 源码分析

### 一、编译阶段

#### parse阶段

我们发现对于过滤器的使用方式有两种：
- 在属性中 v-bind:id="xxx | filterA"
- 在文本双花括号插值中  {{xxx | filterA | filterB(arg1,arg2)}}

##### 1. 属性中 v-bind:id="xxx | filterA"

在parse处理开始节点的processAttrs() 中发现了 通过bindRE.test(name)去匹配响应式的属性，然后通过 parseFilters(value) 去解析值中的过滤器。
```js
if (bindRE.test(name)) { // v-bind
    // 获取属性的名称  移除 : | v-bind:
    name = name.replace(bindRE, '')
        //  处理value 解析成正确的value
    value = parseFilters(value)
    isProp = false

}

```

###### compiler\parser\filter-parser.js

```js
/**
 * 表达式中的过滤器解析 方法
 * @param {*} exp
 */
export function parseFilters(exp: string): string {
    // 是否在 ''中
    let inSingle = false
    // 是否在 "" 中
    let inDouble = false
    // 是否在 ``
    let inTemplateString = false
    //  是否在 正则 \\ 中
    let inRegex = false
    // 是否在 {{ 中发现一个 culy加1 然后发现一个 } culy减1 直到culy为0 说明 { .. }闭合
    let curly = 0
    // 跟{{ 一样 有一个 [ 加1 有一个 ] 减1
    let square = 0
    // 跟{{ 一样 有一个 ( 加1 有一个 ) 减1
    let paren = 0
    //
    let lastFilterIndex = 0
    let c, prev, i, expression, filters

    for (i = 0; i < exp.length; i++) {
        prev = c
        c = exp.charCodeAt(i)
        if (inSingle) {
            //  '  \
            if (c === 0x27 && prev !== 0x5C) inSingle = false
        } else if (inDouble) {
            // " \
            if (c === 0x22 && prev !== 0x5C) inDouble = false
        } else if (inTemplateString) {
            //  `
            if (c === 0x60 && prev !== 0x5C) inTemplateString = false
        } else if (inRegex) {
            // 当前在正则表达式中  /开始
            //  / \
            if (c === 0x2f && prev !== 0x5C) inRegex = false
        } else if (
            // 如果在 之前不在 ' " ` / 即字符串 或者正则中
            // 那么就判断 当前字符是否是 |
            //  如果当前 字符为 |
            // 且下一个（上一个）字符不是 |
            // 且 不在 { } 对象中
            // 且 不在 [] 数组中
            // 且不在  () 中
            // 那么说明此时是过滤器的一个 分界点
            c === 0x7C && // pipe
            exp.charCodeAt(i + 1) !== 0x7C &&
            exp.charCodeAt(i - 1) !== 0x7C &&
            !curly && !square && !paren
        ) {
            /*
                如果前面没有表达式那么说明这是第一个 管道符号 "|"


                再次遇到 | 因为前面 expression = 'message '
                执行  pushFilter()
             */

            if (expression === undefined) {
                // first filter, end of expression
                // 过滤器表达式 就是管道符号之后开始
                lastFilterIndex = i + 1
                // 存储过滤器的 表达式
                expression = exp.slice(0, i).trim()
            } else {
                pushFilter()
            }
        } else {
            switch (c) {
                case 0x22:    
                    inDouble = true;
                    break // "
                case 0x27:
                    inSingle = true;
                    break // '
                case 0x60:
                    inTemplateString = true;
                    break // `
                case 0x28:
                    paren++;
                    break // (
                case 0x29:
                    paren--;
                    break // )
                case 0x5B:
                    square++;
                    break // [
                case 0x5D:
                    square--;
                    break // ]
                case 0x7B:
                    curly++;
                    break // {
                case 0x7D:
                    curly--;
                    break // }
            }
            if (c === 0x2f) { // /
                let j = i - 1
                let p
                    // find first non-whitespace prev char
                for (; j >= 0; j--) {
                    p = exp.charAt(j)
                    if (p !== ' ') break
                }
                if (!p || !validDivisionCharRE.test(p)) {
                    inRegex = true
                }
            }
        }
    }

    if (expression === undefined) {
        expression = exp.slice(0, i).trim()
    } else if (lastFilterIndex !== 0) {
        pushFilter()
    }

    // 获取当前过滤器的 并将其存储在filters 数组中
    //  filters = [ 'filterA' , 'filterB']
    function pushFilter() {
        (filters || (filters = [])).push(exp.slice(lastFilterIndex, i).trim())
        lastFilterIndex = i + 1
    }

    if (filters) {
        for (i = 0; i < filters.length; i++) {
            expression = wrapFilter(expression, filters[i])
        }
    }

    return expression
}
```

解析过滤器的方法其实很简单：

1. 将属性的值从前往后开始一个一个匹配，关键符号 : "|" 并排除 ""、 ''、 ``、 //、 || (字符串、正则)中的管道符号 '|' 。

如：

```html
<div v-bind:id="message + 'xxx|bbb' + (a||b) + `cccc` | filterA | filterB(arg1,arg2)"></div>
```
字符一个一个往后匹配 如果发现 ` " ' 说明在字符串中，那么直到找到下一个匹配的才结束， /一样 同时 匹配 () {} [] 这些需要两边相等闭合 那么 | 才有效，
最后一个条件排除 || 即可

2. 所以上面直到遇到 第一个正确的 | ，那么前面的表达式 并存储在 expression 中，后面继续匹配再次遇到 | ,那么此时 expression有值， 说明这不是第一个过滤器 pushFilter() 去处理上一个过滤器

```js

/**
    生成过滤器的 表达式字符串

    如上面的
    exp = message
    filters = ['filterA','filterB(arg1,arg2)']

    第一步  以exp 为入参 生成 filterA 的过滤器表达式字符串  _f("filterA")(message)

    第二步 以第一步字符串作为入参 生成第二个过滤器的表达式字符串 _f("filterB")(_f("filterA")(message),arg1,arg2)

    => _f("filterB")(_f("filterA")(message),arg1,arg2)

 * @param {string} exp   上一个过滤器的值 没有就是 表达式的值
 * @param {string} filter
 * @returns {string}
 */
function wrapFilter(exp: string, filter: string): string {
    // 判断是否存在入参， 即 'filterB(arg1,arg2)'
    const i = filter.indexOf('(')
    if (i < 0) {
        // 如果不是  直接生成  "_f("filterA")(message)"
        // _f: resolveFilter
        return `_f("${filter}")(${exp})`
    } else {
        // 过滤器名称
        const name = filter.slice(0, i)
        // 过滤器自定义入参
        const args = filter.slice(i + 1)
        // 生成 "_f("filterB")(message,arg1,arg2)"
        return `_f("${name}")(${exp}${args !== ')' ? ',' + args : args}`
    }
}
```

此时 exp = message + 'xxx|bbb' + (a||b) + `cccc`  , filter = filterA

1. 继续判断 过滤器是否存在 ()， 此时不存在, 那么filter就是名称 第一个入参就是前面的 exp。

生成
```js
"_f("filterA")(message + 'xxx|bbb' + (a||b) + `cccc`)"
```

2. 以前面的结果为exp , 发现存在 ( , 然后生成

```js
"_f("filterB")(_f("filterA")(message + 'xxx|bbb' + (a||b) + `cccc`),arg1,arg2)"
```

##### 2. 文本双花括号插值中 {{ message | capitalize }}

文本的处理是在 parse中的chars()方法 其中存在一个解析 {{}} 的方法 parseText()

```js

export function parseText(
    text: string,
    delimiters ? : [string, string]
): TextParseResult | void {
    // 处理 文本内容  如：
    //  {{obj.name}} is {{obj.job}}
    while ((match = tagRE.exec(text))) {
        // ' {{obj.name}} is {{obj.job}} '  => [ 0: '{{obj.name}}' , 1: 'obj.name' ,index : 1, input: ' {{obj.name}} is {{obj.job}} ']
        // match.index 获取当前匹配的 开始下标
        index = match.index
        // push text token
        // 如果 {{ }}的前面存在 静态的文本   如   (空格..{{xx}} xxx {{}})那么需要将这些静态文本保存
        if (index > lastIndex) {

            rawTokens.push(tokenValue = text.slice(lastIndex, index))
            // 将静态文本保存在 tokens
            tokens.push(JSON.stringify(tokenValue))
        }
        // tag token
        // 解析过滤器
        const exp = parseFilters(match[1].trim())
        //生成当前参数在Vue render中获取响应式数据的方法  _s('obj.name')  => this['obj.name']
        tokens.push(`_s(${exp})`)

    }

}
```
发现 其也是通过const exp = parseFilters(match[1].trim()) 去处理 {{}}中的额过滤器。

## render阶段

我们发现在编译节点如果遇到过滤器 会将其编译成 _f(){}的表达式

```js
"_f("filterB")(_f("filterA")(message + 'xxx|bbb' + (a||b) + `cccc`),arg1,arg2)"
```

###### core\instance\render-helpers\resolve-filter.js

```js
/**
 * Runtime helper for resolving filters
 */
export function resolveFilter (id: string): Function {
  return resolveAsset(this.$options, 'filters', id, true) || identity
}

```
其还是通过resolveAsset去获取 vm.$options的filters中相同的过滤器

![image](https://note.youdao.com/yws/public/resource/fa4a717e0bafc76404a2b7658a9371c6/xmlnote/0F9FFE2D6B484C9891F082ECB7D5BD0F/9090)

然后将 _f("filterA")(message + 'xxx|bbb' + (a||b) + `cccc`),arg1,arg2 作为入参。

### 重点:

1. 过滤器的解析方法 parseFilters()
