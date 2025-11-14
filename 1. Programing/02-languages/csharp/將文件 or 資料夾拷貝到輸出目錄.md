需要將在 .csproj 中修改配置，有幾中標籤
1. `<None>`: 對文件或者目錄進行直接拷貝，用於與程序執行較無關的文件 e.g. 說明文檔、測試
2. `<Content>`: 對文件或者目錄進行拷貝，但用於程序執行中會使用的內容 e.g. 配置文件、靜態資源、測試用例等

#### None
* 單個文件
```html
<ItemGroup>
  <None Include="docs\readme.txt">
    <CopyToOutputDirectory>Always</CopyToOutputDirectory>
  </None>
</ItemGroup>
```
* 整個目錄
```html
<ItemGroup>
  <None Include="docs\**\*.txt"> 
    <CopyToOutputDirectory>Always</CopyToOutputDirectory>
  </None>
</ItemGroup>
```
> 選項有 `Always` 與 `PreserveNewest`

#### Content
* 單個文件
```html
<ItemGroup>
  <Content Include="appsettings.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```
* 整個目錄
```html
<ItemGroup>
  <Content Include="inputs\**\*">
    <CopyToOutputDirectory>Always</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```
