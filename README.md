using Microsoft.Data.SqlClient;
using System.Data;

var sourceConnectionString =
    "Server=SERVIDOR_DESARROLLO;Database=MiBaseDatos;Trusted_Connection=True;TrustServerCertificate=True";

var targetConnectionString =
    "Server=localhost;Database=MiBaseDatos;Trusted_Connection=True;TrustServerCertificate=True";

var excludedTables = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
{
    "AuditLog",
    "Telemetry",
    "HugeEvents"
};

await using var sourceConnection = new SqlConnection(sourceConnectionString);
await using var targetConnection = new SqlConnection(targetConnectionString);

await sourceConnection.OpenAsync();
await targetConnection.OpenAsync();

Console.WriteLine("Obteniendo tablas...");

var tables = await GetTablesAsync(targetConnection, excludedTables);

Console.WriteLine($"Tablas encontradas: {tables.Count}");

Console.WriteLine("Deshabilitando constraints...");
await DisableConstraintsAsync(targetConnection);

try
{
    Console.WriteLine("Limpiando datos destino...");
    await DeleteDataAsync(targetConnection, tables);

    var current = 1;

    foreach (var table in tables)
    {
        Console.WriteLine($"[{current}/{tables.Count}] Copiando {table}");

        await CopyTableAsync(
            sourceConnection,
            targetConnection,
            table);

        current++;
    }

    Console.WriteLine("Reajustando identities...");

    foreach (var table in tables)
    {
        await ReseedIdentityAsync(
            targetConnection,
            table);
    }

    Console.WriteLine("Proceso finalizado correctamente.");
}
finally
{
    Console.WriteLine("Habilitando constraints...");
    await EnableConstraintsAsync(targetConnection);
}

static async Task<List<string>> GetTablesAsync(
    SqlConnection connection,
    HashSet<string> excludedTables)
{
    const string sql = """
        SELECT TABLE_NAME
        FROM INFORMATION_SCHEMA.TABLES
        WHERE TABLE_TYPE = 'BASE TABLE'
        ORDER BY TABLE_NAME
        """;

    var result = new List<string>();

    await using var cmd = new SqlCommand(sql, connection);
    await using var reader = await cmd.ExecuteReaderAsync();

    while (await reader.ReadAsync())
    {
        var tableName = reader.GetString(0);

        if (!excludedTables.Contains(tableName))
        {
            result.Add(tableName);
        }
    }

    return result;
}

static async Task DisableConstraintsAsync(SqlConnection connection)
{
    var sql = """
        EXEC sp_MSforeachtable
        'ALTER TABLE ? NOCHECK CONSTRAINT ALL'
        """;

    await using var cmd = new SqlCommand(sql, connection);

    await cmd.ExecuteNonQueryAsync();
}

static async Task EnableConstraintsAsync(SqlConnection connection)
{
    var sql = """
        EXEC sp_MSforeachtable
        'ALTER TABLE ? WITH CHECK CHECK CONSTRAINT ALL'
        """;

    await using var cmd = new SqlCommand(sql, connection);

    await cmd.ExecuteNonQueryAsync();
}

static async Task DeleteDataAsync(
    SqlConnection connection,
    IEnumerable<string> tables)
{
    foreach (var table in tables)
    {
        Console.WriteLine($"Borrando datos de {table}");

        var sql = $"DELETE FROM [{table}]";

        await using var cmd =
            new SqlCommand(sql, connection);

        await cmd.ExecuteNonQueryAsync();
    }
}

static async Task<bool> HasIdentityAsync(
    SqlConnection connection,
    string tableName)
{
    const string sql = """
        SELECT COUNT(*)
        FROM sys.identity_columns ic
        INNER JOIN sys.tables t
            ON ic.object_id = t.object_id
        WHERE t.name = @tableName
        """;

    await using var cmd =
        new SqlCommand(sql, connection);

    cmd.Parameters.AddWithValue(
        "@tableName",
        tableName);

    var result = (int)await cmd.ExecuteScalarAsync();

    return result > 0;
}

static async Task CopyTableAsync(
    SqlConnection sourceConnection,
    SqlConnection targetConnection,
    string tableName)
{
    var hasIdentity =
        await HasIdentityAsync(
            targetConnection,
            tableName);

    if (hasIdentity)
    {
        await using var identityOn =
            new SqlCommand(
                $"SET IDENTITY_INSERT [{tableName}] ON",
                targetConnection);

        await identityOn.ExecuteNonQueryAsync();
    }

    try
    {
        await using var selectCommand =
            new SqlCommand(
                $"SELECT * FROM [{tableName}]",
                sourceConnection);

        await using var reader =
            await selectCommand.ExecuteReaderAsync();

        using var bulkCopy =
            new SqlBulkCopy(
                targetConnection,
                SqlBulkCopyOptions.KeepIdentity,
                null);

        bulkCopy.DestinationTableName = tableName;
        bulkCopy.BatchSize = 10000;
        bulkCopy.BulkCopyTimeout = 0;

        var schema = reader.GetColumnSchema();

        foreach (var column in schema)
        {
            bulkCopy.ColumnMappings.Add(
                column.ColumnName,
                column.ColumnName);
        }

        await bulkCopy.WriteToServerAsync(reader);
    }
    finally
    {
        if (hasIdentity)
        {
            await using var identityOff =
                new SqlCommand(
                    $"SET IDENTITY_INSERT [{tableName}] OFF",
                    targetConnection);

            await identityOff.ExecuteNonQueryAsync();
        }
    }
}

static async Task ReseedIdentityAsync(
    SqlConnection connection,
    string tableName)
{
    var hasIdentity =
        await HasIdentityAsync(
            connection,
            tableName);

    if (!hasIdentity)
        return;

    var identityColumn =
        await GetIdentityColumnAsync(
            connection,
            tableName);

    if (identityColumn is null)
        return;

    var sql = $"""
        DECLARE @MaxValue BIGINT;

        SELECT @MaxValue =
            ISNULL(MAX([{identityColumn}]), 0)
        FROM [{tableName}];

        DBCC CHECKIDENT ('{tableName}',
                         RESEED,
                         @MaxValue);
        """;

    await using var cmd =
        new SqlCommand(sql, connection);

    await cmd.ExecuteNonQueryAsync();
}

static async Task<string?> GetIdentityColumnAsync(
    SqlConnection connection,
    string tableName)
{
    const string sql = """
        SELECT c.name
        FROM sys.identity_columns c
        INNER JOIN sys.tables t
            ON c.object_id = t.object_id
        WHERE t.name = @tableName
        """;

    await using var cmd =
        new SqlCommand(sql, connection);

    cmd.Parameters.AddWithValue(
        "@tableName",
        tableName);

    return (string?)await cmd.ExecuteScalarAsync();
}
