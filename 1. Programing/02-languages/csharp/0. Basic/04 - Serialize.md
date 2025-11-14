## 簡介
---
本章節的序列化是將物件轉為 XML 或者 JSON，
序列化 Attribute:
1. `[Serializable]`
2. `[NonSerialized]`

## JSON 
---
### JSON 序列化
分為 .Net 提供的與第三方庫 Newtonsoft.Json
#### .Net 原生
```c#
// 序列化
// string json = JsonSerializer.Serialize<People>(people);
string json = await JsonSerializerAsync<People>(people, new JsonSerializerOptions
{
	PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
	WriteIndented = true
});

// 反序列化
// People people = JsonSerializer.Deserialize<People>(json);
People people = await JsonSerializer.DeserializeAsync<People>(json);
```
> 其實 Serialize 方法可以不用給泛型類型，因為只是轉成 string
> 但是 Deserialize 方法則需要知道反序列的類型。
#### Newtonsoft.Json
* 效率較高
```c#
string json = JsonConvert.SerializeObject(person);
People people = JsonConvert.DeserialzeObject<People>(json);
```

### Json 操作
使用 JsonDocument 高效率的操作 Json 物件，不用將其一次性讀入 mem 中。
```csharp
string jsonString = "{\"name\": \"John\", \"age\": 30}";
JsonDocument jsonDocument = JsonDocument.Parse(jsonString);
// 1. 遍歷元素
JsonElement root = jsonDocument.RootElement;
foreach (var property in root.EnumerateObject())
{
    Console.WriteLine($"{property.Name}: {property.Value}");
}

// 2. 取得元素
// 使用 JsonDocument 提供的方法進行查詢，比如 TryGetProperty、GetProperty、TryGetArray 等
if (root.TryGetProperty("name", out var nameProperty))
{
    string name = nameProperty.GetString();
    Console.WriteLine($"Name: {name}");
}

// 3. 返回字符串
string json = JsonSerializer.Serialize(root);
```

## XML 序列化
---
```c#
 // 序列化
    string fileName = Path.Combine(Directory.GetCurrentDirectory(), "XmlSerialize.txt");
    using Stream fileStream = File.Create(fileName);
    People people = new People() { Id = new Guid(), Name = "Liam", Age = 25, Description = "Just a normal people." };
    XmlSerializer xmlSerializer = new XmlSerializer(typeof(People));
    xmlSerializer.Serialize(fileStream, people);
    // 反序列化
    fileStream.Position = 0; // 使文件流指針只回文件頭，其實可以關閉流之後在重新開一個新的就好
    People peopleDeserial = xmlSerializer.Deserialize(fileStream) as People;
    Console.WriteLine(peopleDeserial.Age);
    Console.WriteLine(peopleDeserial.Description);
```