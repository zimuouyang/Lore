## ResultType 与ResultMap

ResultType相对与ResultMap而言更简单一点。只有满足ORM（Object Relational Mapping，对象关系映射）时，即数据库表中的字段名和实体类中的属性完全一致时，才能使用，否则会出现数据不显示的情况。

```mysql
 <select id="Sel" resultType="com.example.entity.User">
        select * from user where id = #{id}
    </select>
```

ResultMap和ResultType的功能类似，但是ResultMap更强大一点，ResultMap可以实现将查询结果映射为复杂类型的pojo。

```mysql
<resultMap id="ModelMap" type="com.example.entity.Model">
        <result column="age" jdbcType="INTEGER" property="age" />
        <result column="name" jdbcType="VARCHAR" property="name" />
    </resultMap>

    <select id="getModels" resultMap="ModelMap">
        select * from model
    </select>

```

- ResultMap标签的id属性是唯一的，和select标签的resultMap一致
- type属性是pojo（普通的JavaBean对象）的包名加类名，用来封装信息。如果mybatis里面配置了别名包，也就是给包起别名，那么type里面直接写类名就可以了。
- ResultMap中的id标签是用来描述表中的主键的对应关系，column用来描述数据库表中的主键字段名，property用来描述pojo中的属性名。
- result标签是用来描述表中的普通字段的对应关系，column用来描述数据库表中的普通字段名，property用来描述pojo中的属性名。
- association标签用来实现一对一的关系
- collection标签用来实现一对多的关系

## RequestParam 与 PathVariable

可以在@RequestMapping注解中用{ }来表明它的变量部分

```java
@RequestMapping(value="/user/{username}")
```

在路由中定义变量规则后，通常我们需要在处理方法（也就是@RequestMapping注解的方法）中获取这个URL的具体值，并根据这个值（例如用户名）做相应的操作，SpringMVC提供的@PathVariable可以帮助我们：

```java
@RequestMapping(value="/user/{username}")
    public String userProfile(@PathVariable(value="username") String username) {
    	return "user"+username;
    }
```

除了简单地定义{username}变量，还可以定义正则表达式进行更精确地控制，定义语法是{变量名: 正则表达式}。[a-zA-Z0-9_]+是一个正则表达式，表示只能包含小写字母，大写字母，数字，下划线。如此设置URL变量规则后，不合法的URL则不会被处理，直接由SpringMVC框架返回404NotFound。

```java
@RequestMapping(value = "/user/{username: [a-zA-Z0-9]+}/blog/{blogId}")
```

### @RequestParam

在访问各种各样的网站时，经常会发现网站的URL的最后一部分形如：?xx=yy&zz=ww。这就是HTTP协议中的Request参数

```java
@RequestMapping(value="/user")
	public String getUserBlog(@RequestParam(value="id") int blogId) {
		return "blogId="+blogId;
	}
```

@RequestParam和@PathVariable都能够完成类似的功能——因为本质上，它们都是用户的输入，只不过输入的部分不同，一个在URL路径部分，另一个在参数部分。要访问一篇博客文章，这两种URL设计都是可以的

- 通过@PathVariable，例如/blogs/1
- 通过@RequestParam，例如blogs?blogId=1

那么究竟应该选择哪一种呢？建议：

1、当URL指向的是某一具体业务资源（或资源列表），例如博客，用户时，使用@PathVariable

2、当URL需要对资源或者资源列表进行过滤，筛选时，用@RequestParam

例如我们会这样设计URL：

- /blogs/{blogId}
- /blogs?state=publish而不是/blogs/state/publish来表示处于发布状态的博客文章