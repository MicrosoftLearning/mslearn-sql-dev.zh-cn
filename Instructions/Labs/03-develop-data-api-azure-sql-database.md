---
lab:
  title: 为 Azure SQL 数据库开发数据 API
  module: Develop a Data API for Azure SQL Database
---

# 为 Azure SQL 数据库开发数据 API

在本练习中，你将使用 Azure Static Web Apps 为 Azure SQL 数据库开发和部署数据 API。 这将提供在设置数据 API 生成器配置以及在 Azure Static Web App 环境中部署配置的动手体验。

## 先决条件

在开始此练习之前，请确保满足以下条件：

- 一个有效的 Azure 订阅。
- Azure SQL 数据库、Azure Static Web Apps 和 GitHub 的基础知识。
- 已安装包含必要扩展的 Visual Studio Code。
- 用于管理存储库的 GitHub 帐户。

## 设置环境

为此练习设置环境需要执行几个步骤。

### 安装 Visual Studio Code 扩展

在开始此练习之前，你需要安装 Visual Studio Code 扩展。

1. 打开 Visual Studio Code。
1. 在 Visual Studio Code 中打开终端窗口。
1. 通过运行以下命令来安装 Static Web Apps CLI：

    ```bash
    npm install -g @azure/static-web-apps-cli
    ```

1. 通过运行以下命令来安装数据 API 生成器 CLI：

    ```bash
    dotnet tool install --global Microsoft.DataApiBuilder
    ```

Visual Studio Code 现在已设置了必要的扩展。

### 创建 Azure SQL 数据库

如果尚未设置，则需要创建 Azure SQL 数据库。

