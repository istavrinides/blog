---
tags:
- Azure SQL Managed Instance
- Azure Functions
- CLR Functions
---
# Calling Azure Functions from Azure SQL Managed Instance

I recently came across a situation where a customer wanted to migrate from on-premises SQL Server to Azure SQL Managed Instance. 
They where using CLR functions that relied on having access to the underlying file system for the purposes of their solution, which is not supported
in a managed service such as Azure SQL Managed Instance.

After getting some more details on the use case, the logic of the CLR function could be hosted in an Azure Function. The task at that point was investigating 
whether this was supported in Azure SQL Managed Instance.

## Does Azure SQL Managed Instance support CLR Functions?

First task was verifying the extent of support of CLR Functions in Azure SQL Managed Instance. From [this doc page](https://learn.microsoft.com/en-us/sql/relational-databases/clr-integration/common-language-runtime-integration-overview?view=sql-server-ver16), CLR functions are supported but it doesn't list any limitations. So next step was to try it out and see if it works.

## Let's build our CLR function

You can use your tool of choice to build the CLR Function (I went with Visual Studio 2022). I created a new .Net Framework Class Library project and rename the default `Class.cs` class file to `SQLExternalFunctions.cs`. The code follows:

```csharp
using System.IO;
using System.Net;

public static class SQLExternalFunctions
{
    /// <summary>
    /// Method used to call an Azure Function. Note the following
    ///     - Method should be static
    ///     - Method should be decorated with [Microsoft.SqlServer.Server.SqlFunction]
    /// </summary>
    /// <param name="functionName">Name of the Azure Function - useful if the CLR will be used to call multiple different Azure Functions with the same specification (or different ones, which could be also set as parameters)</param>
    /// <returns>As the Azure Function simply returns a string, we return the string. JSON could also be return and leverage the JSON capabilities of T-SQL</returns>
    [Microsoft.SqlServer.Server.SqlFunction]
    public static string CallAzureFunction(string functionHost, string functionName)
    {
        // Initialize a new web client from System.Net, as this is supported from Azure SQL Managed Instance
        using (var client = new WebClient())
        {
            // Setting up supported TLS versions
            ServicePointManager.Expect100Continue = true;
            ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls | SecurityProtocolType.Tls11 | SecurityProtocolType.Tls12;

            // Call the function (in this case, it allows for Anonymous invocation, a key could be parameterized)
            using (Stream data = client.OpenRead($"https://{functionHost}.azurewebsites.net/api/{functionName}}?"))
            {
                StreamReader reader = new StreamReader(data);
                string s = reader.ReadToEnd();
                return s;
            }
        }
    }
}
```

Once we compile the project, we will get back a dll. Since Azure SQL Managed Instance does not support any way to upload the dll, we need to get it's binary representation. For this purpose, I leverage code from [this blog](https://thedatacrew.com/sql-server-clr-function-on-azure-sql-server-managed-instance/) from The Data Crew, created a simple Console Application and noted the output.

## Preparing Azure SQL Managed Instance

Although Azure SQL Managed Instance supports CLR Functions, we will need to enable the support and also make some changes:

```sql
/* CLR is in advanced options, so let's enable that*/
exec sp_configure 'show advanced options', '1';
RECONFIGURE; 
/* Enable CLR support */
EXEC sp_configure 'clr enabled' , '1';
RECONFIGURE; 
/* Disable strict security - see below */
EXEC sp_configure 'clr strict security', '0';
RECONFIGURE; 
```

Since we will need to register our CLR with `EXTERNAL_ACCESS`, we need to set `clr strict security` to `0`. As per the [documentation](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/clr-strict-security?view=sql-server-ver16):

> CLR uses Code Access Security (CAS) in the .NET Framework, which is no longer supported as a security boundary. A CLR assembly created with PERMISSION_SET = SAFE may be able to access external system resources, call unmanaged code, and acquire sysadmin privileges. Beginning with SQL Server 2017 (14.x), an sp_configure option called clr strict security is introduced to enhance the security of CLR assemblies. clr strict security is enabled by default, and treats SAFE and EXTERNAL_ACCESS assemblies as if they were marked UNSAFE. The clr strict security option can be disabled for backward compatibility, but this is not recommended. Microsoft recommends that all assemblies be signed by a certificate or asymmetric key with a corresponding login that has been granted UNSAFE ASSEMBLY permission in the master database. SQL Server administrators can also add assemblies to a list of assemblies, which the Database Engine should trust. For more information, see sys.sp_add_trusted_assembly.

## Registering the assembly

Next, we need to add our assembly in the trusted assembly list. To do this, we need to hash the assembly binary, and we can do so with the following T-SQL code block:

```sql
DECLARE @myBinary varbinary(max) = <Binary representation of our dll>;
DECLARE @hash varbinary(64);

SELECT @hash = HASHBYTES('SHA2_512', @myBinary);
DECLARE @clrName nvarchar(4000) = 'TestFunction'
exec sys.sp_add_trusted_assembly @hash, @clrName
```

Once we have added our dll in the trusted assembly list, we can now register our dll:

```sql
CREATE ASSEMBLY TestAssembly AUTHORIZATION dbo
FROM
<Binary representation of our dll>
WITH PERMISSION_SET = EXTERNAL_ACCESS;
```

Note that above we use the `EXTERNAL_ACCESS` permission set as we are calling something externally to the Azure SQL Managed Instance.

## Creating the CLR Function and testing

Now, we can create our CLR Function:

```sql
CREATE FUNCTION dbo.testFunction(@val NVARCHAR(100)) RETURNS NVARCHAR(1000)
AS EXTERNAL NAME TestAssembly.[SQLExternalFunctions].CallAzureFunction;
```

Once created, we can now invoke it using T-SQL:

```sql
SELECT dbo.testFunction('callfromsqlmanagedinstance', 'test');
```

Hopefully the above is helpful to anyone looking to leverage this capability.

Live long and Cloud more!