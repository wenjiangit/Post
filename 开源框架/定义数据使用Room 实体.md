# 使用Room Entity定义数据

使用`Room` 持久化库时,可将相关字段的集合定义为实体.对于每个实体，在关联的数据库对象中创建一个表来保存这些条目。

> **注意** :使用实体在你的`app` 内,[添加架构组件依赖](https://developer.android.google.cn/topic/libraries/architecture/adding-components.html)到你`app` 的`build.gradle` 文件中.

默认的,`Room` 为实体中定义的每一个字段创建一个列,如果实体中包含你不想持久化的字段,你可以使用[@Ignore](https://developer.android.google.cn/reference/android/arch/persistence/room/Ignore.html)注解它们,你必须通过`Database` 类中的实体数组引用实体类。

以下的代码片段显示了怎么去定义一个实体:

```java
@Entity
class User {
    @PrimaryKey
    public int id;

    public String firstName;
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

为了持久化一个字段,`Room`必须有访问它的权限.你可以用`public`修饰一个字段,或者为它提供`set`和`get`方法.如果你使用`get` 和`set`方法，请记住这是基于`Room`中`javabean`的约定。

> **注意** :实体可以有一个空构造函数(如果相应的`DAO` 类可以访问每个持久化字段)或者一个构造函数的参数包含与实体类的字段匹配的类型和名称.`Room` 也能使用全部或部分构造函数,例如一个构造函数只接收一些字段.

## 使用主键

每一个实体必须至少定义一个字段作为主键.即使该实体只有一个字段,你仍然需要使用[@PrimaryKey](https://developer.android.google.cn/reference/android/arch/persistence/room/PrimaryKey.html)注解这个字段.并且,如果你想`Room` 为实体自动分配`ID` ,你可以设置`@PrimaryKey's`  [`autoGenerate`](https://developer.android.google.cn/reference/android/arch/persistence/room/PrimaryKey.html#autoGenerate()) 属性.如果实体具有复合主键,你可以使用 [`@Entity`](https://developer.android.google.cn/reference/android/arch/persistence/room/Entity.html) 的 [`primaryKeys`](https://developer.android.google.cn/reference/android/arch/persistence/room/Entity.html#primaryKeys()) 属性,如下面的代码片段所示:

```java
@Entity(primaryKeys = {"firstName", "lastName"})
class User {
    public String firstName;
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

默认的,`Room` 使用类名作为数据库的表名.如果你想给表设置不同的名字,给[`@Entity`](https://developer.android.google.cn/reference/android/arch/persistence/room/Entity.html) 注解设置[`tableName`](https://developer.android.google.cn/reference/android/arch/persistence/room/Entity.html#tableName()) 属性,如以下代码片段所示:

```java
@Entity(tableName = "users")
class User {
    ...
}
```

> **警告** :表名在`SQLite` 数据库中是大小写敏感的.

与`tableName` 属性类似,`Room` 使用字段名字作为数据库表的列名.如果你想给一列设置不同的名字,给这个字段添加

 [`@ColumnInfo`](https://developer.android.google.cn/reference/android/arch/persistence/room/ColumnInfo.html) 注解,如下代码片段所示:

```java
@Entity(tableName = "users")
class User {
    @PrimaryKey
    public int id;

    @ColumnInfo(name = "first_name")
    public String firstName;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

## 注释索引和唯一性

取决于你如何访问你的数据,你可能想给数据库的某些字段添加索引来加快你的查询速度.为实体添加索引,可以在`@Entity` 注解中包含 [`indices`](https://developer.android.google.cn/reference/android/arch/persistence/room/Entity.html#indices()) 属性,列出要包含在索引或复合索引中的列的名称,以下代码片段演示了这个注释过程:

```java
@Entity(indices = {@Index("name"),
        @Index(value = {"last_name", "address"})})
class User {
    @PrimaryKey
    public int id;

    public String firstName;
    public String address;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

有时，数据库中的某些字段或字段组必须是唯一的.您可以通过将`@Index`注解的唯一属性设置为`true`来强制执行此`unique` 属性,下面的代码示例防止表中有两行包含`firstName`和`lastName`列的相同值集合：

```java
@Entity(indices = {@Index(value = {"first_name", "last_name"},
        unique = true)})
class User {
    @PrimaryKey
    public int id;

    @ColumnInfo(name = "first_name")
    public String firstName;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

## 定义对象之间的关系

因为`SQLite` 是一个关系数据库，所以你可以指定对象之间的关系。即使大多数对象关系映射库允许实体对象相互引用,`Room` 明确禁止这一点.了解这个决定背后的技术推理,查看 [理解Room为什么不允许对象引用](https://developer.android.google.cn/training/data-storage/room/referencing-data.html#understand-no-object-references). 

即使您不能使用直接关系，`Room` 仍允许您定义实体之间的外键约束。

例如，如果有另一个名为`Book` 的实体，则可以使用`@ForeignKey` 注释来定义它与`User` 实体的关系，如以下代码片段所示：

```java
@Entity(foreignKeys = @ForeignKey(entity = User.class,
                                  parentColumns = "id",
                                  childColumns = "user_id"))
class Book {
    @PrimaryKey
    public int bookId;

    public String title;

    @ColumnInfo(name = "user_id")
    public int userId;
}
```

外键非常强大，因为它们允许您指定在引用实体更新时发生的情况.例如，如果通过在`@ForeignKey`注解中包含`onDelete = CASCADE`来删除用户的相应实例，则可以让`SQLite`删除用户的所有书籍。

> **注意 **: `SQLite`将`@Insert（onConflict = REPLACE）`作为一组`REMOVE`和`REPLACE`操作处理，而不是单个`UPDATE`操作,这种替换冲突值的方法可能会影响您的外键约束。有关更多详细信息，请参阅[SQLite documentation](https://sqlite.org/lang_conflict.html) `ON_CONFLICT`子句的`SQLite`文档。

## 创建嵌套的对象

有时候，您希望在数据库逻辑中将实体或简单的旧`Java` 对象`（POJO）`表示为一个有凝聚力的整体,即使对象包含几个字段.在这些情况下，您可以使用`@Embedded `注解来表示要分解到表中的子字段的对象,然后，您可以像查看其他单个列一样查询嵌入字段。

例如，我们的`User` 类可以包含一个`Address` 类型的字段，它表示一个名为`street`，`city`，`state`和`postCode` 的字段的组合。在表格中分别存储组成的列,请在`User` 类中包含一个用`@Embedded` 注解的`Address` 字段，如下面的代码片段所示：

```java
class Address {
    public String street;
    public String state;
    public String city;

    @ColumnInfo(name = "post_code")
    public int postCode;
}

@Entity
class User {
    @PrimaryKey
    public int id;

    public String firstName;

    @Embedded
    public Address address;
}
```

表示`User`对象的表格包含具有以下名称的列：`id`，`firstName`，`street`，`state`，`city`和`post_code`。

> **注意** : 嵌入字段也可以包含其他嵌入字段。

如果实体具有多个相同类型的嵌入字段，则可以通过设置`prefix` 属性来保持每个列的唯一性。然后，`Room` 会将提供的值添加到嵌入对象中每个列名的开头。