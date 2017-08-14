---
title: String Join
date: 2017-07-07 18:01:39
tags: python
---
### 简单字符串
直接使用"+"，
例如：
```
full_name = prefix + name
```
### 复杂字符串
有格式化需求时，使用"%"进行格式化连接，
例如：
```
result = "result is %s:%d" % (name, score)
```

### 大量字符串拼接
发生在循环体里时，使用"str.join"进行连接，
例如：
```
result = ''.join(names_tuple)
```

## 总结
前两条出于代码美观考虑，后一条出于性能考虑
