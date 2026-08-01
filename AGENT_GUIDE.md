# Windows Server Tester —— Agent 设计规范

> 本文档是写给 AI Agent（以及人类协作者）的**设计与编码规范**。
> 目的：当用户提出“我要测试 XXX”时，Agent 必须按本文档约定的结构、命名、契约来落地代码，保证项目长期不混乱、可扩展、可维护。
>
> **规则优先级**：本文档 > Agent 自由发挥。若本文档未覆盖的情形，应在动手前与用户确认，而不是自行创造新约定。

---

## 1. 项目定位

- **类型**：.NET 10 控制台程序（`Exe`，`net10.0`）。
- **目标**：一个交互式菜单驱动的工具，用于**逐项验证 Windows Server 的各种功能**（与其它项目无关联，纯本机/单机场景）。
- **典型用例**：验证某组“账号 + 密码”能否成功登录到 Windows 系统；验证某服务是否在运行；验证某端口是否开放等。
- **不做的事**（当前阶段）：
  - 不做登录系统、不做多用户权限；
  - 不做 Server 侧 / 远程服务端；
  - 不做 GUI（仅控制台交互）。

## 2. 核心设计原则

1. **测试项互相独立**：任意两个测试项之间不共享状态、不依赖执行顺序。
2. **框架与测试分离**：`Core/` 是框架代码（菜单、注册、调度、结果模型），`Categories/` 下是具体测试项。**测试项永远不要反向依赖某个具体测试项**。
3. **零手动注册**：测试项通过反射自动发现注册，新增测试项**不需要修改菜单代码**。
4. **输入统一走 `TestContext`**：测试项不直接调用 `Console.ReadLine/Write`，统一通过 `TestContext.Ui`，便于将来替换 UI。
5. **结果用枚举，失败用返回值而非异常**：业务上“没通过”是预期内的结果，返回 `Failed`；只有框架级/意外错误才抛异常（由框架兜底转成 `Error`）。
6. **MVP 优先**：先跑通“框架 + 1 个示例测试项”，再扩展类别。能用简单代码解决的，不引入额外依赖。

---

## 3. 目录结构规范

```
Windows Server Tester/                       # 仓库根
├── AGENT_GUIDE.md                           # 本文档
├── Windows Server Tester.sln
└── Windows Server Tester/                   # 主项目
    ├── Program.cs                           # 入口：构建 → 启动主菜单
    ├── Windows Server Tester.csproj
    │
    ├── Core/                                # 框架核心（与具体测试无关）
    │   ├── ITestItem.cs                     # 测试项契约接口
    │   ├── TestItemBase.cs                  # 抽象基类，提供默认实现
    │   ├── TestStatus.cs                    # 结果状态枚举
    │   ├── TestResult.cs                    # 结果模型（record）
    │   ├── TestContext.cs                   # 运行上下文（提供 Ui 等）
    │   ├── TestRegistry.cs                  # 反射发现 + 分组注册
    │   └── MenuRouter.cs                    # 菜单展示 + 调度执行
    │
    ├── Ui/                                  # 控制台交互辅助
    │   ├── IConsoleUi.cs                    # 输入/输出抽象
    │   ├── ConsoleUi.cs                     # 实现：读输入、写输出、读密码
    │   └── ResultPrinter.cs                 # 格式化打印测试结果
    │
    └── Categories/                          # 测试项：按类别分文件夹
        ├── Authentication/                  # 类别 = 文件夹名
        │   └── CredentialLoginTest.cs       # 一个测试项 = 一个文件
        ├── Network/
        │   └── PortOpenTest.cs
        └── Services/
            └── ServiceRunningTest.cs
```

**规则**：
- 每个类别 = `Categories/` 下的一个文件夹。
- 每个测试项 = 类别文件夹下的一个 `.cs` 文件，文件内只放一个测试项类。
- `Core/` 和 `Ui/` 的文件**不要**放进 `Categories/`；测试项**不要**放进 `Core/`。

---

## 4. 命名规范

| 对象 | 规则 | 示例 |
|------|------|------|
| 命名空间 | `Windows_Server_Tester.<文件夹路径>` | `Windows_Server_Tester.Categories.Authentication` |
| 测试项类名 | `<功能名>Test`，PascalCase | `CredentialLoginTest` |
| 测试项文件名 | 与类名相同 | `CredentialLoginTest.cs` |
| 类别文件夹名 | PascalCase 复数或领域名 | `Authentication`、`Network`、`Services` |
| 测试项 `Id` | `<类别小写>.<短名小写-连字符>` | `authentication.credential-login` |
| `DisplayName` | 中文，简洁动词短语 | “验证账号密码可登录系统” |

