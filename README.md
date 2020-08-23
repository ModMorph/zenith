# Zenith Sql

Combines the power of raw SQL with the convenience of an ORM

### Installation

`dotnet add package Zenith`

### Usage

Make sure `Microsoft.Extensions.Logging.Ilogger` has been registered to the service collection.

``` cs
services.AddTransient<ILogger, MyAppLogger>();
```

Add Zenith sql to the service collection.

```cs
// this will register a SQLite provider to the default profile using `connectionString` 
services.AddZenithSql(options =>
{
	options.ConnectionString = connectionString;
	options.Provider = new Zenith.Providers.SQLite.SQLiteProvider();
});
```
Now, inject `Zenith.IUnitOfWork` into the consumer service or call `processContainer.GetRequiredService<IUnitOfWork>()`.

```cs
using Zenith;

public class MyLogic 
{
	readonly IUnitOfWork unitOfWork;
	public MyLogic(IUnitOfWork unitOfWork)
	{
		this.unitOfWork = unitOfWork;
	}

	public async Task Select()
	{
		/// use unitOfWork ...
		// generate a select sql for BaseBossman table where 'b' is the table alias
		string sql = unitOfWork.CreateSelect<BaseBossman>("b");
		sql += @"
		WHERE b.BossmanId = @id;
		";
		using ISqlCommand command = unitOfWork.NewCommand(SqlTypeEnum.Select, sql);
		command.AddArgument("id", 2);
		BaseBossman row = await command.SelectSingleAsync<BaseBossman>();
	}
}
```

Where `BaseBossman` is a dto that looks like this:

```cs
[SqlMappable(nameof(BossmanId), "Bossman")]
public class BaseBossman
{
	public int BossmanId { get; set; }
	public int ContactId { get; set; }
	public string Name { get; set; }
}
```

For a table that looks like this:
```sql
CREATE TABLE [Bossman](
	[BossmanId] INT CONSTRAINT [PK_Bossman] PRIMARY KEY NOT NULL,
	[ContactId] INT CONSTRAINT FK_Bossman_Contact REFERENCES [Contact]([ContactId]) NOT NULL,
	[Name] NVARCHAR(1000) NOT NULL	
);
```

View the [src/SqlTest](Tests folder) for more examples.

### Profiles

More than one database provider or connection string can be registered using `options.Profile`.

`AddZenithSql` can be called many times as long as the `options.Profile` string is different on each call.

```cs
services.AddZenithSql(options =>
{
	options.Profile = "sql server profile";
	options.ConnectionString = connectionString;
	options.Provider = new Zenith.Providers.SqlServer.SqlServerProvider();
});
```

The profile can then be injected with `Func<string, IUnitOfWork>`

```cs
var factory = processContainer.GetRequiredService<Func<string, IUnitOfWork>>();
// this will be a IUnitOfWork that uses SqlServerProvider
IUnitOfWork unitOfWork = factory("sql server profile");
```

### Middleware

Middleware is a function that will be invoked on every database command that can view and modify command infomation before execution or view and modify the return data before it is returned to the consumer.

``` cs
// this middleware will populate `data.UpdatedOn` when object arguments that 
// extend `IUpdatable` are passed into `command.AddArguments(object);`
options.AddMiddleware((context, next) =>
{
	if ((SqlTypeEnum.Update | SqlTypeEnum.Insert).HasFlag(context.Type))
	{
		foreach (var item in context.ObjectParameters)
		{
			if (item.Data is IUpdatable u)
			{
				u.UpdatedOn = DateTime.UtcNow;
			}
		}
	}
	
	return next();
});
```

You can examine or modify the data as it is consumed, via middleware.

```cs
async IAsyncEnumerable<object> IterateData(IAsyncEnumerable<object> stream)
{
	await foreach (var row in stream)
	{
		// do something with each row of data ...
		yield return row;
	}
}

options.AddMiddleware(async (context, next) =>
{
	var stream = await next();
	return IterateData(stream);
});
```