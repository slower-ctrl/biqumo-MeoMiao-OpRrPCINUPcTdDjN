
书接上回，我们今天开始实现对象集合与DataTable的相互转换。


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241129211727004-1532421911.png)


# ***01***、接口设计


上文中已经详细讲解了整体设计思路以及大致设计了需要哪些方法。下面我们先针对上文设计思想确定对外提供的接口。具体接口如下：



```
//根据列名数组创建表格
public static DataTable Create(string[] columnNames, string? tableName = null);

//根据列名-类型键值对创建表格
public static DataTable Create(Dictionary<string, Type> columns, string? tableName = null);

//根据类创建表格
//如果设置DescriptionAttribute，则将特性值作为表格的列名称
//否则将属性名作为表格的列名称
public static DataTable Create<T>(string? tableName = null);

//把表格转换为实体对象集合
//如果设置DescriptionAttribute，则将特性值作为表格的列名称
//否则将属性名作为表格的列名称
public static IEnumerable<T> ToModels<T>(DataTable dataTable);

//把实体对象集合转为表格
//如果设置DescriptionAttribute，则将特性值作为表格的列名称
//否则将属性名作为表格的列名称
public static DataTable ToDataTable<T>(IEnumerable models, string? tableName = null);

//把一维数组作为一列转换为表格
public static DataTable ToDataTableWithColumnArray<TColumn>(TColumn[] array, string? tableName = null, string? columnName = null);

//把一维数组作为一行转换为表格
public static DataTable ToDataTableWithRowArray<TRow>(TRow[] array, string? tableName = null);

//行列转置
public static DataTable Transpose(DataTable dataTable, bool isColumnNameAsData = true);

```

# ***02***、根据列名数组创建表格


该方法实现比较简单，我们直接看代码：



```
//根据列名数组创建表格
public static DataTable Create(string[] columnNames, string? tableName = null)
{
    var table = new DataTable(tableName);
    foreach (var columnName in columnNames)
    {
        table.Columns.Add(columnName);
    }
    return table;
}

```

我们进行一个简单的单元测试：



```
[Fact]
public void Create()
{
    //正常创建成功
    var columnNames = new string[] { "A", "B" };
    var table = TableHelper.Create(columnNames);
    Assert.Equal("", table.TableName);
    Assert.Equal(2, table.Columns.Count);
    Assert.Equal(columnNames[0], table.Columns[0].ColumnName);
    Assert.Equal(columnNames[1], table.Columns[1].ColumnName);
    Assert.Equal(typeof(string), table.Columns[0].DataType);
    Assert.Equal(typeof(string), table.Columns[1].DataType);

    //验证表名
    table = TableHelper.Create(columnNames, "test");
    Assert.Equal("test", table.TableName);

    //验证列名不能重复
    columnNames = new string[] { "A", "A" };
    Assert.Throws(() => TableHelper.Create(columnNames));
}

```

# ***03***、根据列名\-类型键值对创建表格


此方法是上一个方法的补充，默认直接根据列名创建表格，则所有列的数据类型都是string类型，而此方法可以指定每列的数据类型，实现也很简单，代码如下：



```
//根据列名-类型键值对创建表格
public static DataTable Create(Dictionary<string, Type> columns, string? tableName = null)
{
    var table = new DataTable(tableName);
    foreach (var column in columns)
    {
        table.Columns.Add(column.Key, column.Value);
    }
    return table;
}

```

# ***04***、根据类创建表格


该方法是将类的属性名作为表格的列名称，属性对应的类型作为表格列的数据类型，把类转为表格。


同时我们需要约束类只能为结构体或类，而不能是枚举、基础类型、以及集合类型、委托、接口等。显然我们很难通过泛型约束达到我们的需求，因此我们首先需要对该方法的泛型进行类型校验。


校验成功后，我们只需要通过反射即可拿到类的所有属性信息，即可创建表格。同时我们约定如果属性设置了DescriptionAttribute特性，则特性值作为列名，如果没有设置特性则取属性名称作为列名。



```
//根据类创建表格
//如果设置DescriptionAttribute，则将特性值作为表格的列名称
//否则将属性名作为表格的列名称
public static DataTable Create<T>(string? tableName = null)
{
    //T必须是结构体或类，并且不能是集合类型
    AssertTypeValid();
    //获取类的所有公共属性
    var properties = typeof(T).GetProperties();
    var columns = new Dictionary<string, Type>();
    foreach (var property in properties)
    {
        //根据属性获取列名
        var columnName = GetColumnName(property);
        //组织列名-类型键值对
        columns.Add(columnName, property.PropertyType);
    }
    return Create(columns, tableName);
}

//断言类型有效性
private static void AssertTypeValid<T>()
{
    var type = typeof(T);
    if (type.IsValueType && !type.IsEnum && !type.IsPrimitive)
    {
        //是值类型，但是不是枚举或基础类型
        return;
    }
    else if (typeof(T).IsClass && !typeof(IEnumerable).IsAssignableFrom(typeof(T)))
    {
        //是类类型，但是不是集合类型，同时也不是委托、接口类型
        return;
    }
    throw new InvalidOperationException("T must be a struct or class and cannot be a collection type.");
}

//根据属性获取列名称
private static string GetColumnName(PropertyInfo property)
{
    //获取描述特性
    var attribute = property.GetCustomAttribute();
    //如果存在描述特性则返回描述，否则返回属性名称
    return attribute?.Description ?? property.Name;
}

```

下面我们针对枚举、字符串，基础类型、集合等情况进行详细的单元测试，代码如下：



```
[Fact]
public void Create_T()
{
    //验证枚举
    var expectedMessage = "T must be a struct or class and cannot be a collection type.";
    var exception1 = Assert.Throws(() => TableHelper.Create());
    Assert.Equal(expectedMessage, exception1.Message);

    //验证基础类型
    var exception2 = Assert.Throws(() => TableHelper.Create<int>());
    Assert.Equal(expectedMessage, exception2.Message);

    //验证字符串类型
    var exception3 = Assert.Throws(() => TableHelper.Create<string>());
    Assert.Equal(expectedMessage, exception3.Message);

    //验证集合
    var exception4 = Assert.Throws(() => TableHelper.Createstring, Type>>());
    Assert.Equal(expectedMessage, exception4.Message);

    //验证正常情况
    var table = TableHelper.Createdouble>>();
    Assert.Equal("", table.TableName);
    Assert.Equal(3, table.Columns.Count);
    Assert.Equal("标识", table.Columns[0].ColumnName);
    Assert.Equal("姓名", table.Columns[1].ColumnName);
    Assert.Equal("Age", table.Columns[2].ColumnName);
    Assert.Equal(typeof(string), table.Columns[0].DataType);
    Assert.Equal(typeof(string), table.Columns[1].DataType);
    Assert.Equal(typeof(double), table.Columns[2].DataType);
}

```

***注***：测试方法代码以及示例源码都已经上传至代码库，有兴趣的可以看看。[https://gitee.com/hugogoos/Ideal](https://github.com)


 本博客参考[豆荚加速器官网](https://baitenghuo.com)。转载请注明出处！
