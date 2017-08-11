---
title: PrestoDB自定义函数开发指南
date: 2017-01-19 17:51:04
tags:
- 大数据
- presto
categories:
 - 大数据
---

# Presto 自定义函数开发（译）

<font color="red">本文是基于Presto 0.160版本翻译,最新版的内容请参看最新版文档</font>

英文原文:
> https://prestodb.io/docs/current/develop/functions.html



Presto是一个开源的分布式SQL查询引擎，适用于交互式分析查询，数据量支持GB到PB字节。

Presto的设计和编写完全是为了解决像Facebook这样规模的商业数据仓库的交互式分析和处理速度的问题。

Presto本身自带了一些功能函数，对于我们在做SQL查询时非常方便，但是有些时候这些内建的功能函数并不能完全满足我们的需求，又或者是我们逻辑中有一些共性的部分，需要将这些共性的部分提炼出来形成公共的函数库。Presto为我们提供了这套自定义函数的机制。

## 插件实现
该函数框架用于实现SQL功能函数。Presto包含了一定数量的内建函数支持。 为了实现更多的功能函数,你可以自己写一个插件,实现`getFunctions()`,让这个插件实现更多的功能函数:

```java
public class ExampleFunctionsPlugin
        implements Plugin
{
    @Override
    public Set<Class<?>> getFunctions()
    {
        return ImmutableSet.<Class<?>>builder()
                .add(ExampleNullFunction.class)
                .add(IsNullFunction.class)
                .add(IsEqualOrNullFunction.class)
                .add(ExampleStringFunction.class)
                .add(ExampleAverageFunction.class)
                .build();
    }
}
```

注意: `ImmutableSet`是一个Guava类库的一个工具类, `getFunctions()`方法包含了本掼中所有的我们将要实现的功能函数。

如果想要查看完整的代码例子,可以查看presto的功能模块`presto-ml`(用于机器学习)或者`presto-teradata-functions`(Teradata-compatible functions), 以上两个包都是presto源代码包中。

<!--more-->

## 标量函数实现

函数框架使用注解来表示相关的函数信息,如: 名称、描述、返回类型和参数类型。 以下的例子是一个实现`is_null`的功能函数例子:
```java
public class ExampleNullFunction
{
    @ScalarFunction("is_null")
    @Description("Returns TRUE if the argument is NULL")
    @SqlType(StandardTypes.BOOLEAN)
    public static boolean isNull(@SqlNullable @SqlType(StandardTypes.VARCHAR) Slice string)
    {
        return (string == null);
    }
}
```

该函数`is_null`只需要一个`VARCHAR`类型的参数,并且返回一个`BOOLEAN`类型的结果,判断给定的参数是否为`NULL`。 这里需要注意的是提供给函数的参数是一个`Slice`类型。`VARCHAR`使用`Slice`,
而不使用`String`（原生的容器类型）,这个类型是对`byte[]`的包装。

- `@SqlType`:

   `@SqlType`注解用于申明返回类型和参数类型。注意Java代码的参数和返回类型必须和注释中申明的类型一致。

- `@SqlNullable`:

   `@SqlNullable`注解表明参数可以为空。 如果没有这个注解, 任何一个参数为`NULL`,那么该函数都会返回`NULL`。当类型有对应的原始类型时,比如:`BigintType`,如果要使用`@SqlNullable`请使用包装类型,
   当参数不为`NULL`时,如果函数想要返回`NULL`,则必须在函数上申明`@SqlNullable`。

## 参数化标量函数

拥有类型参数的标量函数会增加额外的复杂度。下面我们将演示如果将上面的例子能够适应任何类型:
```java
@ScalarFunction(name = "is_null")
@Description("Returns TRUE if the argument is NULL")
public final class IsNullFunction
{
    @TypeParameter("T")
    @SqlType(StandardTypes.BOOLEAN)
    public static boolean isNullSlice(@SqlNullable @SqlType("T") Slice value)
    {
        return (value == null);
    }

    @TypeParameter("T")
    @SqlType(StandardTypes.BOOLEAN)
    public static boolean isNullLong(@SqlNullable @SqlType("T") Long value)
    {
        return (value == null);
    }

    @TypeParameter("T")
    @SqlType(StandardTypes.BOOLEAN)
    public static boolean isNullDouble(@SqlNullable @SqlType("T") Double value)
    {
        return (value == null);
    }

    // ...and so on for each native container type
}
```

