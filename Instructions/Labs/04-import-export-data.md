---
lab:
  title: 在 Azure SQL 数据库中导入和导出用于开发的数据
  module: Import and export data for development in Azure SQL Database
---

# 在 Azure SQL 数据库中导入和导出用于开发的数据

在本练习中，你将从外部 REST 终结点（使用 Azure Static Web 应用模拟）导入数据，并使用 Azure 函数导出数据。 实验室将提供使用 Azure SQL 数据库进行开发的实践经验，重点是集成 REST API 和 Azure Functions 来处理数据导入/导出操作。

## 先决条件

在开始此实验室之前，请确保已具备以下各项：

- 包含创建和管理资源权限的可用 Azure 订阅。
- Azure SQL 数据库、REST API 和 Azure Functions 的基础知识。
- 已安装包含以下扩展的 Visual Studio Code：
      - Azure Functions 扩展。
- 已安装用于克隆存储库的 Git。
- 用于管理数据库的 SQL Server Management Studio (SSMS) 或 Azure Data Studio。

## 设置环境

让我们首先为此实验室设置必要的资源，包括 Azure SQL 数据库以及导入和导出数据所需的工具。

### 创建 Azure SQL 数据库

此步骤需要在 Azure 中创建一个数据库：

1. 在 Azure 门户中，转到“**SQL 数据库**”。
1. 选择**创建**。
1. 填写必填字段：

    | 设置 | “值” |
    |---|---|
    | 免费无服务器产品/服务 | 应用产品/服务 |
    | 订阅 | 订阅 |
    | 资源组 | 选择或创建新资源组 |
    | 数据库名称 | **MyDB** |
    | 服务器 | 选择或创建新服务器 |
    | 身份验证方法 | SQL 身份验证 |
    | 服务器管理员登录名 | **sqladmin** |
    | 密码 | 输入安全密码 |
    | 确认密码 | 确认该密码 |

1. 选择“查看 + 创建”，然后选择“创建” 。
1. 部署完成后，导航到 ***Azure SQL Server***（非 Azure SQL 数据库）的“**网络**”部分，然后：
    1. 将你的 IP 地址添加到防火墙规则中。 这样，就可以使用 SQL Server Management Studio（SSMS）或 Azure Data Studio 来管理数据库。
    1. 选中“允许 Azure 服务和资源访问此服务器”复选框****。 这将允许 Azure 函数应用访问数据库服务器。
    1. 保存所做更改。
1. 导航到 **Azure SQL Server** 的 **Microsoft Entra ID** 部分，并确保*取消选中*“**此服务器仅支持 Microsoft Entra 身份验证**”，然后“**保存**”更改（如已选中）。 此示例使用 SQL 身份验证，因此我们需要禁用仅支持 Entra。

> [!NOTE]
> 在生产环境中，需要确定要授予访问权限的类型以及从何处授予访问权限。 虽然仅选择 Entra 身份验证时函数会略有变化，但请注意，仍需启用“*允许 Azure 服务和资源访问此服务器*”，以允许 Azure 函数应用访问服务器。

### 克隆 GitHub 存储库

1. 打开 **Visual Studio Code**。

1. 克隆 GitHub 存储库并准备项目：

    1. 在 **Visual Studio Code** 中，通过按 **Ctrl+Shift+P** (Windows) 或 **Cmd+Shift+P** (Mac) 打开“**命令面板**”。
    1. 键入 **Git: 克隆**并选择 **Git: 克隆**。
    1. 在提示符中，输入以下 URL 以克隆存储库：
        ```bash
        https://github.com/MicrosoftLearning/mslearn-sql-dev.git
        ```

    1. 选择要克隆存储库的目标文件夹。

### 设置用于 JSON 数据的 Azure Blob 存储

现在，我们将设置 **Azure Blob 存储**来托管 **employees.json** 文件。 在 Azure 门户 和 **Visual Studio Code** 中执行这些步骤。

我们首先创建 Azure 存储帐户。

1. 在 **Azure 门户**中，转到“**存储帐户**”页。
1. 选择**创建**。
1. 填写必填字段：

    | 设置 | 值 |
    |---|---|
    | 订阅 | 订阅 |
    | 资源组 | 选择或创建新资源组 |
    | 存储帐户名称 | 选择一个全局唯一名称。 |
    | 区域 | 选择离你最近的区域 |
    | 主服务 | **Azure Blob 存储或 Azure Data Lake Storage Gen2** |
    | 性能 | 标准 |
    | 冗余 | 本地冗余存储 (LRS) |