1. 登录到 [Azure 门户](https://portal.azure.com?azure-portal=true)。 
1. 导航到 Azure SQL**** 页面，然后选择“+ 创建”****。
1. 选择“SQL 数据库”、“单一数据库”和“创建”按钮**********。
1. 在“**创建 SQL 数据库**”对话框中填写所需的信息并选择“**确定**”（其他所有选项保留默认设置）。

    | 设置 | “值” |
    | --- | --- |
    | 免费无服务器产品/服务 | *应用产品/服务* |
    | 订阅 | 订阅 |
    | 资源组 | *选择或创建新资源组* |
    | 数据库名称 | *MyDB* |
    | 服务器 | *选择“**新建**”链接* |
    | 服务器名称 | *选择一个唯一名称* |
    | 位置 | 选择位置** |
    | 身份验证方法 | 使用 SQL 身份验证** |
    | 服务器管理员登录名 | *sqladmin* |
    | 密码 | *输入密码* |
    | 确认密码 | *确认该密码* |

1. 选择“查看 + 创建”，然后选择“创建”。
1. 部署完成后，导航到创建的 Azure SQL 数据库*服务器*。
1. 在左侧窗格中，选择“**安全性**”下的“**网络**”。 将你的 IP 地址添加到防火墙规则中。
1. 选择“保存”。

### 将示例数据添加到数据库

拥有 Azure SQL 数据库后，需要添加一些示例数据。 这将帮助你在 API 启动并运行后对其进行测试。

1. 导航到新创建的 Azure SQL 数据库。
1. 使用 Azure 门户中的**查询编辑器**运行以下 SQL 脚本：

    ```sql
    CREATE TABLE [dbo].[Employees]
    (
        EmployeeID INT PRIMARY KEY,
        FirstName NVARCHAR(50),
        LastName NVARCHAR(50),
        Department NVARCHAR(50)
    );
    
    INSERT INTO [dbo].[Employees] VALUES (1,'John', 'Smith', 'Accounting');
    INSERT INTO [dbo].[Employees] VALUES (2,'Michelle', 'Sanchez', 'IT');
    
    SELECT * FROM [dbo].[Employees];
    ```

### 在 GitHub 中创建基本 Web 应用

在创建 Azure Static Web 应用之前，需要在 GitHub 中创建基本 Web 应用。

1. 若要在 GitHub 中创建基本 Web 应用，请转到[生成原始网站](https://github.com/staticwebdev/vanilla-basic/generate)。
1. 确保存储库模板设置为 **staticwebdev/vanilla-basic**。
1. 在“***所有者***”下，选择你的帐户。
1. 在“***存储库名称***”下，输入名称 **my-sql-repo**。
1. 将存储库设为“**专用**”。
1. 选择“**创建存储库**”按钮。

## 创建 Azure Static Web 应用

首先创建静态 Web 应用，然后将数据 API 生成器配置添加到其中。

1. 在 Azure 门户上，导航到“**Static Web Apps**”页。
1. 选择“+ 新建”。
1. 在“**创建 Static Web 应用**”对话框中填写以下信息（其他所有选项保留默认值）：

    | 设置 | 值 |
    | --- | --- |
    | 订阅 | 订阅 |
    | 资源组 | *选择或创建新资源组* |
    | 名称 | *唯一的名称* |
    | 托管计划源 | *GitHub* |
    | GitHub 帐户 | *选择你的帐户* |
    | 组织 | *最有可能是你的 GitHub 用户名* |
    | 存储库 | *选择在上一步骤中创建的存储库* |
    | 分支 | 主 |

1. 选择“查看 + 创建”，然后选择“创建” 。
1. 部署后，转到该资源。
1. 选择“**在浏览器中查看应用**”按钮，应该会看到一个包含欢迎消息的简单网页。 可以关闭该选项卡。

## 添加数据 API 生成器配置文件

将数据 API 生成器配置添加到 Azure Static Web 应用。 我们需要在 GitHub 存储库中创建新文件，以添加数据 API 生成器配置。

1. 在 Visual Studio Code 中，克隆之前创建的 GitHub 存储库。
1. 在 Visual Studio Code 中打开终端窗口。
1. 运行以下命令来创建新的数据 API 生成器配置文件：

    ```bash
    swa db init --database-type "mssql"
    ```

    这将创建一个名为 *swa-db-connections* 的新文件夹，以及该文件夹中一个名为 *staticwebapp.database.config.json* 的文件。

1. 运行以下命令，将数据库实体添加到配置文件：

    ```bash
    dab add "Employees" --source "dbo.Employees" --permissions "anonymous:*" --config "swa-db-connections/staticwebapp.database.config.json"
    ```

1. 查看 *staticwebapp.database.config.json* 文件的内容。 
1. 提交这些更改并将其推送到 Git 存储库。

## 配置数据库连接

1. 在 Azure 门户上，导航到创建的 Azure Static Web 应用。
1. 在“**设置**”下，选择“**数据库连接**”。
1. 选择“**链接现有数据**”。
1. 在*“*链接数据库*”对话框中，选择之前创建的 Azure SQL 数据库，并进行以下附加设置。

    | 设置 | “值” |
    | --- | --- |
    | 数据库类型 | *Azure SQL 数据库* |
    | 身份验证类型 | *连接字符串* |
    | 用户名 | *管理员用户名* |
    | 密码 | *为管理员用户提供的密码* |
    | 确认复选框 | *已选中* |

   > **备注：** 在生产环境中，请将访问权限限制为仅允许必要的 IP 地址。 此外，请考虑使用 Static Web 应用的托管标识来访问数据库，而不是使用 SQL 身份验证。 有关详细信息，请参阅 [Microsoft Entra 中用于 Azure SQL 的托管标识](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true)。
1. 选择**链接**。

## 测试数据 API 终结点

现在，我们只需要测试数据 API 终结点。

1. 在 Azure 门户上，导航到创建的 Azure Static Web 应用。
1. 在“概述”页上，复制 Web 应用的 URL。
1. 打开新的浏览器标签页并粘贴 URL。 仍应会看到包含 **Vanilla JavaScript 应用**消息的简单网页。
1. 将 **/data-api** 添加到 URL 末尾，然后按 **Enter**。 它应显示“**正常**”，以表明数据 API 正在运行。
1. 将 **/data-api/rest/Employees** 添加到 URL 末尾，然后按 **Enter**。 应会看到前面添加到 Azure SQL 数据库的示例数据。

你已使用 Azure Static Web 应用成功开发和部署了用于 Azure SQL 数据库的数据 API。

## 清理

在自己的订阅中操作时，最好在项目结束时确定是否仍需要已创建的资源。 

让资源不必要地运行可能会导致额外费用。 可以在 [Azure 门户](https://portal.azure.com?azure-portal=true)中单独删除资源或删除整套资源。

## 详细信息

有关适用于 Azure SQL 数据库的数据 API 生成器的详细信息，请参阅[什么是适用于 Azure 数据库的数据 API 生成器？](https://learn.microsoft.com/azure/data-api-builder/overview?azure-portal=true)。
