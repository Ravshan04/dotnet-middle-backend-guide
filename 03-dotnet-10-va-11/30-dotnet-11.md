# 30. .NET 11 (RC2, GA 2026-noyabr)

> **Bir jumlada:** .NET 11 hozir (2026-oktabr) Release Candidate 2 bosqichida, go-live litsenziya bilan, GA noyabrda; u STS (2 yil qo'llab-quvvatlash) bo'ladi. Backend uchun eng muhim yangiliklar: C# 15 union types va closed hierarchies, runtime-async, System.Text.Json'da union va JSONL, ASP.NET Core'da async validatsiya, OpenAPI 3.2 va SignalR auth refresh, EF Core 11 da `FullJoin`, vector index va migration vositalari.
> **Nega Middle uchun:** Suhbatda "kuzatib borasizmi?" savoliga 3-4 ta aniq fakt va "enterprise LTS'da qoladi, nega?" degan mulohaza kifoya. Bu fayl RC2 holatiga asoslangan; GA'da kichik o'zgarishlar bo'lishi mumkin.
> **O'qish vaqti:** ~25 daqiqa

## Mundarija

1. [Muammo va yechim](#1-muammo-va-yechim)
2. [Qanday ishlaydi](#2-qanday-ishlaydi)
3. [Asosiy tushunchalar](#3-asosiy-tushunchalar)
4. [Amaliy misol](#4-amaliy-misol)
5. [Tipik xatolar va anti-patternlar](#5-tipik-xatolar-va-anti-patternlar)
6. [Best practices checklist](#6-best-practices-checklist)
7. [Suhbat savollari va javoblar](#7-suhbat-savollari-va-javoblar)
8. [Qo'shimcha manbalar](#8-qoshimcha-manbalar)

## 1. Muammo va yechim

Vaqt chizig'i:

| Sana | Hodisa |
|---|---|
| 2026-fevral | .NET 11 Preview 1 |
| 2026-sentabr 8 | RC1, go-live litsenziya, C# 15 default `net11.0` uchun |
| 2026-oktabr 13 | RC2, go-live |
| 2026-noyabr | GA (kutilmoqda), STS: 2028-noyabrgacha (24 oy) |
| 2028-noyabr | .NET 10 LTS qo'llab-quvvatlash tugaydi |

Enterprise uchun savol: "o'tish kerakmi?" Qoida: ishlab chiqarish tizimlari LTS'da (.NET 10) qoladi; .NET 11 STS, ya'ni 2028-noyabrda .NET 10 bilan bir vaqtda tugaydi, lekin oraliqda bir marta ko'proq migratsiya talab qiladi (.NET 12 LTS 2027-noyabrda chiqadi). .NET 11 ga o'tish mantiqli bo'lgan holatlar: aniq xususiyat kerak (union types, runtime-async, EF 11 vector index), yangi loyiha 2027'da baribir .NET 12 ga o'tadi, yoki jamoa har yili yangilash odatiga ega.

.NET 11 yechadigan muammolar:

- **Discriminated union yo'qligi.** `OneOf`, `ErrorOr` kutubxonalari yoki `abstract record` + `switch` bilan "exhaustive" bo'lmagan tekshiruv. C# 15 `union` va `closed` buni tilga kiritadi, kompilyator barcha holatlar qamrab olinganini tekshiradi.
- **Async overhead.** `async/await` state machine kompilyator darajasida: har await uchun allocation ehtimoli, stack trace'da `MoveNext`. Runtime-async await'ni runtime ichiga ko'chiradi.
- **Validatsiya async emas.** "Email bazada mavjudmi" tekshiruvi DataAnnotations bilan bo'lmasdi. `AsyncValidationAttribute`, `IAsyncValidatableObject`.
- **EF: FULL JOIN yo'q, vector index yo'q, jamoada migration konflikti kech aniqlanadi.** EF 11 har biriga javob beradi.

## 2. Qanday ishlaydi

### Runtime-async

Hozirgacha `async` metod kompilyator tomonidan state machine klassga aylantirilardi; `await` bu "state'ni saqla, continuation'ni ro'yxatdan o'tkaz, qayt". Runtime-async'da await runtime'ning o'zi tushunadigan operatsiya: JIT suspension nuqtalarini biladi, continuation'ni samaraliroq saqlaydi, `ExecutionContext` kerak bo'lmasa capture qilmaydi, stack trace'da `MoveNext` emas, haqiqiy metod nomlari. .NET 11 da `net11.0` loyihalar uchun preview flag'siz yoqiladi va runtime kutubxonalari shu rejimda kompilyatsiya qilingan. Kod o'zgarmaydi; foyda: kamroq allocation, tozaroq diagnostika.

### C# 15 union va closed

```csharp
public union PaymentResult(Paid, Declined, Pending);
```

Kompilyator `PaymentResult` uchun har case tipdan implicit konversiya va `switch` exhaustiveness tekshiruvini beradi. Runtime'da `UnionAttribute`/`IUnion` scaffolding (System.Runtime.CompilerServices) bilan ifodalanadi; System.Text.Json va ASP.NET Core buni biladi (JSON body'da union qabul qiladi). `closed` klass: faqat o'z assembly'sida meros olinadi, shuning uchun `switch` barcha avlodlarni biladi.

## 3. Asosiy tushunchalar

### 3.1 C# 15: union types

```csharp
public sealed record Paid(string TransactionId, decimal Amount);
public sealed record Declined(string Reason);
public sealed record Pending(DateTime RetryAt);

public union PaymentResult(Paid, Declined, Pending);

PaymentResult result = await gateway.ChargeAsync(order, ct);   // har case implicit konversiya

string message = result switch
{
    Paid p => $"To'landi: {p.TransactionId}",
    Declined d => $"Rad etildi: {d.Reason}",
    Pending pe => $"Kutilmoqda, qayta urinish {pe.RetryAt:HH:mm}",
    // default kerak emas: kompilyator exhaustiveness'ni tekshiradi
};
```

- **Nega kerak.** Result/Error naqshi uchun tashqi kutubxona shart emas; yangi case qo'shilsa, barcha `switch` lar kompilyatsiya xatosi beradi (unutilgan holat yo'q).
- **ASP.NET Core 11.** Union tipni Minimal API/MVC JSON body'da qabul qiladi va qaytaradi; route/query binding'da emas.

### 3.2 C# 15: closed hierarchies

```csharp
public closed record class OrderState;
public sealed record Draft : OrderState;
public sealed record Submitted(DateTime At) : OrderState;
public sealed record Shipped(string Tracking) : OrderState;

string Label(OrderState s) => s switch
{
    Draft => "qoralama",
    Submitted x => $"yuborilgan {x.At:d}",
    Shipped x => x.Tracking,
};   // ogohlantirish yo'q, hamma avlod qamrab olingan
```

`closed` implicit `abstract`; faqat o'z assembly'sida meros olinadi. Union'dan farqi: ierarxiya (umumiy bazaviy member'lar) kerak bo'lganda.

### 3.3 C# 15: boshqalar

- **Collection expression arguments.** `List<string> names = [with(capacity: 100), .. values];`, `HashSet<string> set = [with(StringComparer.OrdinalIgnoreCase), "a", "A"];`.
- **Extension indexers.** `extension(IEnumerable<int> seq) { public int this[int i] => seq.ElementAt(i); }`.
- **Labeled `break`/`continue`.** Ichma-ich sikllardan flag'siz chiqish: `continue outer;`.
- **Non-virtual static interface members** eski runtime'larda ham.
- **Memory safety (preview).** `unsafe` ni pointer e'lonidan emas, xotiraga kirishdan talab qilish yo'nalishi; `LangVersion=preview` va feature flag kerak. Backend uchun hozircha kuzatish darajasida.

### 3.4 Runtime va kutubxonalar

- **Runtime-async** (yuqorida). Amaliy: kodni o'zgartirmasdan async-og'ir servislarda kamroq allocation va aniq stack trace.
- **JIT.** Bounds check elimination, devirtualization, `SequenceEqual` constant folding, Arm SVE2, AVX-VNNI-512. Minimal apparat talablari ko'tarildi (juda eski CPU'lar qo'llab-quvvatlanmaydi; konteyner host'larini tekshiring).
- **NativeAOT** interface dispatch tezroq, kichikroq binar.
- **System.Text.Json.** Union qo'llab-quvvatlash, JSON Lines chiqarish (`JSONL`, log/stream eksport uchun), kengaytirilgan polymorphism va source generation, yangi raqam tiplari (`Decimal32/64/128`).
- **Process.** `Process.Signal(PosixSignal)`, `ProcessExitStatus`, run-and-capture helper'lar, `TryGetProcessById`. Ops-utilita yozadiganlar uchun.
- **Networking.** `DnsResolver` (tipli DNS so'rovlar: SRV, TXT), HTTP request-body compression, `SocketsHttpHandler.InitialHttp2MaxConcurrentStreams`, HTTP/3 stream prioritet.
- **Kriptografiya.** `X25519DiffieHellman`, AES KeyWrap, TLS hardening, eksperimental composite ML-KEM va HPKE.
- **Stream tiplar.** `ReadOnlyMemoryStream`, `WritableMemoryStream`, `ReadOnlySequenceStream`, `StringStream` (nusxasiz).
- **MemoryCache** uchun built-in OpenTelemetry metrikalari; `Activity` tracing qoidalar bilan deklarativ sozlanadi.
- **LINQ.** `FullJoin`, tuple qaytaradigan `Join`/`GroupJoin` overload'lari.
- **DataAnnotations.** `AsyncValidationAttribute`, `IAsyncValidatableObject`, `Validator.ValidateObjectAsync`; Options uchun `IAsyncStartupValidator`.

### 3.5 ASP.NET Core 11

- **Async validatsiya.** `AsyncValidationAttribute.IsValidAsync` yoki `IAsyncValidatableObject.ValidateAsync`; `AddValidation()` bilan ishlaydi. Validatsiya atributlari (`[ValidatableType]`, `[SkipValidation]`) endi eksperimental emas; built-in lokalizatsiya (`AddLocalization()` + `ValidationOptions.LocalizerProvider`).
- **Filter'lar binding xatosini ko'radi.** Parametr binding 400 bersa ham endpoint filter pipeline ishga tushadi; filter javobni almashtira oladi.
- **`[ShortCircuit]`.** Routing'dan keyin qolgan middleware'ni o'tkazib yuboradi (health endpoint uchun).
- **OpenAPI 3.2** (breaking: `Microsoft.OpenApi` 3.x), HTTP `QUERY` metodi, `FileContentResult` → `format: binary`, obsolete API'lar hujjatda belgilanadi, build-time generatsiya uchun environment tanlash.
- **SignalR auth refresh.** `EnableAuthenticationRefresh`, `CloseOnAuthenticationExpiration`, hub'da `OnAuthenticationRefreshedAsync`; .NET va TypeScript client'larda `withAuthenticationRefresh`. Uzoq ulanishlarda token muddati tugaganda ulanishni uzmasdan yangilash. Oddiy hub invocation'ni client `CancellationToken` bilan bekor qila oladi.
- **Negotiate auth** TLS channel binding bilan.
- **Blazor.** Static SSR'da TempData va Session, `CacheView`, auto-pause circuits, eksperimental AI komponentlari (backend o'quvchi uchun ma'lumot darajasida).

### 3.6 EF Core 11

- **`FullJoin`** → `FULL JOIN`; hybrid search (full-text + vector) uchun qulay.
- **Vector index va `VectorSearch()`** (SQL Server 2025, eksperimental): `HasVectorIndex(b => b.Embedding, "cosine")`, `db.Products.VectorSearch(p => p.Embedding, vec, "cosine").OrderBy(r => r.Distance).Take(5).WithApproximate()`. Vector ustunlar endi default `SELECT` ga kirmaydi (katta ma'lumot uzatilmaydi, 9x gacha tezlashuv).
- **Full-text:** katalog/indeks migration bilan, `FreeTextTable`/`ContainsTable` rank bilan.
- **JSON:** `JSON_CONTAINS` tarjimasi (compat 170), `EF.Functions.JsonContains`, `JsonPathExists`, JSON indekslar, complex type ichidagi property'ga key/index.
- **Complex types** TPT/TPC bilan, Cosmos'da, `Property(e => e.Details.Description)` lambda zanjiri.
- **SQL sifati:** to-one join'larda ortiqcha JOIN va `ORDER BY` olib tashlandi (22-29% tezlashuv benchmark'da), no-op `CAST` lar olib tashlanadi, `MaxBy`/`MinBy`, `GroupBy` yaxshilanishlari, `DateTimeOffset` tarjimalari, `DateTrunc`.
- **Migrations:** model snapshot'da oxirgi migration ID (branch merge konflikti erta ko'rinadi), `dotnet ef database update Name --add` (yaratish + qo'llash bir qadamda, Aspire/konteyner uchun), `--offline`/`--connection` opsiyalari, `.config/dotnet-ef.json` default sozlamalar, `--context "*"`, `ExcludeForeignKeyFromMigrations()`.
- **Default compat level 160** (SQL Server 2022): `LEAST`/`GREATEST` tarjimasi.
- Cosmos: transactional batch default, bulk execution, System.Text.Json serializer (breaking).

### 3.7 SDK

- `dotnet run -e KEY=VALUE`, file-based app'da `#:include` (bir nechta fayl), `dotnet format` qo'llab-quvvatlash.
- `dotnet test`: `--timeout`, `--maximum-failed-tests`, `--results-directory-layout`, `--no-dependencies`, affected-test (eksperimental); shablonlar xUnit v3 + MTP default.
- NativeAOT CLI va MSBuild server default yoqiq (tezroq `dotnet` buyruqlari); kichikroq installer (Linux/macOS).
- `dotnet sln` `.slnf` filtrlarni boshqaradi; konteyner publish reproducible digest, Podman multi-arch, `wslc`/`container` runtime'larni afzal ko'radi.
- `dotnet nuget add source --min-publish-age-hours` (supply-chain: yangi chiqarilgan paketni darhol olmaslik).

## 4. Amaliy misol

.NET 11 ni mavjud LTS loyihada sinab ko'rish: alohida branch, `global.json`, union bilan domain natija, async validatsiya.

```json
// global.json (branch: try-net11)
{ "sdk": { "version": "11.0.100-rc.2.25510.1", "rollForward": "latestPatch" } }
```

```xml
<!-- Directory.Build.props -->
<PropertyGroup>
  <TargetFramework>net11.0</TargetFramework>   <!-- C# 15 default -->
</PropertyGroup>
```

```csharp
// Domain: union bilan natija, tashqi kutubxonasiz
public sealed record Reserved(int ReservationId);
public sealed record OutOfStock(string Sku, int Available);
public sealed record ItemBlocked(string Sku);
public union ReserveResult(Reserved, OutOfStock, ItemBlocked);

public sealed class StockService(WarehouseDbContext db)
{
    public async Task<ReserveResult> ReserveAsync(string sku, int qty, CancellationToken ct)
    {
        var item = await db.StockItems.SingleAsync(s => s.Sku == sku, ct);
        if (item.Status == StockStatus.Blocked) return new ItemBlocked(sku);
        if (item.Quantity < qty) return new OutOfStock(sku, item.Quantity);
        item.Quantity -= qty;
        var r = new Reservation { Sku = sku, Qty = qty };
        db.Reservations.Add(r);
        await db.SaveChangesAsync(ct);
        return new Reserved(r.Id);
    }
}

// API: async validatsiya (ASP.NET Core 11) + union → HTTP mapping
public sealed class SkuExistsAttribute : AsyncValidationAttribute
{
    protected override async ValueTask<ValidationResult?> IsValidAsync(object? value, ValidationContext ctx, CancellationToken ct)
    {
        var repo = ctx.GetRequiredService<IStockRepository>();
        return value is string sku && await repo.ExistsAsync(sku, ct)
            ? ValidationResult.Success
            : new ValidationResult("SKU topilmadi", [ctx.MemberName!]);
    }
}

public record ReserveRequest([property: Required, SkuExists] string Sku, [property: Range(1, 10_000)] int Qty);

app.MapPost("/reservations", async (ReserveRequest req, StockService svc, CancellationToken ct) =>
{
    ReserveResult result = await svc.ReserveAsync(req.Sku, req.Qty, ct);
    return result switch
    {
        Reserved r => Results.Created($"/reservations/{r.ReservationId}", r),
        OutOfStock o => Results.Problem(statusCode: 409, title: "Qoldiq yetarli emas", detail: $"{o.Sku}: {o.Available} dona bor"),
        ItemBlocked b => Results.Problem(statusCode: 423, title: "Mahsulot bloklangan", detail: b.Sku),
    };
});
```

```bash
# Konteynerda sinash, host SDK'ga tegmasdan
docker run --rm -v "$PWD":/src -w /src mcr.microsoft.com/dotnet/sdk:11.0 dotnet test
```

Nima uchun shunday: union natija HTTP kodlariga `switch` bilan map qilinadi va yangi case qo'shilsa endpoint kompilyatsiya bo'lmaydi (unutilgan holat yo'q); async validatsiya bazaga murojaatni DataAnnotations ichida qiladi, FluentValidation shart emas; `global.json` branch'da, asosiy branch LTS'da qoladi.

## 5. Tipik xatolar va anti-patternlar

**❌ Production'ni RC'ga ko'chirish "go-live bor-ku" deb**

Nega yomon: go-live Microsoft qo'llab-quvvatlashini bildiradi, lekin uchinchi tomon paketlar (EF provider'lar, Hangfire, MassTransit) GA'gacha to'liq test qilinmagan bo'lishi mumkin. ✅ Staging'da sinash, GA + birinchi servicing'dan keyin ko'rib chiqish.

**❌ STS'ga o'tib, keyingi LTS'ga o'tish rejasini qilmaslik**

Nega yomon: .NET 11 2028-noyabrda tugaydi, .NET 12 LTS 2027-noyabrda chiqadi; rejasiz jamoa bir yil ichida yana migratsiya qiladi. ✅ Migratsiya kalendarini hujjatlashtiring.

**❌ Union'ni hamma joyda, oddiy `bool`/`exception` o'rnida ishlatish**

Nega yomon: API shovqini, o'quvchi uchun qo'shimcha tushuncha. ✅ Union haqiqatan 2+ muqobil natija bo'lganda; kutilmagan xato hali ham exception.

**❌ OpenAPI 3.2 ga o'tib, mijoz kod generatorini tekshirmaslik**

Nega yomon: ko'p generatorlar 3.2 ni hali tushunmaydi. ✅ `OpenApiVersion` ni 3.1 da qoldiring yoki generatorni yangilang.

**❌ Minimal apparat talablarini e'tiborsiz qoldirish**

Nega yomon: eski CPU'li on-prem server'larda .NET 11 runtime ishga tushmasligi mumkin. ✅ Host'larni tekshiring (x64: yangi instruction set talablari).

## 6. Best practices checklist

- [ ] Production LTS'da (.NET 10); .NET 11 faqat aniq sabab bilan. Sabab: STS 24 oy, qo'shimcha migratsiya.
- [ ] Migratsiya kalendari: .NET 12 LTS (2027-noyabr) → 2028-noyabrgacha o'tish. Sabab: .NET 10 va 11 bir vaqtda tugaydi.
- [ ] RC'ni alohida branch + `global.json` + Docker SDK image bilan sinang; host SDK'ni almashtirmang.
- [ ] Uchinchi tomon paketlar `net11.0` ni qo'llab-quvvatlashini tekshiring (EF provider, Npgsql, Hangfire, MassTransit, Polly).
- [ ] Union/closed'ni Result naqshi uchun kiriting; exhaustiveness'ga tayaning, `default` arm yozmang. Sabab: yangi case kompilyatsiya xatosi bersin.
- [ ] Runtime-async'dan foyda ko'rish uchun hech narsa qilmang, lekin allocation/stack trace o'zgarishini profiler bilan tasdiqlang.
- [ ] Async validatsiyani faqat tashqi resursga murojaat kerak bo'lganda; oddiy qoidalar sinxron qolsin.
- [ ] OpenAPI versiyasini mijoz generatorlariga qarab tanlang.
- [ ] EF 11: migration snapshot'dagi ID konfliktini merge jarayonining bir qismi qiling; `.config/dotnet-ef.json` ni repo'ga qo'shing.
- [ ] Vector qidiruv: `HasVectorIndex` + `WithApproximate()` faqat katta jadvallarda; kichikda aniq `VectorDistance`.
- [ ] SignalR'da uzoq ulanishlar bo'lsa auth refresh yoqing; token muddati tugaganda uzilish yo'qoladi.
- [ ] NuGet `--min-publish-age-hours` bilan supply-chain himoyasi.

## 7. Suhbat savollari va javoblar

1. **.NET 11 hozir qaysi bosqichda va enterprise unga o'tishi kerakmi?**
   2026-oktabr holatiga RC2, go-live litsenziya bilan, GA noyabrda. STS (24 oy, 2028-noyabrgacha). Enterprise odatda LTS'da (.NET 10, 2028-noyabrgacha) qoladi va 2027-noyabrdagi .NET 12 LTS'ni kutadi. .NET 11 ga o'tish aniq xususiyat kerak bo'lganda yoki jamoa yillik yangilash odatiga ega bo'lganda mantiqli.

2. **C# 15 union types nima beradi va `abstract record` dan farqi?**
   `union T(A, B, C)`: har case'dan implicit konversiya va `switch` exhaustiveness tekshiruvi; yangi case qo'shilsa barcha `switch` lar kompilyatsiya xatosi beradi. `abstract record` ierarxiyasida kompilyator barcha avlodlarni bilmaydi (boshqa assembly meros olishi mumkin), shuning uchun `default` arm kerak edi. `closed` modifier shu muammoni ierarxiya uchun hal qiladi.

3. **Runtime-async nima va kodga ta'siri?**
   `await` kompilyator state machine'i emas, runtime tushunadigan operatsiya bo'ladi: JIT suspension nuqtalarini biladi, continuation samaraliroq, `ExecutionContext` kerak bo'lmasa capture qilinmaydi, stack trace'da haqiqiy metod nomlari. Kod o'zgarmaydi; `net11.0` uchun default. Foyda: allocation kamayadi, diagnostika tozalanadi.

4. **ASP.NET Core 11 da validatsiya nimasi bilan o'zgardi?**
   Async validatsiya (`AsyncValidationAttribute`, `IAsyncValidatableObject`) `AddValidation()` bilan ishlaydi, shuning uchun "bazada mavjudmi" tekshiruvi DataAnnotations ichida; built-in lokalizatsiya; validatsiya atributlari eksperimental emas; filter'lar binding xatosini ko'radi.

5. **EF Core 11 da backend uchun eng foydali uchta narsa?**
   (1) SQL sifati: to-one join va ortiqcha `ORDER BY` olib tashlandi, no-op `CAST` yo'q, bu index ishlatilishini yaxshilaydi. (2) Migration vositalari: snapshot'da oxirgi migration ID (merge konflikti erta), `database update --add`, `.config/dotnet-ef.json`. (3) `FullJoin`, vector index + `VectorSearch`, JSON indekslar, `JSON_CONTAINS`.

6. **OpenAPI 3.2 ga o'tish xavfi nima?**
   `Microsoft.OpenApi` 3.x breaking change va ko'p mijoz kod generatorlari 3.2 ni hali qo'llab-quvvatlamaydi. `OpenApiVersion` bilan 3.1 da qolish mumkin.

7. **SignalR auth refresh qaysi muammoni hal qiladi?**
   Uzoq ulanishda JWT muddati tugasa, oldin ulanish uzilardi yoki eskirgan token bilan davom etardi. Endi server `EnableAuthenticationRefresh` bilan client'dan yangi token so'raydi, client avtomatik yangilaydi, hub `OnAuthenticationRefreshedAsync` da claims'ni yangilaydi; `CloseOnAuthenticationExpiration` bilan muddati tugasa yopadi.

8. **STS va LTS farqini va migratsiya strategiyasini tushuntiring.**
   LTS 36 oy (juft versiyalar: 8, 10, 12), STS 24 oy (toq: 9, 11). .NET 10 va 11 ikkalasi 2028-noyabrda tugaydi. LTS'da qolgan jamoa 2 yilda bir marta migratsiya qiladi; STS'ga o'tgan jamoa har yili. Strategiya: production LTS, R&D va yangi loyihalar kerak bo'lsa STS, kalendar hujjatlashtirilgan.

9. **RC'ni xavfsiz sinab ko'rish usuli?**
   Alohida branch, `global.json` da RC SDK, `TargetFramework=net11.0`, Docker `sdk:11.0` image bilan build/test (host SDK'ga tegmasdan), uchinchi tomon paketlar mosligi, integration testlar real baza bilan, performance taqqoslash. Production'ga GA va birinchi servicing'dan keyin.

10. **.NET 11 minimal apparat talablari nega muhim?**
    Runtime endi yangiroq instruction set'larni talab qiladi (x64 va Arm64 uchun); eski on-prem server'lar yoki ba'zi arzon VM'larda ishga tushmasligi mumkin. Konteyner host'larini va CI runner'larni oldindan tekshirish kerak.

## 8. Qo'shimcha manbalar

- [What's new in .NET 11 (Microsoft Learn, RC2 holati)](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-11/overview)
- [What's new in C# 15](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-15)
- [What's new in ASP.NET Core 11](https://learn.microsoft.com/aspnet/core/release-notes/aspnetcore-11)
- [What's New in EF Core 11](https://learn.microsoft.com/ef/core/what-is-new/ef-core-11.0/whatsnew)
- [Announcing .NET 11 Release Candidate 1 (.NET Blog)](https://devblogs.microsoft.com/dotnet/dotnet-11-rc-1/)
- [Announcing .NET 11 Release Candidate 2 (.NET Blog)](https://devblogs.microsoft.com/dotnet/dotnet-11-rc-2/)
- [.NET support policy (LTS/STS)](https://dotnet.microsoft.com/platform/support/policy/dotnet-core)
- [.NET 11 downloads](https://dotnet.microsoft.com/download/dotnet/11.0)