---

## 5. TestItem 契约

### 5.1 接口与基类

```csharp
namespace Windows_Server_Tester.Core;

public interface ITestItem
{
    string Id { get; }            // 全局唯一，格式见命名规范
    string Category { get; }      // 类别名，与所在文件夹对应
    string DisplayName { get; }   // 菜单显示文本
    string Description { get; }   // 一句话说明本测试做什么
    TestResult Run(TestContext context);
}
```

```csharp
namespace Windows_Server_Tester.Core;

public abstract class TestItemBase : ITestItem
{
    public abstract string Id { get; }
    public abstract string Category { get; }
    public abstract string DisplayName { get; }
    public abstract string Description { get; }
    public abstract TestResult Run(TestContext context);
}
```

> **约定**：所有具体测试项继承 `TestItemBase`，不直接实现 `ITestItem`，以便未来在基类注入通用能力（日志、计时、取消等）时无需逐项改动。

### 5.2 结果模型

```csharp
namespace Windows_Server_Tester.Core;

public enum TestStatus
{
    Passed,    // 通过
    Failed,    // 未通过（业务预期内的否定结果）
    Warning,   // 通过但有注意事项
    Skipped,   // 主动跳过（如缺少前置条件）
    Error      // 框架/意外错误（通常由异常兜底产生）
}

public sealed record TestResult(
    TestStatus Status,
    string Summary,        // 一句话结论，如 "账号 admin 登录成功"
    string? Detail = null  // 可选的详细信息（多行文本）
);
```

### 5.3 运行上下文

```csharp
namespace Windows_Server_Tester.Core;

public sealed class TestContext
{
    public IConsoleUi Ui { get; }
    public CancellationToken CancellationToken { get; }

    public TestContext(IConsoleUi ui, CancellationToken cancellationToken)
    {
        Ui = ui;
        CancellationToken = cancellationToken;
    }
}
```

> 测试项需要的所有外部输入（账号、密码、主机、端口……）一律通过 `context.Ui` 在 `Run` 内部读取，**不要**在构造时读取、也不要直接用 `Console`。

---

## 6. UI 抽象

```csharp
namespace Windows_Server_Tester.Ui;

public interface IConsoleUi
{
    string ReadLine(string prompt);            // 普通输入
    string ReadPassword(string prompt);        // 密码输入（不回显）
    void Write(string text);
    void WriteLine(string? text = null);
    void WriteLine(string color, string text); // 带颜色输出
}
```

**说明**：
- 密码类输入必须走 `ReadPassword`，控制台不回显。
- 颜色输出统一在 `Ui` 层处理，测试项不直接拼 ANSI 转义码。

---

## 7. 菜单与自动注册机制

### 7.1 注册（`TestRegistry`）

- 程序启动时，`TestRegistry` 通过反射扫描 `Windows_Server_Tester.Categories` 命名空间下所有 `ITestItem` 实现，实例化（要求有无参构造）。
- 按 `Category` 分组，组内按 `DisplayName` 排序。
- 若发现重复 `Id`，启动即报错（fail-fast），避免静默覆盖。

### 7.2 调度（`MenuRouter`）

两层菜单：
1. **类别菜单**：列出所有类别 + “退出”。
2. **测试项菜单**：列出选中类别下的所有测试项 + “返回上级” + “返回主菜单”。

选中测试项后：
1. 实例化 `TestContext`。
2. 调用 `Run`，捕获一切异常 → 转成 `TestResult(Error, "执行出错", ex.Message)`。
3. 调 `ResultPrinter` 打印结果。
4. 回到测试项菜单（不退出程序）。

### 7.3 入口（`Program.cs`）

```csharp
using Windows_Server_Tester.Core;
using Windows_Server_Tester.Ui;

var ui = new ConsoleUi();
var registry = TestRegistry.Discover();
var router = new MenuRouter(ui, registry);
await router.RunAsync(CancellationToken.None);
```

> 入口保持极简，所有逻辑在 `Core` / `Ui` / `Categories` 中。

---

## 8. 添加一个新测试项 —— 标准步骤

当用户说“我要测试 XXX”时，**严格按以下步骤执行**：

