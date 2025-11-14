### 打包
在要打包 package 的 .csproj 中添加
```xml
<PropertyGroup>
  <TargetFramework>net6.0</TargetFramework>
  <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
  <PackageId>MyLibrary</PackageId>
  <Version>1.0.0</Version>
  <Authors>YourName</Authors>
  <Description>My first NuGet package</Description>
  <PackageLicenseExpression>MIT</PackageLicenseExpression>
</PropertyGroup>
```
* GeneratePackageOnBuild: 在編譯時自動生成 .nupkg 文件。
* PackageId: 套件名稱。
* Version: 套件版本號。
* Authors: 作者名稱。
* Description: 套件描述。
* PackageLicenseExpression: 授權類型（例如 MIT 或 Apache-2.0）。

進行編譯
```shell
dotnet pack -c Release
```

### 建立本地 Nuget 倉庫
```shell
# 建立資料夾
mkdir LocalNuGet
# 移入 nuget 包
mv bin/Release/*.nupkg LocalNuGet/
# 添加本地倉庫到 nuget 中
dotnet nuget add source ./LocalNuGet --name LocalNuGet
# 查看倉庫
dotnet nuget list source
# 添加到項目
dotnet add package MyLibrary --version 1.0.0 --source LocalNuGet
# 刪除本地倉庫
dotnet nuget remove source LocalNuGet
```