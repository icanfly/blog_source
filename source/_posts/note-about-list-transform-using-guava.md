title: 关于Guava类库中Lists.transform的问题解析
date: 2015-12-30 20:28:00


tags:
 - guava
categories:
 - 问题解析
---

这里讲述是的google的Guava类库中的一个需要注意的问题，如下：
```java
public class GuavaListTest {

	public static class TestDO {
		private String name;
		private int age;
		private String description;

		public TestDO(String name, int age, String description) {
			this.name = name;
			this.age = age;
			this.description = description;
		}

		public TestDO() {
		}

		public String getName() {
			return name;
		}

		public void setName(String name) {
			this.name = name;
		}

		public int getAge() {
			return age;
		}

		public void setAge(int age) {
			this.age = age;
		}

		public String getDescription() {
			return description;
		}

		public void setDescription(String description) {
			this.description = description;
		}
	}

	public static class TestDTO {
		private String name;
		private int age;
		private String description;

		public TestDTO(String name, int age, String description) {
			this.name = name;
			this.age = age;
			this.description = description;
		}

		public TestDTO() {
		}

		public String getName() {
			return name;
		}

		public void setName(String name) {
			this.name = name;
		}

		public int getAge() {
			return age;
		}

		public void setAge(int age) {
			this.age = age;
		}

		public String getDescription() {
			return description;
		}

		public void setDescription(String description) {
			this.description = description;
		}
	}

	public static void main(String[] args) {

		List<TestDTO> retList = query();

		System.out.println(JSON.toJSONString(retList,true));


	}

	private static List<TestDTO> query() {
		List<TestDO> queryList = ImmutableList.of(
				new TestDO("test1", 18, "test obj1"),
				new TestDO("test2", 19, "test obj2"),
				new TestDO("test3", 20, "test obj3")
		);

		List<TestDTO> retList = Lists.transform(queryList, new Function<TestDO, TestDTO>() {
			@Override
			public TestDTO apply(TestDO input) {
				return DTOUtils.createAndCopy(TestDTO.class, input);
			}
		});

		//此处是见证神奇的地方
		for(TestDTO dto : retList){
			dto.setAge(dto.getAge()+1);
		}
		return retList;
	}

}
```

这段代码的输出是什么？

<!--more-->

这段代码的测试本意是将得到的数据做一些处理（这里简单将age+1），然后返回结果。粗看起来应该打印的结果是这样的：
```json
[
	{
		"age":19,
		"description":"test obj1",
		"name":"test1"
	},
	{
		"age":20,
		"description":"test obj2",
		"name":"test2"
	},
	{
		"age":21,
		"description":"test obj3",
		"name":"test3"
	}
]
```

但是实际上的结果是这样的：
```json
[
	{
		"age":18,
		"description":"test obj1",
		"name":"test1"
	},
	{
		"age":19,
		"description":"test obj2",
		"name":"test2"
	},
	{
		"age":20,
		"description":"test obj3",
		"name":"test3"
	}
]
```

如果第一次使用Guava类库或者对其不熟悉，也许某一天你就会踩上这个坑，其实也不算是坑，因为它的文档已经说明了：

>Returns a list that applies function to each element of fromList. The returned list is a transformed view of fromList; changes to fromList will be reflected in the returned list and vice versa.
Since functions are not reversible, the transform is one-way and new items cannot be stored in the returned list. The add, addAll and set methods are unsupported in the returned list.
The function is applied lazily, invoked when needed. This is necessary for the returned list to be a view, but it means that the function will be applied many times for bulk operations like List.contains and List.hashCode. For this to perform well, function should be fast. To avoid lazy evaluation when the returned list doesn't need to be a view, copy the returned list into a new list of your choosing.
If fromList implements RandomAccess, so will the returned list. The returned list is threadsafe if the supplied list and function are.
If only a Collection or Iterable input is available, use Collections2.transform or Iterables.transform.
Note: serializing the returned list is implemented by serializing fromList, its contents, and function -- not by serializing the transformed values. This can lead to surprising behavior, so serializing the returned list is not recommended. Instead, copy the list using ImmutableList.copyOf(Collection) (for example), then serialize the copy. Other methods similar to this do not implement serialization at all for this reason.

大体翻译意思是在说：

>该方法返回一个列表，这个列表中元素是运用方法中传入的功能函数(Function)对原列表中的元素进行处理后的结果，它是原列表的一个功能视图，任何对原列表的改变将会体现到视图列表中。因为Function函数是不可逆的，所以这样的转换是单向的，并且转换的结果不能存储在返回的列表中。所有对视图列表的添加（add/addAll）、设置（set）等都是不被支持的。
Function函数的调用是延迟的（只有在需要的时候才进行调用），这对于视图列表来说是非常有必要的，但是这也同时意味着Function函数会因为各种情况而重复调用多次，比如List.contains/List.hashCode等，也正因为这个原因，所以建议Function应该是尽量轻量级而快速。
如果你不是将返回的视图作为视图使用，那么你需要将该此视图列表拷贝到新的你需要的各种列表中。
如果原列表实现了RandomAccess接口，那么返回的视图列表也会实现该接口。如果原列表是线程安全的，同时Function函数是线程安全的，那么返回的视图列表也是线程安全的。
如果原集合是一个更高的抽象类，如：Collection、Iterable，那么使用Collection2.transform或者Iterables.transform也能满足需求。
注意：序列化视图列表实际上是对原列表的内容的序列化，并不是序列化转换过后的视图内容。这会产生一些比较奇怪的行为，所以序列化返回的视图对象不建议的。如果确实需要序列化返回视图，请使用ImmutableList.copyOf(Collection),然后序列化这个拷贝，而介于这个原因，其它与该方法相似的方法根本就没有实现序列化。

所以说，返回的对象列表是一个视图，其中对它的任何更改都是无效的，并且也不建议对视图对象产生更改，如果需要更改返回的列表，那么需要自己进一步包装，如Lists.newArrayList(retList);

如果需要对返回的结果视图进行处理：
```java
List<TestDTO> retList = query();
List<TestDTO> copyList = Lists.newArrayList(retList);
for(TestDTO dto : copyList){
	dto.setAge(dto.getAge()+1);
}
System.out.println(JSON.toJSONString(copyList,true));
```

这时输出了正确的结果：
```json
[
	{
		"age":19,
		"description":"test obj1",
		"name":"test1"
	},
	{
		"age":20,
		"description":"test obj2",
		"name":"test2"
	},
	{
		"age":21,
		"description":"test obj3",
		"name":"test3"
	}
]
```
