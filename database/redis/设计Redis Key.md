-   Date: 2022-08-19
-   Author: Michael
-   Keyword: #redis 

# 设计Redis Key
Redis中存在大量的Key，而且Key之间不存在联系，为了更好的管理大量的缓存，需要对Key的命名进行规范，要求实现**简洁**、**高效**、**可维护**。

## 内容要求
1. **以英文字母开头**。
2. **建议只是用小写字母、数字、冒号、点号**。
3. **不要包含特殊字符**
4. **使用冒号分割不同类的描述信息，使用`.`分割单词**
5. **key不要太长，能缩写就缩写**，尽量使用[embstr](Redis数据类型.md#OBJ_ENCODING_EMBSTR)编码。

## 命名模板
`服务名:业务逻辑名:{uid}:type`。

- 服务名：可以用缩写，减少字符占用。
- 业务逻辑名：可以是多层级的，可以在其中加入uid。
- uid：具体业务ID。
- type：表示数据类型，提高可读性。

### 正例
```
account:user:uid:123:string

account:user.info:123:hash
```