1. 选择“查看 + 创建”，然后选择“创建” 。
1. 等待存储帐户创建完成。

现在我们有了一个帐户，接下来将 **employees.json** 上传到 Blob 存储。

1. 在 Azure 门户中，转到“**存储帐户**”页。
1. 选择存储帐户。
1. 导航到“**容器**”部分。
1. 创建名为 **jsonfiles** 的新容器。
1. 在容器中，单击“**上传**”并上传位于克隆目录中 **/Allfiles/Labs/04/blob-storage** 下的 **employees.json** 文件。

虽然我们可以允许匿名访问文件，但在本例中，我们要为此文件生成*共享访问签名 (SAS)*，以确保安全访问。

1. 在 **jsonfiles** 容器上，选择 **employees.json** 文件。
1. 从文件的上下文菜单中选择“**生成 SAS**”。
1. 查看设置并选择“**生成 SAS 和 URL**”。
1. 将会生成 Blob SAS 令牌和 Blob SAS URL。 复制并保存 **Blob SAS 令牌**和 **Blob SAS URL**，以便在后续步骤中使用。 关闭该窗口后，将无法再次访问令牌值。

现在我们应该有一个安全 URL 来访问 **employees.json** 文件，我们将继续对其进行测试。

1. 打开新的浏览器选项卡并粘贴 **Blob SAS URL**。
1. 应该能看到浏览器中显示的 **employees.json** 文件的内容，应如下所示：

    ```json
    {
        "employees": [
            {
                "EmployeeID": 1,
                "FirstName": "John",
                "LastName": "Doe",
                "Department": "HR"
            },
            {
                "EmployeeID": 2,
                "FirstName": "Jane",
                "LastName": "Smith",
                "Department": "Engineering"
            }
        ]
    }
    ```

### 将数据从 Blob 存储复制到 Azure SQL 数据库

现在，我们已准备好将数据从托管在 Azure Blob 存储上的 **employees.json** 文件导入到 Azure SQL 数据库。

我们首先需要在 Azure SQL 数据库中创建**主密钥**和**数据库作用域凭据**。

1. 使用 **SQL Server Management Studio (SSMS)** 或 **Azure Data Studio** 连接到 Azure SQL 数据库。
1. *如果还没有主密钥*，请运行以下 SQL 命令来创建主密钥：

    ```sql
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'YourStrongPassword!';
    ```

1. 接下来，通过运行以下 SQL 命令创建**数据库作用域凭据**以访问 Azure Blob 存储：

    ```sql
    CREATE DATABASE SCOPED CREDENTIAL MyBlobCredential
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE', 
    SECRET = '<your-sas-token>';
    ```

    将 ***<your-sas-token>*** 替换为前面生成的 **Blob SAS 令牌**。

1. 最后，需要**数据源**才能访问 Azure Blob 存储。 运行以下 SQL 命令以创建**数据源**：

    ```sql
    CREATE EXTERNAL DATA SOURCE MyBlobDataSource
    WITH (
        TYPE = BLOB_STORAGE,
        LOCATION = 'https://<your-storage-account-name>.blob.core.windows.net',
        CREDENTIAL = MyBlobCredential
    );
    ```

    将 ***<your-storage-account-name>*** 替换为 Azure 存储帐户的名称。

现在，所有内容都已设置完成，可以将数据从 **employees.json** 文件导入到 *Azure SQL 数据库*。

使用以下 SQL 命令从托管在 *Azure Blob 存储*上的 **employees.json** 文件导入数据：

```sql
SELECT EmployeeID
    , FirstName
    , LastName
    , Department
INTO dbo.employee_data
FROM OPENROWSET(
    BULK 'jsonfiles/employees.json',
    DATA_SOURCE = 'MyBlobDataSource',
    SINGLE_CLOB
) AS JSONData
CROSS APPLY OPENJSON(JSONData.BulkColumn, '$.employees')
WITH (
    EmployeeID INT '$.EmployeeID',
    FirstName NVARCHAR(50) '$.FirstName',
    LastName NVARCHAR(50) '$.LastName',
    Department NVARCHAR(50) '$.Department'
) AS EmployeeData;
```

此命令从 **Azure Blob 存储**中的 *jsonfiles* 容器读取 **employees.json** 文件，并将数据导入 Azure SQL 数据库的 **employee_data**表中。

