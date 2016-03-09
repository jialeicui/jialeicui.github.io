---
layout: post
title:  typeahead.js 中使用别名匹配
date:   2016-03-09
categories: js
---

[typeahead.js](http://twitter.github.com/typeahead.js/) 是一个自动补全库, 使用文档和demo可以在 [Github](https://github.com/twitter/typeahead.js) 项目中看到

我的需求里需要支持类似于别名的功能, 比如输入 [flash](https://movie.douban.com/subject/25662329/), 会自动匹配 `树懒`, 这里基于官方demo修改了一下, 实现别名的功能

```html
<div id="the-basics">
  <input class="typeahead tt-hint" type="text">
</div>
```

```javascript
var substringMatcher = function(dicts) {
  return function findMatches(q, cb) {
    var matches, substringRegex;
    function matchInArray(reg, arr) {
      var ret = false;
      if (!arr) {
        return ret;
      }

      $.each(arr, function(i, str) {
        if (reg.test(str)) {
          ret = true;
          return;
        }
      });
      return ret;
    }
    matches = [];
    if (q === '') {
      // 默认显示全部
      $.each(dicts, function(k, v){
        matches.push(k);
      });
      cb(matches);
    } else {
      substrRegex = new RegExp(q, 'i');
      $.each(dicts, function(k, v) {
        if (substrRegex.test(k) || matchInArray(substrRegex, v)) {
        matches.push(k);
        }
      });
      cb(matches);
  }
  };
};
//建议值:[别名列表]
var states = {
  '建议1':['foo', 'bar'],
  '建议2':'',
  '建议3':['test'],
  '拼音':['py', 'pinyin'],
  '树懒':['flash']
};

function customTokenizer(obj) {
  return obj.tags.concat(Bloodhound.tokenizers.obj.whitespace('val'));
}

$('#the-basics .typeahead').typeahead({
  hint: true,
  highlight: true,
  minLength: 0 // 默认显示全部
},
{
  name: 'states',
  source: substringMatcher(states)
});
```

[demo](/demo/typeahead/index.html)

