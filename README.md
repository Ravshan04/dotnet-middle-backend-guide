# .NET Middle Backend Yo'l xaritasi

Enterprise web tizim quradigan **Middle darajadagi .NET backend dasturchi** bilishi kerak bo'lgan 30 ta mavzu. Har bir mavzu alohida faylda: muammo va yechim, ichki mexanizm, amaliy misollar, tipik xatolar, checklist va suhbat savollariga to'liq javoblar.

| | |
|---|---|
| **Til** | O'zbek (lotin), texnik atamalar inglizcha |
| **Versiya** | .NET 10 LTS (2025-noyabr, 2028-noyabrgacha qo'llab-quvvatlanadi), C# 14, EF Core 10 |
| **Keyingi** | .NET 11 RC2 (go-live, GA 2026-noyabr, STS) alohida bo'limda |
| **Holat** | 2026-oktabr |

## Qanday o'qish kerak

1. **Tartib bilan.** 1-bo'lim (C# va platforma) qolgan hamma narsaning poydevori. 2-bo'lim kundalik ishning 80 foizi. 3-bo'lim qisqa, "oxirgi versiyada nima yangi" savoliga javob.
2. **Har faylning oxiridagi savollarga** avval o'zingiz javob bering, keyin yozilgan javob bilan solishtiring.
3. **Kodni ishga tushiring.** Misollar .NET 10 uchun yozilgan. `dotnet run app.cs` (file-based apps) bilan kichik misollarni loyiha yaratmasdan sinash mumkin.
4. **Checklist'ni** o'z loyihangizga qo'llang: har bandning sababi yozilgan, shunchaki qoida emas.

## Mundarija

### 1. C# va .NET platforma asoslari

| # | Mavzu | Nima haqida |
|---|---|---|
| 01 | [C# tilini chuqur bilish](01-csharp-va-platforma/01-csharp-chuqur.md) | Generics, LINQ, records, pattern matching, nullable, Span, source generators |
| 02 | [Async/await va multithreading](01-csharp-va-platforma/02-async-await-multithreading.md) | State machine, thread pool, starvation, CancellationToken, Channel |
| 03 | [Memory va GC](01-csharp-va-platforma/03-memory-va-gc.md) | Avlodlar, LOH, Server GC, IDisposable, leak topish, ArrayPool |
| 04 | [Runtime, SDK va loyiha tuzilmasi](01-csharp-va-platforma/04-runtime-sdk-loyiha.md) | JIT, NativeAOT, LTS/STS, central package management |
| 05 | [SOLID va dizayn patternlar](01-csharp-va-platforma/05-solid-va-patternlar.md) | SOLID real misolda, Repository va EF, Strategy, Decorator, Options |

### 2. Web backend va enterprise arxitektura

| # | Mavzu | Nima haqida |
|---|---|---|
| 06 | [ASP.NET Core ichki tuzilishi](02-web-backend-va-arxitektura/06-aspnet-core-pipeline.md) | Middleware tartibi, Kestrel, routing, filters, Minimal API vs Controllers |
| 07 | [Dependency Injection](02-web-backend-va-arxitektura/07-dependency-injection.md) | Lifetime'lar, captive dependency, keyed services, Options, HttpClientFactory |
| 08 | [Konfiguratsiya va muhitlar](02-web-backend-va-arxitektura/08-konfiguratsiya.md) | Provider tartibi, sirlar, Key Vault, feature flags |
| 09 | [REST API dizayni](02-web-backend-va-arxitektura/09-rest-api-dizayni.md) | Status kodlar, ProblemDetails, versioning, keyset pagination, idempotency |
| 10 | [Validatsiya va xatolar](02-web-backend-va-arxitektura/10-validatsiya-va-xatolar.md) | FluentValidation, IExceptionHandler, Result pattern |
| 11 | [EF Core chuqur](02-web-backend-va-arxitektura/11-ef-core.md) | Change tracker, N+1, migrations, concurrency, interceptors, bulk |
| 12 | [SQL va ma'lumotlar bazasi](02-web-backend-va-arxitektura/12-sql-va-database.md) | Indekslar, execution plan, isolation levels, deadlock |
| 13 | [Authentication va Authorization](02-web-backend-va-arxitektura/13-authentication-authorization.md) | JWT, refresh token, OAuth2/OIDC, policy va resource-based authz |
| 14 | [Xavfsizlik](02-web-backend-va-arxitektura/14-xavfsizlik.md) | OWASP Top 10 ASP.NET Core'da, rate limiting, Data Protection |
| 15 | [Arxitektura uslublari](02-web-backend-va-arxitektura/15-arxitektura-uslublari.md) | Clean, Vertical Slice, CQRS, DDD, modular monolith |
| 16 | [Microservices va integratsiya](02-web-backend-va-arxitektura/16-microservices-integratsiya.md) | Broker, outbox, saga, resilience, gRPC, YARP |
| 17 | [Caching](02-web-backend-va-arxitektura/17-caching.md) | HybridCache, Redis, stampede, invalidation, ETag |
| 18 | [Background ishlar](02-web-backend-va-arxitektura/18-background-ishlar.md) | BackgroundService, Channel, Hangfire, Quartz |
| 19 | [Real-time](02-web-backend-va-arxitektura/19-real-time.md) | SignalR, backplane, Server-Sent Events |
| 20 | [Observability](02-web-backend-va-arxitektura/20-observability.md) | Structured logging, OpenTelemetry, health checks, dotnet-counters |
| 21 | [Testing](02-web-backend-va-arxitektura/21-testing.md) | xUnit, WebApplicationFactory, Testcontainers, architecture tests |
| 22 | [Deploy va DevOps](02-web-backend-va-arxitektura/22-deploy-devops.md) | Docker, CI/CD, Kubernetes, zero-downtime migration |
| 23 | [Fayl, hisobot va integratsiya](02-web-backend-va-arxitektura/23-fayl-hisobot-integratsiya.md) | Object storage, Excel/PDF, email, webhook |
| 24 | [Soft skills va jarayon](02-web-backend-va-arxitektura/24-soft-skills-jarayon.md) | Talabdan dizaynga, code review, ADR, mentorlik |

### 3. .NET 10 va .NET 11 yangiliklari

| # | Mavzu | Nima haqida |
|---|---|---|
| 25 | [C# 14](03-dotnet-10-va-11/25-csharp-14.md) | field keyword, extension members, null-conditional assignment |
| 26 | [ASP.NET Core 10](03-dotnet-10-va-11/26-aspnet-core-10.md) | Built-in validation, OpenAPI 3.1, SSE, passkeys |
| 27 | [EF Core 10](03-dotnet-10-va-11/27-ef-core-10.md) | LeftJoin, named query filters, complex types |
| 28 | [Runtime va SDK 10](03-dotnet-10-va-11/28-runtime-sdk-10.md) | File-based apps, JIT, NativeAOT, post-quantum crypto |
| 29 | [Microsoft.Extensions.AI](03-dotnet-10-va-11/29-microsoft-extensions-ai.md) | IChatClient, tool calling, RAG |
| 30 | [.NET 11 (RC2)](03-dotnet-10-va-11/30-dotnet-11.md) | C# 15 union types, runtime-async, EF Core 11, qachon o'tish |

## Har bir fayl tuzilishi

Barcha mavzular bir xil skeletda yozilgan, shunda ikkinchi faylga o'tganda qayerda nima borligini bilasiz:

1. **Muammo va yechim.** Bu narsa qaysi real og'riq tufayli bor.
2. **Qanday ishlaydi.** Kompilyator, runtime yoki framework ichida nima bo'ladi.
3. **Asosiy tushunchalar.** Har biri: ta'rif, nega kerak, kod, tipik xato.
4. **Amaliy misol.** Enterprise kontekstdagi to'liq kod.
5. **Tipik xatolar va anti-patternlar.** Yomon kod, oqibati, to'g'ri kod.
6. **Best practices checklist.**
7. **Suhbat savollari va to'liq javoblar.**
8. **Qo'shimcha manbalar.** Rasmiy hujjatlar.

## Hissa qo'shish

Xato yoki eskirgan ma'lumot topsangiz, Issue oching yoki Pull Request yuboring. Yangi .NET versiyasi chiqqanda 3-bo'lim yangilanadi.

## Litsenziya

[MIT](LICENSE). Matn va kod misollaridan erkin foydalanishingiz mumkin, manbaga havola qoldiring.