现在可以运行以下 SQL 命令来验证数据导入： 

```sql
SELECT * FROM dbo.employee_data;
```

应该能看到 **employees.json** 文件中的数据被导入到 **employee_data** 表中。

---

## 使用 Azure 函数应用导出数据

在本实验室的这一部分中，你将在 C# 中创建一个 Azure 函数应用，以从 Azure SQL 数据库导出数据。 此函数将检索数据并将其作为 JSON 响应返回。

### 在 Visual Studio Code 中创建 Azure 函数应用

首先，在 Visual Studio Code 中创建 Azure 函数应用：

1. 打开 **Visual Studio Code**。
1. 在资源管理器窗格中，导航到 **/Allfiles/Labs/04/azure-functions** 文件夹。
1. 右键单击 **azure-functions** 文件夹，然后选择“**在集成终端中打开**”。
1. 在 VS Code 终端中，使用以下命令登录到 Azure：

    ```bash
    az login
    ```

1. （可选）如果你有多个订阅，请设置可用订阅：

    ```bash
    az account set --subscription <your-subscription-id>
    ```

1. 运行以下命令来创建 Azure 函数应用：

    ```bash
    $functionappname = "YourUniqueFunctionAppName"
    $resourcegroup = "YourResourceGroupName"
    $location = "YourLocation"
    # NOTE - The following should be a new storage account name where your Azure function will resided.
    # It should not be the same Storage Account name used to store the JSON file
    $storageaccount = "YourStorageAccountName"

    az storage account create --name $storageaccount --location $location --resource-group $resourcegroup --sku Standard_LRS
    
    az functionapp create --resource-group $resourcegroup --consumption-plan-location $location --runtime dotnet --name  $functionappname --os-type Linux --storage-account $storageaccount --functions-version 4
    
    ```

    ***将占位符替换为你自己的值。请勿使用用于 json 文件的存储帐户名称，此脚本需要创建新的存储帐户来存储 Azure 函数应用***。


### 在 Visual Studio Code 中创建新的函数应用

我们将在 Visual Studio Code 中创建一个新函数，以便从 Azure SQL 数据库导出数据：

可能需要将 Azure Functions 扩展添加到 Visual Studio Code（如果尚未添加）。 为此，可以在扩展窗格中搜索 **Azure Functions** 并安装它。

1. 在 Visual Studio Code 中，按 **Ctrl+Shift+P** (Windows) 或 **Cmd+Shift+P** (Mac) 打开命令面板。
1. 键入并选择“Azure Functions: 创建新项目”。
1. 选择“**函数应用**”目录。 选取 GitHub 克隆存储库的 **/Allfiles/Labs/04/azure-functions** 文件夹。
1. 选择 **C#** 作为语言。
1. 选择 **.Net 8.0 LTS** 作为运行时。
1. 选择 **HTTP 触发器**作为模板。
1. 调用函数 **ExportDataFunction**。
1. 创建命名空间 **Contoso.ExportFunction**。
1. 为函数设置**匿名**访问级别。

### 编写用于导出数据的 C# 代码

1. Azure 函数应用可能需要先安装几个包。 可以通过运行以下命令来安装它们：

    ```bash
    dotnet add package Microsoft.Data.SqlClient
    dotnet add package Newtonsoft.Json
    dotnet restore
    
    npm install -g azure-functions-core-tools@4 --unsafe-perm true

    ```

