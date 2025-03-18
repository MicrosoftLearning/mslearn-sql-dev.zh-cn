---
lab:
  title: 配置和部署用于 Azure SQL 数据库项目的 CI/CD 管道
  module: Develop for an Azure SQL Database
---

# 配置和部署用于 Azure SQL 数据库项目的 CI/CD 管道

在本练习中，你将使用 Visual Studio Code 和 GitHub Actions 为 Azure SQL 数据库项目创建、配置和部署 CI/CD 管道。 这样，你将可以熟悉为 Azure SQL 数据库项目设置 CI/CD 管道的过程。

完成此练习大约需要 30 分钟。

## 准备工作

在开始本练习之前，需要：

- 包含创建和管理资源适当权限的 Azure 订阅。
- 已在你的计算机上安装包含以下扩展的 [Visual Studio Code](https://code.visualstudio.com/download)：
  - [SQL 数据库项目](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql)。
  - [GitHub 拉取请求](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-pull-request-github)。
- GitHub 帐户。
- *GitHub Actions* 管道的基础知识。

## 创建 Azure SQL 数据库

首先，需要创建新的 Azure SQL 数据库。

1. 登录到 [Azure 门户](https://portal.azure.com?azure-portal=true)。 
1. 导航到 Azure SQL**** 页面，然后选择“+ 创建”****。
1. 选择“SQL 数据库”、“单一数据库”和“创建”按钮**********。
1. 在“**创建 SQL 数据库**”对话框中填写所需的信息并选择“**确定**”，其他所有选项保留默认设置。

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
1. 选择“**允许 Azure 服务和资源访问此服务器**”选项。 此选项允许 GitHub Actions 访问数据库。

    > **备注：** 在生产环境中，请将访问权限限制为仅允许必要的 IP 地址。 此外，请考虑使用 GitHub Action 的托管标识来访问数据库，而不是使用 SQL 身份验证。 有关详细信息，请参阅 [Microsoft Entra 中用于 Azure SQL 的托管标识](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true)。

1. 选择“保存”。

## 设置 Github 存储库

接下来，需要设置新的 GitHub 存储库。

1. 打开 [GitHub](https://github.com) 网站。
1. 登录到 GitHub 帐户。
1. 转到帐户下的“**存储库**”，然后选择“**新建**”。
1. 选择你的帐户作为“**所有者**”。 输入名称 **my-sql-db-repo**。
1. 将存储库设置为“**专用**”。
1. 选择“创建存储库”。

### 安装 Visual Studio Code 扩展并克隆存储库

在克隆存储库之前，请确保已安装所需的 **Visual Studio Code** 扩展。 如需指导，请参阅“**开始之前**”部分。

1. 在 Visual Studio Code 中，选择“视图” > “命令面板”。
1. 在命令面板中，键入并选择 `Git: Clone`。
1. 输入在上一步中创建的存储库 URL 并选择“**克隆**”。 URL 应遵循以下格式：*https://github.com/<your_account>/<your_repository>.git*
1. 选择或创建用于存储存储库文件的文件夹。

## 创建和配置 Azure SQL 数据库项目

通过 Visual Studio 中的 Azure SQL 数据库项目，可以开发、生成、测试和发布数据库架构和数据。 在本部分中，你将创建一个项目并将其配置为连接到之前设置的 Azure SQL 数据库。

1. 在 Visual Studio Code 中，选择“视图” > “命令面板”。
1. 在命令面板中，键入 `Database projects: New` 并将其选中。
    > **备注：** 安装 mssql 扩展的 SQL 工具服务可能需要几分钟时间。
1. 选择“Azure SQL 数据库”。
1. 输入名称 **MyDBProj**，然后按 **Enter** 进行确认。
1. 选择克隆的 GitHub 存储库文件夹以保存项目。
1. 对于 **SDK 样式项目**，请选择“**是(推荐)**”。
    > **备注：** 请注意，已创建一个名为 **MyDBProj** 的新项目。

### 在项目中创建新的 SQL 文件

创建 Azure SQL 数据库项目后，我们将向项目中添加新的 SQL 文件以创建新表。

1. 在 Visual Studio Code 上，选择左侧活动栏上的**数据库项目**图标。
1. 右键单击项目名称，然后选择“**添加表**”。
1. 将表命名为 **Employees**，然后按 **Enter**。
1. 用以下代码替换现有脚本。

    ```sql
    CREATE TABLE [dbo].[Employees]
    (
        EmployeeID INT PRIMARY KEY,
        FirstName NVARCHAR(50),
        LastName NVARCHAR(50),
        Department NVARCHAR(50)
    );
    ```

1. 关闭编辑器。 请注意，`Employees.sql` 文件已保存在项目中。

## 将更改提交至存储库

创建 Azure SQL 数据库项目并将表脚本添加到项目中后，我们将更改提交至存储库。

1. 在 Visual Studio Code 上，选择左侧活动栏上的**源代码管理**图标。
1. 输入消息*已创建项目并添加了创建表脚本*。
1. 选择“**提交**”来提交更改。
1. 在省略号下，选择“**推送**”将更改推送到存储库。

## 验证存储库中的更改

推送更改后，我们将在 GitHub 存储库中验证这些更改。

1. 打开 [GitHub](https://github.com) 网站。
1. 导航到 **my-sql-db-repo** 存储库。
1. 在“**<>代码**”选项卡上，打开 **MyDBProj** 文件夹。
1. 检查 **Employees.sql** 文件中的更改是否是最新的。

## 使用 GitHub Actions 设置持续集成 (CI)

通过 GitHub Actions，可以直接在 GitHub 存储库中自动化、自定义和运行软件开发工作流。 在本部分中，你将配置 GitHub Actions 工作流，以便通过在数据库中创建新表来生成和测试 Azure SQL 数据库项目。

### 创建服务主体

1. 选择 Azure 门户右上角的 **Cloud Shell** 图标。 它看起来像一个 `>_` 符号。 如果出现提示，请选择 **Bash** 作为 shell 类型。

1. 在 Cloud Shell 终端中运行以下命令。 将值 `<your_subscription_id>` 和 `<your_resource_group_name>` 替换为实际值。 可以在 Azure 门户上的“**订阅**”和“**资源组**”页上获取这些值。

    ```azurecli
    az ad sp create-for-rbac --name "MyDBProj" --role contributor --scopes /subscriptions/<your_subscription_id>/resourceGroups/<your_resource_group_name>
    ```

    打开文本编辑器并使用上一个命令的输出创建类似于下面的凭据代码片段：
    
    ```
    {
    "clientId": <your_service_principal_appId>,
    "clientSecret": <your_service_principal_password>,
    "tenantId": <your_service_principal_tenant>,
    "subscriptionId": <your_subscription_id>
    }
    ```

1. 使文本编辑器保持打开状态。 我们将在下一部分中提及它。

### 将机密添加到存储库

1. 在 GitHub 存储库中，选择“**设置**”。
1. 选择“**机密和变量**”，然后选择“**操作**”。
1. 在“**机密**”选项卡上，选择“**新存储库机密**”，并提供以下信息。

    | 名称 | 值 |
    | --- | --- |
    | AZURE_CREDENTIALS | 上一部分中复制的服务主体输出。|
    | AZURE_CONN_STRING | 连接字符串。 |
   
    连接字符串应类似于以下内容：

    ```Server=<your_sqldb_server>.database.windows.net;Initial Catalog=MyDB;Persist Security Info=False;User ID=sqladmin;Password=<your_password>;Encrypt=True;Connection Timeout=30;```

### 创建 GitHub Actions 工作流

1. 在 GitHub 存储库中，选择“操作”选项卡。
1. 选择“**自己设置工作流**”链接。
1. 在 **main.yml** 文件上复制以下代码。 该代码包括生成和部署数据库项目的步骤。

    ```yaml
    name: Build and Deploy SQL Database Project
    on:
      push:
        branches:
          - main
    jobs:
      build:
        permissions:
          contents: 'read'
          id-token: 'write'
          
        runs-on: ubuntu-latest  # Can also use windows-latest depending on your environment
        steps:
          - name: Checkout repository
            uses: actions/checkout@v3

        # Install the SQLpackage tool
          - name: sqlpack install
            run: dotnet tool install -g microsoft.sqlpackage
    
          - name: Login to Azure
            uses: azure/login@v1
            with:
              creds: ${{ secrets.AZURE_CREDENTIALS }}
    
          # Build and Deploy SQL Project
          - name: Build and Deploy SQL Project
            uses: azure/sql-action@v2.3
            with:
              connection-string: ${{ secrets.AZURE_CONN_STRING }}
              path: './MyDBProj/MyDBProj.sqlproj'
              action: 'publish'
              build-arguments: '-c Release'
              arguments: '/p:DropObjectsNotInSource=true'  # Optional: Customize as needed
      ```

      YAML 文件中的“**生成和部署 SQL 项目**”步骤使用存储在 `AZURE_CONN_STRING` 机密中的连接字符串连接到 Azure SQL 数据库。 该操作指定 SQL 项目文件的路径，设置要发布的操作以部署项目，并包括要在“发布”模式下编译的生成参数。 此外，它还使用 `/p:DropObjectsNotInSource=true` 参数来确保在部署期间从目标数据库中删除源中不存在的任何对象。

1. 提交更改。

### 测试 GitHub Actions 工作流

1. 在 GitHub 存储库中，选择“操作”选项卡。
1. 选择“**生成和部署 SQL 数据库项目**”工作流。
    > **备注：** 你将看到正在进行中的工作流。 等待检查完成。 如果已完成，请选择最新的运行来查看详细信息。

### 验证 Azure SQL 数据库中的更改

设置 GitHub Actions 工作流来生成和部署 Azure SQL 数据库项目后，就可以验证 Azure SQL 数据库中的更改。

1. 登录到 [Azure 门户](https://portal.azure.com?azure-portal=true)。 
1. 导航到 **MyDB** SQL 数据库。
1. 选择**查询编辑器**。
1. 使用 **sqladmin** 凭据连接到数据库。
1. 在“**表**”部分下，验证是否已创建 **Employees** 表。 如有需要，请刷新。

已成功设置 GitHub Actions 工作流来生成和部署 Azure SQL 数据库项目。

## 清理

在自己的订阅中操作时，最好在项目结束时确定是否仍需要已创建的资源。 

让资源不必要地运行可能会导致额外费用。 可以在 [Azure 门户](https://portal.azure.com?azure-portal=true)中单独删除资源或删除整套资源。

## 详细信息

有关 Azure SQL 数据库的 SQL 数据库项目扩展的详细信息，请参阅 [SQL 数据库项目扩展入门指南](https://learn.microsoft.com/azure-data-studio/extensions/sql-database-project-extension-getting-started?azure-portal=true)。