- `@TypeParameter`:

   `@TypeParameter`注解用于申明哪些类型可以被用于`@SqlType`注解或者函数的返回类型。 它也可以用于注解一个参数的类型`Type`. 在运行时,引擎会将正确的类型绑定到该参数上。 `@OperatorDependency`
   可以用于申明一个参数需要另一个额外的功能函数用于操作。 例如,下面的功能函数只会绑定到拥有一个`equals`方法的类型上:

   ```java
   @ScalarFunction(name = "is_equal_or_null")
   @Description("Returns TRUE if arguments are equal or both NULL")
   public final class IsEqualOrNullFunction
   {
       @TypeParameter("T")
       @SqlType(StandardTypes.BOOLEAN)
       public static boolean isEqualOrNullSlice(
               @OperatorDependency(operator = OperatorType.EQUAL, returnType = StandardTypes.BOOLEAN, argumentTypes = {"T", "T"}) MethodHandle equals,
               @SqlNullable @SqlType("T") Slice value1,
               @SqlNullable @SqlType("T") Slice value2)
       {
           if (value1 == null && value2 == null) {
               return true;
           }
           if (value1 == null || value2 == null) {
               return false;
           }
           return (boolean) equals.invokeExact(value1, value2);
       }

       // ...and so on for each native container type
   }
   ```

   另一个标量函数的例子

   下面的`lowercaser`函数需要一个`VARCHAR`参数,并且返回`VARCHAR`的结果, 目的是转换参数字符到小写形式:

   ```java
   public class ExampleStringFunction
   {
       @ScalarFunction("lowercaser")
       @Description("converts the string to alternating case")
       @SqlType(StandardTypes.VARCHAR)
       public static Slice lowercaser(@SqlType(StandardTypes.VARCHAR) Slice slice)
       {
           String argument = slice.toStringUtf8();
           return Slices.utf8Slice(argument.toLowerCase());
       }
   }
   ```

   对于常见的字符串函数（包括转换小写）, Slice类型也提供了直接基于底层`byte[]`的实现来提供更好的性能。 这个功能函数没有使用`@SqlNullable`注解,意味着如果参数为`NULL`,那么返回结果会自动变成`NULL`（函数不会被调用）。

## 聚合函数实现

聚合函数使用和标量函数相似的框架,但是更加复杂一点。

- `AccumulatorState`:

   所有的聚合函数聚集input rows到一个state对象中。这个对象必须实现`AccumulatorState`这个接口。对于简单聚合, 仅仅需求继承`AccumulatorState`来创建一个带getter和setter方法的新接口就可以了, 框架会帮你
   生成实现代码和序列化器。 如果你需要一个更复杂的state对象, 你需要实现`AccumulatorStateFactory`和`AccumulatorStateSerializer`,
   并通过`AccumulatorStateMetadata`这个注解来标注。下面的代码实现了聚合函数`avg_double`用于计算`double`类型的列:
   ```java
   @AggregationFunction("avg_double")
   public class AverageAggregation
   {
       @InputFunction
       public static void input(LongAndDoubleState state, @SqlType(StandardTypes.DOUBLE) double value)
       {
           state.setLong(state.getLong() + 1);
           state.setDouble(state.getDouble() + value);
       }

       @CombineFunction
       public static void combine(LongAndDoubleState state, LongAndDoubleState otherState)
       {
           state.setLong(state.getLong() + otherState.getLong());
           state.setDouble(state.getDouble() + otherState.getDouble());
       }

       @OutputFunction(StandardTypes.DOUBLE)
       public static void output(LongAndDoubleState state, BlockBuilder out)
       {
           long count = state.getLong();
           if (count == 0) {
               out.appendNull();
           }
           else {
               double value = state.getDouble();
               DOUBLE.writeDouble(out, value / count);
           }
       }
   }
   ```
   有两部分: 每一行的sum和总行数. `LongAndDoubleState`一个继承至`AccumulatorState`的接口:
   ```java
   public interface LongAndDoubleState
           extends AccumulatorState
   {
       long getLong();

       void setLong(long value);

       double getDouble();

       void setDouble(double value);
   }
   ```

   就像上面看到的, 对于简单的`AccumulatorState`对象, 只需要简单定义一个接口,并写上getter和setter方法,后面的事就由框架帮你实现了。

深入了解各种注解对于聚合函数开发的用途:

- `@InputFunction`:

   `@InputFunction`注解申明哪个function应该来接收输入的rows并且将它们存储在`AccumulatorState`对象里. 这有点类似于标量函数（你必须给参数指定`@SqlType`）,
   不同的于上面标量的例子（`Slice`用于存储`VARCHAR`）,原始类型`double`用于参数输入, 在这个例子中, 输入函数(input function)简单地跟踪记录的行数（通过setLong()函数）和 总值(通过setDouble()函数)。

- `@CombineFunction`:

   `@CombineFunction`注解用于申明哪个function用于合并两个state对象。该函数用于合并分区的聚合state,它将两个state对象合并到第一个（在上面的例子中, 就是把它们相加）。

- `@OutputFunction`:

   `@OutputFunction`是聚合操作最后一个被调用的function,它携带最终的state对象(所有的分区state的结果)并且将这个结果写入到`BlockBuilder`。
   序列化是在哪里发生的呢? 并且什么是`GroupedAccumulatorState`?

   `@InputFunction`和`@CombineFunction`通常运行在不同的worker机器上, 所以state对象被聚合框架序列化并且在worker机器之间进行传输。`GroupedAccumulatorState`被用于`GROUP BY`聚合,
   并且框架会自动为你生成实现,你不需要指定一个`AccumulatorStateFactory`
