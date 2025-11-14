# 簡介
---
比 Automapper 更快且配置更簡單的 Mapper。

# 使用
---
### 基本映射 (擴展方法)
這種映射就是名稱一樣的兩個類型進行映射，不需要其餘配置。
Mapperster 提供過展方法 `Adapt` 可以輕鬆做到
```c#
// 生成
var destObject = sourceObject.Adapt<Destination>();
// 寫入
srouceObject.Adapt(destObject);
```

### Queryable 擴展
```c#
// 從 DbContext 中取得 Sources 這個 Queryable，並對每個元素進行 Projection
var destinations = context.Sources.ProjectToType<Destination>().ToList();

// 原始方式: 手動編寫
var destinations = context.Sources.Select(s => new Destination {
        Id = s.Id,
        Name = s.Name,
        Surname = s.Surname,
        ....
    })
    .ToList();
```


### 配置 Mapper (用於 IMapper)
有些時候不能直接進行映射，這時候就需要手動添加配置。
##### 方式一: 覆蓋配置
若該映射已經存在配置，這種方式會進行覆蓋
```c#
TypeAdapterConfig<TSource, TDestination>
    .NewConfig() // 覆蓋配置
    .Ignore(dest => dest.Age)
    .Map(dest => dest.FullName,
        src => string.Format("{0} {1}", src.FirstName, src.LastName));
```

##### 方式二: 添加配置
若該映射已經存在配置，這種方式會對原有的配置進行添加
```c#
TypeAdapterConfig<TSource, TDestination>
        .ForType() // 添加配置
        .Ignore(dest => dest.Age)
        .Map(dest => dest.FullName,
            src => string.Format("{0} {1}", src.FirstName, src.LastName));
```

### 在 Asp.net 中使用
