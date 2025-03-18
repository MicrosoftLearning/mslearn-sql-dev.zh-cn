---
lab:
  title: 使用 Azure SQL 数据库的自动故障转移组启用应用程序复原能力
  module: Get started with Azure SQL Database for cloud-native application development
---

# 使用 Azure SQL 数据库的自动故障转移组启用应用程序复原能力

在本练习中，你将创建两个充当主要数据库和辅助数据库的 Azure SQL 数据库。 你将配置自动故障转移组，以确保应用程序数据库的高可用性和灾难恢复，并验证应用程序的复制状态。

完成此练习大约需要 30 分钟。

## 准备工作

在开始本练习之前，需要：

- 包含创建和管理资源适当权限的 Azure 订阅。
- [**Visual Studio Code**](https://code.visualstudio.com/download?azure-portal=true) 已安装在你的计算机上，并安装了以下扩展：
    - [C# 开发工具包](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit?azure-portal=true)

## 创建主要和辅助 Azure SQL 服务器

首先，我们将设置主服务器和辅助服务器，并使用 **AdventureWorksLT** 示例数据库。

1. 登录到 [Azure 门户](https://portal.azure.com?azure-portal=true)。

1. 选择 Azure 门户右上角的 Cloud Shell 图标。 它看起来像一个 `>_` 符号。 如果出现提示，请选择 **Bash** 作为 shell 类型。

1. 在 Cloud Shell 终端中运行以下命令。 将值 `<your_resource_group>`、`<your_primary_server>`、`<your_location>`、`<your_secondary_server>` 和 `<your_admin_password>` 替换为实际值：

    * 创建资源组
    ```azurecli
    az group create --name <your_resource_group> --location <your_location>
    ```

    * 创建主 SQL 服务器
    ```azurecli        
    az sql server create --name <your_primary_server> --resource-group <your_resource_group> --location <your_location> --admin-user "sqladmin" --admin-password <your_admin_password>
    ```

    * 创建辅助 SQL 服务器。 同一脚本，仅更改服务器名称和位置
    ```azurecli
    az sql server create --name <your_secondary_server> --resource-group <your_resource_group> --location <your_location> --admin-user "sqladmin" --admin-password <your_admin_password>    
    ```

    * 在主服务器上创建具有指定定价层的示例数据库
    ```azurecli
    az sql db create --resource-group <your_resource_group> --server <your_primary_server> --name AdventureWorksLT --sample-name AdventureWorksLT --service-objective "S0"    
    ```
    
1. 部署完成后，导航到创建的主 Azure SQL 服务器。
1. 在左侧窗格中，选择“**安全性**”下的“**网络**”。 将你的 IP 地址添加到防火墙规则中。
1. 选择“**允许 Azure 服务和资源访问此服务器**”选项。
1. 选择“保存”。
1. 对辅助服务器重复上述步骤。

    这些步骤可确保有一个可随时使用的结构化且冗余的 Azure SQL 数据库环境。

## 配置自动故障转移组

接下来，将为之前设置的 Azure SQL 数据库创建自动故障转移组。 这涉及到在两个服务器之间建立故障转移组，并验证设置以确保其正常工作。

1. 在 Cloud Shell 终端中运行以下命令。 将值 `<your_failover_group>`、`<your_resource_group>`、`<your_primary_server>` 和 `<your_secondary_server>` 替换为实际值：

    * 创建故障转移组
    ```azurecli
    az sql failover-group create -n <your_failover_group> -g <your_resource_group> -s <your_primary_server> --partner-server <your_secondary_server> --failover-policy Automatic --grace-period 1 --add-db AdventureWorksLT
    ```

    * 验证故障转移组
    ```azurecli    
    az sql failover-group show -n <your_failover_group> -g <your_resource_group> -s <your_primary_server>
    ```

    > 花点时间查看结果和 `partnerServers` 值。 为什么这很重要？

    > 通过检查每个伙伴服务器中的 `role` 属性，可以确定服务器当前是充当主服务器还是辅助服务器。 此信息对于了解故障转移组的当前配置和就绪情况至关重要。 它有助于在故障转移场景中评估对应用程序的潜在影响，并确保已正确配置设置，以实现高可用性和灾难恢复。
    
## 与应用程序代码集成

若要将 .NET 应用程序连接到 Azure SQL 数据库终结点，需要执行以下步骤。

1. 在 Visual Studio Code 中，打开终端并运行以下命令以安装 `Microsoft.Data.SqlClient` 包并创建新的 .NET 控制台应用程序。

    ```bash
    dotnet new console -n AdventureWorksLTApp
    cd AdventureWorksLTApp 
    dotnet add package Microsoft.Data.SqlClient --version 5.2.1
    ```

1. 在 **Visual Studio Code** 中打开上一步创建的 `AdventureWorksLTApp` 文件夹。

1. 在项目目录的根目录中创建一个 `appsettings.json` 文件。 此配置文件将存储数据库连接字符串。 请务必将连接字符串中的 `<your_failover_group>` 和 `<your_password>` 值替换为实际详细信息。

    ```json
    {
      "ConnectionStrings": {
        "FailoverGroupConnection": "Server=tcp:<your_failover_group>.database.windows.net,1433;Initial Catalog=AdventureWorksLT;Persist Security Info=False;User ID=sqladmin;Password=<your_password>;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
      }
    }
    ```

1. 在 **Visual Studio Code** 中打开 `.csproj` 文件，并在 `</PropertyGroup>` 标记下方添加以下内容。

    ```xml
    <ItemGroup>
        <PackageReference Include="Microsoft.Data.SqlClient" Version="5.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="6.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="6.0.0" />
    </ItemGroup>
    ```

    完成的 `.csproj` 文件应如下所示。

    ```xml
    <Project Sdk="Microsoft.NET.Sdk">

      <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>net8.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
      </PropertyGroup>
    
      <ItemGroup>
        <PackageReference Include="Microsoft.Data.SqlClient" Version="5.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="6.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="6.0.0" />
      </ItemGroup>
    
    </Project>
    ```

1. 在 **Visual Studio Code** 中打开 `Program.cs` 文件。 在编辑器中，将所有现有代码替换为以下提供的代码。

    > **备注：** 请花点时间查看代码并观察它如何打印有关自动故障转移组中的主服务器和辅助服务器的信息。

    ```csharp
    using System;
    using Microsoft.Data.SqlClient;
    using Microsoft.Extensions.Configuration;
    using System.IO;
    
    namespace AdventureWorksLTApp
    {
        class Program
        {
            static void Main(string[] args)
            {
                var configuration = new ConfigurationBuilder()
                    .SetBasePath(Directory.GetCurrentDirectory())
                    .AddJsonFile("appsettings.json")
                    .Build();
    
                string connectionString = configuration.GetConnectionString("FailoverGroupConnection");
    
                ExecuteQuery(connectionString);
            }
    
            static void ExecuteQuery(string connectionString)
            {
                using (SqlConnection connection = new SqlConnection(connectionString))
                {
                    try
                    {
                        connection.Open();
                        string query = @"
                            SELECT 
                                @@SERVERNAME AS [Primary_Server],
                                partner_server AS [Secondary_Server],
                                partner_database AS [Database],
                                replication_state_desc
                            FROM 
                                sys.dm_geo_replication_link_status";
    
                        using (SqlCommand command = new SqlCommand(query, connection))
                        {
                            using (SqlDataReader reader = command.ExecuteReader())
                            {
                                while (reader.Read())
                                {
                                    Console.WriteLine($"Primary Server: {reader["Primary_Server"]}");
                                    Console.WriteLine($"Secondary Server: {reader["Secondary_Server"]}");
                                    Console.WriteLine($"Database: {reader["Database"]}");
                                    Console.WriteLine($"Replication State: {reader["replication_state_desc"]}");
                                }
                            }
                        }
                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine($"Error executing query: {ex.Message}");
                    }
                    finally
                    {
                        connection.Close();
                    }
                }
            }
        }
    }
    ```

1. 通过从菜单中选择“**运行**” > “**开始调试**”来运行代码，或者直接按 **F5**。 也可以选择顶部工具栏中的播放按钮来启动应用程序。

    > **重要提示：** 如果收到消息 *“你没有用于调试 C# 的扩展。是否需要我们在市场中查找 C# 扩展？“*，请确保已安装 **C# 开发工具包**扩展。

1. 运行代码后，应该会在 Visual Studio Code 的“**调试控制台**”选项卡中看到输出。

    ```
    Primary Server: <your_server_name>
    Secondary Server: <your_server_name>
    Database: AdventureWorksLT
    Replication State: CATCH_UP
    ```
    
    复制状态 `CATCH_UP` 表示数据库已与其伙伴完全同步，并且已准备好进行故障转移。 监视复制状态有助于识别性能瓶颈并确保数据复制高效进行。

## 故障转移到次要区域

想象一下主 Azure SQL 数据库由于区域中断而出现问题的场景。 若要保持服务连续性并最大程度地减少故障时间，需要通过执行强制故障转移将应用程序切换到次要副本。

在强制故障转移期间，所有新的 TDS 会话会自动重新路由到辅助服务器，然后成为主服务器。 最棒的是，无需更改应用程序连接字符串，因为终结点保持不变。

让我们启动故障转移并运行应用程序来检查主服务器和辅助服务器的状态。

1. 返回到 Azure 门户，打开 Cloud Shell 终端的新实例。 运行以下代码。 将值 `<your_failover_group>`、`<your_resource_group>` 和 `<your_primary_server>` 替换为实际值。 `--server` 参数值应为当前的辅助服务器。

    ```azurecli
    az sql failover-group set-primary --name <your_failover_group> --resource-group <your_resource_group> --server <your_server_name>
    ```

    > **注意**：此操作可能需要几分钟时间。

1. 故障转移完成后，再次运行应用程序以验证复制状态。 应会看到辅助服务器现在已接管成为主服务器，而原来的主服务器已成为辅助服务器。

请考虑为什么你可能希望将主应用程序数据库和辅助应用程序数据库放置在同一区域，以及何时选择不同的区域会更有利。

## 清理

在自己的订阅中操作时，最好在项目结束时确定是否仍需要已创建的资源。 

让资源不必要地运行可能会导致额外费用。 可以在 [Azure 门户](https://portal.azure.com?azure-portal=true)中单独删除资源或删除整套资源。

## 详细信息

有关 Azure SQL 数据库的自动故障转移组的详细信息，请参阅[自动故障转移组概述和最佳做法（Azure SQL 数据库）](https://learn.microsoft.com/azure/azure-sql/database/failover-group-sql-db?azure-portal=true)。