1. 将占位符函数代码替换为以下 C# 代码，以查询 Azure SQL 数据库并以 JSON 形式返回结果：

    ```csharp
    using System;
    using System.IO;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Azure.WebJobs;
    using Microsoft.Azure.WebJobs.Extensions.Http;
    using Microsoft.AspNetCore.Http;
    using Microsoft.Extensions.Logging;
    using Microsoft.Data.SqlClient;
    using Newtonsoft.Json;
    using System.Collections.Generic;
    using System.Threading.Tasks;
    
    public static class ExportDataFunction
    {
        [FunctionName("ExportDataFunction")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", Route = null)] HttpRequest req,
            ILogger log)
        {
            // Connection string to the database
            // NOTE: REPLACE THIS CONNECTION STRING WITH THE CONNECTION STRING OF YOUR AZURE SQL DATABASE
            string connectionString = "Server=tcp:yourserver.database.windows.net;Database=DataLabDatabase;User ID=youruserid;Password=yourpassword;Encrypt=True;";
            
            // List to hold employee data
            List<Employee> employees = new List<Employee>();
            
            try
            {
                // Establishing connection to the database
                using (SqlConnection conn = new SqlConnection(connectionString))
                {
                    await conn.OpenAsync();
                    var query = "SELECT EmployeeID, FirstName, LastName, Department FROM employee_data";
                    
                    // Executing the query
                    using (SqlCommand cmd = new SqlCommand(query, conn))
                    {
                        // Adding parameters to the query (if needed)
                        // cmd.Parameters.AddWithValue("@ParameterName", parameterValue);

                        using (SqlDataReader reader = await cmd.ExecuteReaderAsync())
                        {
                            // Reading data from the database
                            while (await reader.ReadAsync())
                            {
                                employees.Add(new Employee
                                {
                                    EmployeeID = (int)reader["EmployeeID"],
                                    FirstName = reader["FirstName"].ToString(),
                                    LastName = reader["LastName"].ToString(),
                                    Department = reader["Department"].ToString()
                                });
                            }
                        }
                    }
                }
            }
            catch (SqlException ex)
            {
                // Logging SQL errors
                log.LogError($"SQL Error: {ex.Message}");
                return new StatusCodeResult(500);
            }
            catch (System.Exception ex)
            {
                // Logging unexpected errors
                log.LogError($"Unexpected Error: {ex.Message}");
                return new StatusCodeResult(500);
            }
    
            // Returning the list of employees as a JSON response
            return new OkObjectResult(JsonConvert.SerializeObject(employees, Formatting.Indented));
        }
    
        // Employee class to hold employee data
        public class Employee
        {
            public int EmployeeID { get; set; }
            public string FirstName { get; set; }
            public string LastName { get; set; }
            public string Department { get; set; }
        }
    }
    ```

    *请记住，将 **connectionString** 替换为 Azure SQL 数据库的连接字符串，并在连接字符串中输入 sqladmin 密码。*

    > **备注：** 在生产环境中，请将访问权限限制为仅允许必要的 IP 地址。 此外，请考虑使用 Azure 函数应用的托管标识来访问数据库，而不是使用 SQL 身份验证。 有关详细信息，请参阅 [Microsoft Entra 中用于 Azure SQL 的托管标识](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true)。

1. 保存函数代码并确保 **.csproj** 文件包含 **Newtonsoft.Json** 包，以便将对象序列化为 JSON。 如果未包含，请添加它：

    ```xml
    <PackageReference Include="Newtonsoft.Json" Version="13.X.X" />
    ```

将 Azure 函数应用部署到 Azure。

### 将 Azure 函数应用部署到 Azure

1. 在 **Visual Studio Code** 集成终端中，运行以下命令将 Azure 函数部署到 Azure：

    ```bash
    func azure functionapp publish <your-function-app-name>
    ```

    将 ***<your-function-app-name>*** 替换为 Azure 函数应用的名称。

1. 等待部署完成。

### 获取 Azure 函数应用 URL

1. 打开 Azure 门户并导航到 Azure 函数应用。
1. 在“*概述*”部分的“*函数*”选项卡下，将看到新函数已列出，请将其选中。
1. 在“**代码 + 测试**”选项卡下，选择“**获取函数 URL**”。
1. 复制**默认（函数密钥）**，我们很快就会用到它。 该 URL 应如下所示：
   
   ```url
   https://YourFunctionAppName.azurewebsites.net/api/ExportDataFunction?code=2pjO0HqRyz_13DHQg8ga-ysdDWbDU_eHdtlixbAHLVEGAzFuplomUg%3D%3D
   ```

### 测试 Azure 函数应用

1. 部署完成后，可以通过将 HTTP 请求发送到之前从 Visual Studio Code 终端复制的函数密钥 URL 来测试函数：

    ```bash
    curl https://<your-function-app-name>.azurewebsites.net/api/ExportDataFunction?code=<the function key embedded to your function URL>
    ```

1. 响应应包含从 ***employee_data*** 表中导出的 JSON 格式的数据。

虽然此函数只是一个简单的示例，但可以将其扩展以包含更复杂的逻辑和数据处理，例如筛选、排序和聚合数据等。  代码还可以扩展到包含错误处理、日志记录和安全功能。

### 清理资源

完成实验室后，可以删除本练习中创建的资源，以避免产生额外费用：

- 删除 Azure SQL 数据库。
- 删除 Azure 存储帐户。
- 删除 Azure 函数应用。
- 删除包含资源的资源组。
