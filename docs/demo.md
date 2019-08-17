# 缓存自动化

?> 缓存自动化是可插拔的

缓存自动化分为了两部分, 自动缓存, 以及自动清缓存, 你可以根据实际需要去使用它们中的一部分也行.

## 自动缓存

自动缓存的最基本使用例子如下:

```java
@CacheIn(value = "你的缓存名")
public String getDataPermissionSQLByCode(String code) {

}
```

针对需要特定条件的缓存, 我们提供了一些内容变量可以供你使用

| 变量名   |      描述      |
|----------|:-------------:|
| ${self} |  登陆者自身ID |

用起来的时候, 直接把它跟缓存名拼接在一起即可, 系统会自动解析: 

```java
    @CacheIn(value = DATA_PERMISSION + V_SELF)
```

还可以用SpEl表达式哦

```java
@CacheIn(value = DATA_PERMISSION + V_SELF, params =
    {
    "#code"
})
@Override
public String getDataPermissionSQLByCode(String code) {

}
```

## 自动清缓存

自动清缓存的作用机制是在实体进行增删改这种非幂等操作的时候, 会自动清除相应的缓存.

!> 自定义的sql增删改请自行实现

使用方法是在实体上面加上相应的缓存名称

```java
@CacheOut({
    LOGIN_INFO
})
public class User extends BaseEntity {}
```

> 注意使用场景应当是那些真的需要缓存的场景(例如查询慢, 很久变动的)


!> 这个是测试下的啦的

