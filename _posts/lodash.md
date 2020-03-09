https://www.lodashjs.com/docs/latest

pick and omit

```javascript
// 剔除规则
var object = { 'a': 1, 'b': '2', 'c': 3 };
 
_.omitBy(object, _.isNumber);
// => { 'b': '2' }
```