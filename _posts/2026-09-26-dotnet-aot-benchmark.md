---
layout: post
title: "How small is .NET native AOT, really? Benchmarked against Node, Python and regular .NET"
date: 2026-09-26
categories: dotnet aot performance docker benchmark
---

I keep a few templates for new services: a Node one, a Python one and a .NET one. Most of what I write are small things, like Discord bots and tiny APIs, and they all run on one small home server. There, memory and image size matter more to me than raw speed.

.NET has had native AOT for a while now. It compiles the whole app into one native binary: no JIT and no runtime to ship. The promise is a small image, a fast start and low memory. I wanted numbers before I moved my .NET template over, so I measured it against my other three templates.

For a quick answer, see [Results](#results).

# The contestants

| Image | What it is | Base image |
|---|---|---|
| node-server | Node 24, plain `node:http`, TypeScript run directly by Node | `node:24-slim` |
| python-server | FastAPI on one uvicorn worker | `python:3.14-slim` |
| dotnet-server (JIT) | ASP.NET Core on .NET 10, framework-dependent, with EF Core | `dotnet/aspnet:10.0-noble-chiseled` |
| **AOT spike** | ASP.NET Core minimal API on .NET 10, native AOT | `dotnet/runtime-deps:10.0-noble-chiseled` |

The first three are my real templates. The AOT spike is the smallest app that still does something, built for this test. All of it is below:

```csharp
// Program.cs
using System.Text.Json.Serialization;

var builder = WebApplication.CreateSlimBuilder(args);
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));

var app = builder.Build();

app.MapGet("/health", () => "ok");
app.MapGet("/greet", (string name) => new Greeting($"hello, {name}"));

app.Run();

internal sealed record Greeting(string Message);

[JsonSourceGenerationOptions(PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
[JsonSerializable(typeof(Greeting))]
internal sealed partial class AppJsonContext : JsonSerializerContext;
```

```xml
<!-- AotSpike.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <PublishAot>true</PublishAot>
    <InvariantGlobalization>true</InvariantGlobalization>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

```dockerfile
# Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0-noble-aot AS build
WORKDIR /src
COPY AotSpike.csproj ./
RUN dotnet restore -r linux-x64
COPY . .
RUN dotnet publish -c Release -r linux-x64 --no-restore -o /app

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-noble-chiseled
WORKDIR /app
COPY --from=build /app/AotSpike /app/appsettings.json ./
ENV ASPNETCORE_HTTP_PORTS=3000
EXPOSE 3000
ENTRYPOINT ["./AotSpike"]
```

`TreatWarningsAsErrors` matters here. If a trim or AOT warning shows up, the build fails instead of the app failing later at runtime. The spike builds with 0 warnings.

# Results

Every service ran alone under `docker run --cpus 2 --memory 512m`. I ran everything twice; the second run is in brackets.

| Image | Build (clean) | Size: disk / compressed | Startup (median) | Idle memory | Requests/s | p99 | Peak memory under load |
|---|---|---|---|---|---|---|---|
| node-server | 38.5 s [16.0] | 330 MB / 80.8 MB | 424 ms [379] | 28.7 MiB | 29,699 [29,914] | 3.52 ms | 41.6 MiB [42.8] |
| python-server | 17.7 s [19.8] | 222 MB / 53.0 MB | 1,415 ms [1,405] | 50.0 MiB | 3,279 [3,298] | 20.7 ms | 53.5 MiB [53.1] |
| dotnet-server (JIT) | 36.2 s [33.4] | 197 MB / 59.6 MB | 1,040 ms [1,078] | 37.1 MiB | 72,424 [72,804] | 1.34 ms | 49.1 MiB [46.5] |
| **AOT spike** | 100.4 s [87.6] | **34.2 MB / 10.4 MB** | **345 ms [350]** | **10.7 MiB** | 73,675 [75,849] | 1.31 ms | **16.1 MiB [16.3]** |

So, in short:

- **Size:** 34 MB on disk, 5–10× smaller than the rest. The binary itself is 9.4 MB. A 20 MB `.dbg` symbol file also comes out of the publish, but it stays out of the image.
- **Memory:** 10.7 MiB idle and 16 MiB at peak under load. That is 3–5× less than everything else.
- **Startup:** `docker run -d` alone takes 305–383 ms for every image. Take that away and the apps need about: AOT **~12 ms**, Node ~70 ms, .NET JIT ~710 ms (EF Core migrations included), Python ~1,070 ms.
- **Throughput:** the same as regular .NET. Both .NET images used the full 2 CPUs. Node and Python used about 1 CPU each, because each is one process with one event loop or worker. So this column says more about the runtime model than about the language.
- **Build time:** this is the cost. The restore takes 34–36 s (it pulls the native compiler packages) and the native compile ~13 s. The table's ~90–100 s also includes 39–50 s of installing clang by hand, which the `-aot` SDK image above already ships.

# Do my libraries work with AOT?

A small binary means nothing if the libraries I need break. My bots use [NetCord](https://netcord.dev/) for Discord and SQLite through Dapper. Plain Dapper generates code at runtime, which AOT can't do. [Dapper.AOT](https://aot.dapperlib.dev/) generates that code at build time instead.

I published a small probe app with all three. It builds a NetCord `GatewayClient` (without connecting) and runs a parameterised Dapper query against in-memory SQLite:

| Library | Version | AOT publish |
|---|---|---|
| NetCord | 1.0.0-beta.26 | 0 warnings |
| Dapper.AOT (+ Dapper 2.1.89) | 1.1.0 | 0 warnings with `[module: DapperAot]` |
| Microsoft.Data.Sqlite | 10.0.12 | 0 warnings |

The query returned the 2 rows I expected. 🎉

## The trap

Dapper.AOT does nothing until you turn it on with one attribute:

```csharp
[module: DapperAot]
```

Without it, the publish **still succeeds**, with 3 warnings (DAP005, IL2104, IL3053). Then the first query crashes at runtime with `PlatformNotSupportedException: Dynamic code generation is not supported`, thrown from `Dapper.SqlMapper.CreateParamInfoGenerator`.

This is why I keep `TreatWarningsAsErrors` on. With it, the build fails, and I don't find out from a crash.

Two more things:

- Dapper.AOT doesn't bring in Dapper. It rewrites Dapper's calls, so you still reference the `Dapper` package yourself.
- Generated interceptors can go stale. My first run without the attribute *passed*, because old generated files from an earlier build were still being compiled in. After a clean build it crashed as expected.

# How I tested it

- **Host:** WSL2 Ubuntu 26.04, 16 vCPU, 15 GB RAM, Docker Desktop 29.6.1 with the containerd image store. Date: 2026-09-26.
- **Build:** `docker build --no-cache`, with base images pulled beforehand. The measured AOT builds used `sdk:10.0` with clang and zlib1g-dev installed by apt; the Dockerfile above uses `sdk:10.0-noble-aot` instead, and its runtime stage is unchanged.
- **Startup:** the time from just before `docker run -d` to the first 200 on `/health`, polling every 20 ms. Median of 3 runs.
- **Idle memory:** `docker stats --no-stream`, 10 s after startup, with no traffic.
- **Load:** 20 s at 50 connections against `GET /health`, using [oha](https://github.com/hatoo/oha) from a container on the same Docker network. Every image had a 100% success rate.

  ```bash
  docker run --rm --network bench-net ghcr.io/hatoo/oha \
    -z 20s -c 50 --no-tui --output-format json http://svc:3000/health
  ```

- **Peak memory:** `docker stats` streaming during the load (about 1 sample per second). A 100 ms sampler on the container's cgroup and its `memory.peak` agreed with it within 0.3 MiB.
- **Probe:** `dotnet publish -c Release -r linux-x64 -p:PublishAot=true` in the same build image as the measured spike.

# Caveats

- **`/health` is a trivial endpoint.** It shows runtime overhead, not a real workload. My real AOT template (NetCord, SQLite, Dapper.AOT) came out heavier than the spike: a 64 MB image and 23.7 MiB idle, against 34 MB and 10.7 MiB. That is still well below the others, but check the numbers on your own app.
- **No warm-up** before the load test. That may favour AOT over JIT a little.
- **Startup is precise to a few ms only:** a 20 ms poll, plus the cost of starting `curl` each time, through Docker Desktop's port forwarding.
- **Build times include package downloads,** so they depend on the network.
- **Python had a handicap:** one uvicorn worker that logs every request. More workers would raise its number.
- **`CreateSlimBuilder` without an `appsettings.json` logs every request** at Information level. That slows it down quite a bit, so the spike got an `appsettings.json` before I measured.

# Closing thoughts

For small services that sit idle most of the day, native AOT is a clear win: a tenth of the image, a third of the memory, a start that is basically instant, and no loss in speed. You pay for it with a slower build and a bit more care with libraries. Check that they support AOT, and make warnings fail the build.

I'm moving my .NET template over :>
