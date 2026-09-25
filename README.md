# AlphaQuantum.URLCategorizationDatabase

A .NET 8 client that fills the gaps in a URL category file. Organisations that license [licensed URL categories with an API top-up](https://www.urlcategorizationdatabase.com) match most traffic against the file locally. The rest (new registrations, obscure hosts, fresh campaign sites) goes to this client, which classifies it live and returns content categories from the IAB taxonomy.

```bash
dotnet add package AlphaQuantum.URLCategorizationDatabase
```

## One call

```csharp
using AlphaQuantum.URLCategorizationDatabase;

var client = new URLCategorizationDatabaseClient(Environment.GetEnvironmentVariable("AQ_API_KEY")!);
var result = await client.ClassifyAsync("theverge.com");
Console.WriteLine(JsonSerializer.Serialize(result, new JsonSerializerOptions { WriteIndented = true }));
```

`ClassifyAsync` posts `query`, `data_type=url` and your key as a form, and returns the JSON reply as `Dictionary<string, JsonElement>`. The client does not rename or drop fields, so the online API reference describes the result exactly.

## A typical enrichment job

Say you have a SQL table of domains, from web analytics, CRM company websites or proxy logs, and a nullable `category_json` column. The job below classifies unlabelled rows with bounded parallelism:

```csharp
var pending = await db.QueryAsync<string>(
    "SELECT domain FROM sites WHERE category_json IS NULL");

await Parallel.ForEachAsync(pending,
    new ParallelOptions { MaxDegreeOfParallelism = 4, CancellationToken = ct },
    async (domain, token) =>
    {
        try
        {
            var r = await client.ClassifyAsync(domain, token);
            await db.ExecuteAsync(
                "UPDATE sites SET category_json = @j, categorized_at = SYSUTCDATETIME() WHERE domain = @d",
                new { j = JsonSerializer.Serialize(r), d = domain });
        }
        catch (ApiException ex) when (ex.StatusCode == 429)
        {
            await Task.Delay(TimeSpan.FromSeconds(30), token); // row stays NULL, picked up next run
        }
    });
```

The query in the example uses Dapper-style calls, but any data access layer works the same way. Points worth copying:

- **Four at a time.** Bounded parallelism is quick and polite. Lower it if 429s show up.
- **Store the raw JSON.** You can extract more fields later without paying for the calls again.
- **Store a timestamp.** It tells you which rows to refresh.
- **Failures stay NULL.** The next run retries them automatically, with no separate retry queue.

## Cleaning input first

Every call costs quota, so normalise before you query. Lower-case the host, strip `www.`, and drop paths when you only need a site-level label:

```csharp
static string Normalise(string raw)
{
    var s = raw.Contains("://") ? raw : "https://" + raw;
    var host = new Uri(s).Host.ToLowerInvariant();
    return host.StartsWith("www.") ? host[4..] : host;
}
```

On real analytics data, this step alone often removes a large share of duplicates.

## Domain, host or URL?

- Use the **domain** for company-level labels and most analytics.
- Use the **host** where subdomains are separate sites, for example on blogging and hosting platforms.
- Use the **full URL** when individual pages differ, such as articles on large publishers.

Choose per job and keep it consistent, so cache keys and table rows line up.

## When the licensed file updates

New releases of the file cover domains your job may have classified live. After each import, delete or ignore live results for domains the file now contains. The file stays authoritative, and your table stays a small, current supplement.

## Errors and cancellation

| Exception | Meaning |
|---|---|
| `ArgumentException` | Empty key at construction, or empty input |
| `ApiException` | Non-success status. Check `StatusCode`: 401 or 403 means a key or quota problem, 429 means slow down |
| `TaskCanceledException` | Timeout (30 seconds by default) or your token was cancelled |
| `JsonException` | The reply was not a JSON object |

The client makes one attempt per call. Add retries in your job, or through a resilience handler on an `IHttpClientFactory` registration, where you can see what is being retried.

## Supplying your own HttpClient

```csharp
var http = new HttpClient { Timeout = TimeSpan.FromSeconds(20) };
var client = new URLCategorizationDatabaseClient(key, http);
```

Pass one when you need a proxy, custom certificates or shorter timeouts. For tests, build the `HttpClient` on a stub `HttpMessageHandler`. The package's own tests check that the key and query travel in the form body and that errors surface as `ApiException`.

## Uses we see most

- **CRM hygiene**: tag company websites with an industry before leads are routed.
- **Analytics**: add a category dimension to referrer and outbound-click reports.
- **Brand safety**: screen placement lists before a campaign goes live.
- **Security reporting**: label outbound traffic by topic, then leave blocking to [block and allow categories for filters](https://www.webfilteringdatabase.com), which are designed for that job.

## AI hosts deserve their own label

A general taxonomy files AI products under technology or software. To [spot AI services inside category exports](https://www.aitoolsblocklist.com), cross-check the same hosts against the AI register. For an organisation-wide view of AI use, the log audit produces an [AI adoption report from DNS data](https://www.shadowaitools.com).

## Other packages

[The Go module](https://pkg.go.dev/github.com/explainableaixai/urlcategorizationdatabase-go) suits batch pipelines on Linux, [the Rust crate](https://crates.io/crates/urlcategorizationdatabase) suits high-throughput services, and PHP applications can use [the Packagist library](https://packagist.org/packages/urlcategorizationdatabase/urlcategorizationdatabase).

## License

MIT. IAB Tech Lab taxonomy names are used for compatibility only.
