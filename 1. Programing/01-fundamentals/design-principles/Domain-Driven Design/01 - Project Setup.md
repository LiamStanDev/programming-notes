### 介紹
我們將採取 Clean Architecture 架構來建立項目，Clean Architecture 分為四個部分，Presentation、Application、Domain 與 Infrastructure。如下圖
這個架構有以下幾個重點:
![[Pasted image 20240510134849.png]]
* 依賴只能從外流向內部。
* Domain Layer 不會依賴於任何層，為核心業務實體，
* Application Layer，依賴於 Domain。
* Presentation Layer 依賴於 Application。
* Infrastructure
	* 這邊有不同的做法，Infrastructure 可能依賴於 Application 或者是 Domain，主要是看對於 Infrastructure 的抽象接口定義在哪
		* 定義在 Application: 由具體業務來指導，因為 Infrasture 可能有很多內容 e.g. Database, Message Queue, Email 等等。
		* 定義在 Domain: 由實體直接指導資料庫該如何實作。

> 注意: 依賴表示有直接使用實體，而不依賴表示使用其抽象或者接口。

### 項目建立
```bash
# 建立 project
dotnet new sln
dotnet new webapi -o src/Test.Api
dotnet new classlib -o src/Test.Application
dotnet new classlib -o src/Test.Domain
dotnet new classlib -o src/Test.Infrastructure

# 添加到 solution
dotnet sln add $(ls **/*.csproj)

# 構建 dependencies
# 依賴於 infrastructure 是因為要依賴注入，我們將 api 放在 Presentation 中，也可以拆開 api 與 Presentation
dotnet add src/Test.Api reference src/Test.Application src/Test.Infrastructure
dotnet add src/Test.Infrastructure reference src/Test.Application
dotnet add src/Test.Application reference src/Test.Domain
```


#### 目錄結構
``` shell
.
├── Demo01.sln
├── src  # 源代碼
│   ├── Api # Presentation Layer
│   │   ├── Api.csproj
│   │   ├── Controllers
│   │   │   ├── BaseApiController.cs
│   │   │   └── WebinarController.cs
│   │   ├── Program.cs
│   │   ├── Properties
│   │   │   └── launchSettings.json
│   │   ├── appsettings.Development.json
│   │   ├── appsettings.json
│   │
│   │
│   ├── Application
│   │   ├── Abstractions # CQRS 抽象介面
│   │   │   └── Messaging
│   │   │       ├── ICommand.cs
│   │   │       ├── ICommandHandler.cs
│   │   │       ├── IQuery.cs
│   │   │       └── IQueryHandler.cs
│   │   ├── Application.csproj
│   │   ├── Webinars # Features1 其中包含 CQRS 實現，可以添加多個 feature
│   │       ├── Commands
│   │       │   └── CreateWebinar
│   │       │       ├── CreateWebinarCommand.cs
│   │       │       ├── CreateWebinarHandler.cs
│   │       │       └── CreateWebinarRequest.cs
│   │       └── Queries
│   │							└── CreateWebinar
│   │                ├── GetWebinarByIdQuery.cs
│   │                ├── GetWebinaryByIdHandler.cs
│   │                └── WebinarResponse.cs
│   │
│   ├── Domain
│   │   ├── Abstractions # 指導 Infrasture 的實現
│   │   │   ├── IUnitOfWork.cs
│   │   │   └── IWebinarRepository.cs
│   │   ├── Domain.csproj
│   │   ├── Entities
│   │   │   └── Webinar.cs
│   │   ├── Exceptions # Domain 下的客製化異常
│   │   │   ├── Base
│   │   │   │   └── NotFoundException.cs
│   │   │   └── WebinarNotFoundException.cs
│   │   ├── Primitivies
│   │       └── Entity.cs
│   │
│   │
│   └── Infrastructure
│       ├── ApplicationDbContext.cs
│       ├── Configurations
│       │   └── WebinarConfiguration.cs # Feature Repository
│       ├── Infrastructure.csproj
│       ├── Repositories
│           └── WebinarRepository.cs
│
│
└── test # 單元測試與集成測試
```