---
title: "dotnetでWebAPIを作成する方法"
emoji: "😸"
type: "tech"
topics: ["dotnet", "API", "csharp"]
published: true
date: "2024-07-29"
description: "dotnetでWebAPIを作成する方法"
---

## はじめに

C#や dotnet の環境をまったく触らなくなったので忘れないように API を作ってみることにしました。
せっかくなので備忘として記事に残しておきます。

## 完成のイメージ

![](/images/dotnet_todo_webapi/image1.png)

## 利用技術

- dotnet8
- C#
- Dapper
- Npgsql
- Swagger

## API 仕様

1. Get /Todo 指定した Id で１件取得する
2. POST /Todo Todo を追加する
3. PUT /Todo Todo を更新する
4. PATCH Todo Todo を更新する（ステータスを更新する）
5. Get /Todo/{id} Todo を一覧で取得する

## プロジェクトの作成

dotnet8 がインストールされていない場合は[ここから](https://dotnet.microsoft.com/ja-jp/download/dotnet/8.0)インストールする。

```
# プロジェクトの作成
$ dotnet new web -o TodoApi
```

## アプリの実行

HTTPS 開発証明書を信頼する以下のコマンドを実行する

```
$ dotnet dev-certs https --trust
```

開発サーバーを起動する

```
$ dotnet run --launch-profile https
```

以下の URL で swagger が表示されたら OK！
`https://localhost:7243/swagger/index.html`

---

## 全体のフォルダ構成

先に全体のフォルダ構成は以下のようにしました。
・Controllers：コントローラークラスを格納する場所
・Dto；クライアントとのデータのやり取りする際のモデルクラスを格納する場所
・Infrastructure：DB,メール,その他システム連携などの設定ファイルを格納する場所
・Libraries：プロジェクト内の共通処理を格納する場所
・Repositories：DB アクセス処理を格納する場所
・Services：ビジネスロジックを格納する場所

```
├── Api
│   ├── Controllers
│   │   └── TodoController.cs
│   ├── Dto
│   │   ├── CreateRequestDto.cs
│   │   └── CreateResponseDto.cs
│   ├── Infrastructure
│   │   ├── Configure
│   │   │   ├── DatabaseConfigure.cs
│   │   │   └── ServiceConfigure.cs
│   │   └── Database
│   │       └── TodoEntity.cs
│   ├── Libraries
│   │   ├── Exception
│   │   │   └── ValidationException.cs
│   │   └── Middleware
│   │       └── CustomExceptionMiddleware.cs
│   ├── Program.cs
│   ├── Properties
│   │   └── launchSettings.json
│   ├── Repositories
│   │   └── Todo
│   │       ├── ITodoRepository.cs
│   │       └── TodoRepository.cs
│   ├── Services
│   │   └── Todo
│   │       ├── ITodoService.cs
│   │       └── TodoService.cs
│   ├── TodoApi.csproj
│   ├── TodoApi.http
│   ├── appsettings.Development.json
│   └── appsettings.json
├── Api.Tests
│   ├── Api.Tests.csproj
│   ├── GlobalUsings.cs
│   └── UnitTests
│       └── Services
│           └── TodoServiceTests.cs
└── dotnet-todo-web-api.sln
```

## Entity を作成する

DB のテーブル定義に従って Entity クラスを作成する。

```csharp:TodoEntity.cs
namespace TodoApi.Infrastructure.Database;

public class TodoEntity
{
    public int Id { get; set; }
    public string? Title { get; set; }
    public bool Completed { get; set; }
}
```

## DB アクセス処理を作成する

まず、DBConnection を DI する処理を記述する。

```csharp:DatabaseConfigure.cs
using System.Data;

using Npgsql;

using TodoApi.Repositories.Todo;

namespace TodoApi.Infrastructure.Configure;

public static class DatabaseConfigure
{
    public static void Init(IServiceCollection services)
    {
        // 環境変数からDefaultConnectionを取得しコネクションの初期化する。
        var connectionString = Environment.GetEnvironmentVariable("DefaultConnection");
        services.AddScoped<IDbConnection>(sp => new NpgsqlConnection(connectionString));

        // Todoテーブルに対するDBアクセス処理を記述するクラス
        services.AddScoped<ITodoRepository, TodoRepository>();
    }
}

```

Repositories フォルダを作成して DB からデータを取得する処理を記述する。

```csharp:ITodoRepository.cs
using TodoApi.Infrastructure.Database;
using TodoApi.Models;

namespace TodoApi.Repositories.Todo;

public interface ITodoRepository
{
    Task<IEnumerable<TodoEntity>> GetTodoListAsync();
}
```

```csharp:TodoRepository.cs

using System.Data;

using Dapper;

using TodoApi.Infrastructure.Database;
using TodoApi.Models;

namespace TodoApi.Repositories.Todo;

public class TodoRepository : ITodoRepository
{
    private readonly IDbConnection _dbConnection;

    public TodoRepository(IDbConnection dbConnection)
    {
        _dbConnection = dbConnection;
    }

    public async Task<IEnumerable<TodoEntity>> GetTodoListAsync()
    {
        var sql = "select * from \"Todo\"";
        var todos = await _dbConnection.QueryAsync<TodoEntity>(sql);
        return todos;
    }
}
```

## サービスクラスを作成する

次にビジネスロジックを記述するクラスを作成する。

```csharp:ITodoService.cs
using TodoApi.Models;

namespace TodoApi.Services.Todo;

public interface ITodoService
{
    Task<IEnumerable<GetListResponseDto>> GetTodoListAsync();
}
```

```csharp:TodoService.cs
using TodoApi.Models;
using TodoApi.Repositories.Todo;

namespace TodoApi.Services.Todo;

public class TodoService : ITodoService
{
    private readonly ITodoRepository _todoRepository;

    public TodoService(ITodoRepository todoRepository)
    {
        _todoRepository = todoRepository;
    }

    public async Task<IEnumerable<GetListResponseDto>> GetTodoListAsync()
    {
        var todoList = await _todoRepository.GetTodoListAsync();
        return todoList.Select(todo => new GetListResponseDto
        {
            Id = todo.Id,
            Title = todo.Title,
            Completed = todo.Completed
        });
    }
}
```

最後に DI する。

```csharp:ServiceConfigure.cs
using TodoApi.Services.Todo;

namespace TodoApi.Infrastructure.Configure;

public static class ServiceConfigure
{
    public static void Init(IServiceCollection services)
    {
        services.AddScoped<ITodoService, TodoService>();
    }
}
```

## クライアントへ返却するデータを格納する DTO を作成する

今回は仕様がシンプル過ぎるので Entity クラスと同じになっているが、
画面にこの項目以外に表示するものがあったりする場合はここで定義する。
その値を処理するのはビジネスロジックになります。

```csharp:GetListResponseDto.cs
namespace TodoApi.Models;

public class GetListResponseDto
{
    public int Id { get; set; }
    public string? Title { get; set; }
    public bool Completed { get; set; }
}
```

## Controller からサービスクラスを呼び出す

```csharp:TodoController.cs
using Microsoft.AspNetCore.Mvc;

using TodoApi.Models;
using TodoApi.Services.Todo;

namespace TodoApi.Controllers;

[ApiController]
[Route("[controller]")]
public class TodoController : ControllerBase
{
    private readonly ITodoService _todoService;

    public TodoController(ITodoService todoService)
    {
        _todoService = todoService;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<GetListResponseDto>>> GetList()
    {
        var todoList = await _todoService.GetTodoListAsync();
        return Ok(todoList);
    }
}

```

## Program.cs への設定追加

```csharp:Program.cs
using dotenv.net;

using TodoApi.Infrastructure.Configure;

var builder = WebApplication.CreateBuilder(args);
DotEnv.Load();
builder.Configuration.AddEnvironmentVariables();

// DBとビジネスロジックのDIの設定を呼びだす
DatabaseConfigure.Init(builder.Services);
ServiceConfigure.Init(builder.Services);

builder.Services.AddControllers();
...

```

## おわりに

やっぱりサーバーサイド書くのも楽しい
テストも書いてみたい

## ソースコード

https://github.com/yuunull/dotnet-todo-web-api