1. **归类**：判断它属于哪个类别。已有合适类别就复用；没有就在 `Categories/` 下新建文件夹（PascalCase）。
2. **建文件**：在 `Categories/<类别>/` 下新建 `<功能名>Test.cs`。
3. **写命名空间**：`namespace Windows_Server_Tester.Categories.<类别>;`
4. **继承基类**：`public sealed class <功能名>Test : TestItemBase`
5. **填元数据**：`Id`（`<类别小写>.<短名>`）、`Category`、`DisplayName`（中文）、`Description`。
6. **实现 `Run`**：
   - 通过 `context.Ui` 读取所需输入；
   - 执行验证逻辑；
   - 返回 `TestResult`（业务否定返回 `Failed`，不要抛异常）；
   - 若有敏感信息（密码），**不得**出现在 `Summary` / `Detail` 中。
7. **不要**修改 `Program.cs`、`MenuRouter`、`TestRegistry` —— 框架会自动发现。
8. **编译验证**：`dotnet build` 通过即完成。

### 示例：账号密码登录验证测试项

```csharp
namespace Windows_Server_Tester.Categories.Authentication;

using Windows_Server_Tester.Core;
using Windows_Server_Tester.Ui;

public sealed class CredentialLoginTest : TestItemBase
{
    public override string Id => "authentication.credential-login";
    public override string Category => "Authentication";
    public override string DisplayName => "验证账号密码可登录系统";
    public override string Description => "使用给定账号密码尝试登录本机 Windows 系统，验证凭据是否正确。";

    public override TestResult Run(TestContext context)
    {
        var ui = context.Ui;
        var username = ui.ReadLine("请输入账号: ");
        var password = ui.ReadPassword("请输入密码: ");

        // TODO: 实现真实校验逻辑（如 P/Invoke LogonUser / advapi32）
        bool ok = TryLogon(username, password, out string detail);

        return ok
            ? new TestResult(TestStatus.Passed, $"账号 {username} 登录成功", detail)
            : new TestResult(TestStatus.Failed,  $"账号 {username} 登录失败", detail);
    }

    private static bool TryLogon(string user, string pass, out string detail)
    {
        detail = "（占位实现）";
        return false;
    }
}
```

---

## 9. 约束与反模式（禁止做的事）

| 反模式 | 为什么禁止 | 正确做法 |
|--------|------------|----------|
| 在 `Run` 里直接 `Console.ReadLine()` | 绕过 UI 抽象，无法替换/测试 | 用 `context.Ui.ReadLine(...)` |
| 新增测试项时手动改 `MenuRouter` | 破坏自动发现，易遗漏 | 只新建文件，靠反射注册 |
| 测试项之间通过静态字段传数据 | 破坏独立性、引入隐式顺序依赖 | 测试项自包含，输入从 `Ui` 读 |
| 把业务“不通过”用 `throw` 表达 | 把预期结果当异常，混淆语义 | 返回 `TestResult(Failed, ...)` |
| 把密码写进 `Summary`/`Detail` | 泄露敏感信息 | 摘要里只用账号，不含密码 |
| 把测试项类放进 `Core/` | 框架与业务混淆 | 放 `Categories/<类别>/` |
| 一个文件放多个测试项类 | 难以发现、难命名 | 一项一文件 |
| 引入新 NuGet 依赖不告知 | 隐式扩大依赖面 | 先与用户确认，能不引则不引 |

---

## 10. 敏感信息处理

- 密码、令牌等凭据一律用 `IConsoleUi.ReadPassword` 读取，**不回显**。
- 结果文本（`Summary`、`Detail`）中**不得**包含明文密码；需要指代时用 `***` 或仅显示账号。
- 暂不持久化任何凭据到磁盘；如未来需要写日志，必须先脱敏。

---

## 11. MVP 阶段范围

第一个里程碑只交付：
1. `Core/` 全套框架（接口、基类、结果、上下文、注册、菜单）。
2. `Ui/` 的 `ConsoleUi` + `ResultPrinter`。
3. `Program.cs` 极简入口。
4. **一个**示例测试项：`Authentication/CredentialLoginTest`（验证账号密码可登录系统）。

在此范围内不做：多语言、配置文件、日志文件、远程执行、插件动态加载。

---

## 12. 变更本规范

当出现本文档未约定的新情形时：
1. **不**自行创造约定；
2. 先与用户确认；
3. 确认后把新约定补回本文档对应章节，保持文档为唯一事实来源。
