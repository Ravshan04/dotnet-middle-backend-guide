# 08. Konfiguratsiya va muhitlar

> **Bir jumlada:** Bitta build dev, staging va production'da ishlaydi, farq faqat konfiguratsiyada; provider'lar tartibi, sirlarni qayerda saqlash va noto'g'ri sozlamani deploy paytida (birinchi mijozdan oldin) tutish shu bobning mazmuni.
> **Nega Middle uchun:** "Qaysi qiymat g'olib" savoli har ikkinchi production incident'ining ildizi: env var yozilgan, lekin `appsettings.Production.json` uni bosib qo'ygan; connection string repo'ga tushgan; feature flag staging'da yoqiq, prod'da yo'q.
> **O'qish vaqti:** ~30 daqiqa

## Mundarija

- [1. Muammo va yechim](#1-muammo-va-yechim)
- [2. Qanday ishlaydi](#2-qanday-ishlaydi)
- [3. Asosiy tushunchalar](#3-asosiy-tushunchalar)
- [4. Amaliy misol](#4-amaliy-misol)
- [5. Tipik xatolar va anti-patternlar](#5-tipik-xatolar-va-anti-patternlar)
- [6. Best practices checklist](#6-best-practices-checklist)
- [7. Suhbat savollari va javoblar](#7-suhbat-savollari-va-javoblar)
- [8. Qo'shimcha manbalar](#8-qoshimcha-manbalar)

## 1. Muammo va yechim

Ombor tizimi deploy qilindi, health check yashil, lekin 20 daqiqadan keyin mijozlar SMS olmayotgani ma'lum bo'ldi. Tekshiruv: Kubernetes'da `Sms__ApiKey` env var bor, lekin kod `Sms:ApiKey`ni o'qiyapti va `appsettings.Production.json`da `"ApiKey": ""` yozilgan. Env var JSON'ni bosib o'tishi kerak edi, va bosib o'tgan ham: muammo boshqa joyda. Deploy skripti `ASPNETCORE_ENVIRONMENT=Prod` qo'ygan, `Production` emas; shuning uchun `appsettings.Production.json` umuman o'qilmagan, `appsettings.json`dagi test URL ishlagan va API key haqiqiy edi, lekin test server'ga ketdi. Ilova bu haqda bir og'iz ham gapirmadi, chunki hech qanday startup validatsiyasi yo'q edi.

Ikkinchi holat: dasturchi `appsettings.Development.json`ga haqiqiy production connection string'ni yozib commit qildi. Repo private, lekin 40 kishi ko'rdi; DB paroli almashtirildi, bu 2 kunlik ish bo'ldi.

Uchinchi holat: komissiya foizi `appsettings.json`da; moliya bo'limi uni o'zgartirishni so'radi; har safar yangi deploy kerak. `reloadOnChange` va `IOptionsMonitor` buni yechadi, lekin jamoa `IOptions` ishlatgan.

Yechim: konfiguratsiya tizimini provider'lar zanjiri sifatida tushunish, sirlarni kod va repo'dan tashqariga chiqarish, va noto'g'ri sozlamani ilova ishga tushganda tutadigan validatsiya.

## 2. Qanday ishlaydi

### 2.1 Provider'lar va kalit-qiymat tekisligi

`IConfiguration` ierarxik ko'rinsa ham, ichida tekis `Dictionary<string, string?>` to'plamlari bor. Har bir **provider** (JSON fayl, env var, command line, Key Vault) o'z lug'atini quradi; kalitlar `:` bilan ajratilgan yo'l: `Sms:Providers:0:Name`. JSON'dagi ichma-ich obyektlar va massivlar shu tekis shaklga aylanadi (massiv indeksi kalitning bir qismi).

`ConfigurationRoot` provider'lar ro'yxatini saqlaydi. `config["Sms:ApiKey"]` chaqirilganda u ro'yxatni **oxiridan boshiga** aylanib chiqadi va birinchi topilgan qiymatni qaytaradi. Shuning uchun "keyin qo'shilgan provider oldingisini bosib o'tadi". Kalitlar case-insensitive.

`WebApplication.CreateBuilder` qo'shadigan tartib (keyingisi oldingisini yozadi):

```mermaid
flowchart LR
    A[Host: DOTNET_ va ASPNETCORE_ env, args] --> B[appsettings.json]
    B --> C[appsettings.{Environment}.json]
    C --> D[User Secrets faqat Development]
    D --> E[Environment variables prefikssiz]
    E --> F[Command line args]
    F --> G[Siz qo'shganlar: Key Vault, KeyPerFile ...]
```

Siz `builder.Configuration.AddXxx()` deb qo'shgan har qanday provider ro'yxat oxiriga tushadi va hammasini bosib o'tadi. Key Vault'ni env var'dan **keyin** qo'shsangiz, env var bilan lokal override qilib bo'lmaydi; bu ko'pincha kutilmagan.

### 2.2 Environment nomi

`ASPNETCORE_ENVIRONMENT` (yoki `DOTNET_ENVIRONMENT`) qiymati `IHostEnvironment.EnvironmentName` bo'ladi. Standart nomlar: `Development`, `Staging`, `Production` (default, o'rnatilmagan bo'lsa). Nom erkin: `Staging2`, `UzProd` ham mumkin, lekin `IsDevelopment()`/`IsProduction()` faqat standart nomlarni taniydi, qolganiga `IsEnvironment("UzProd")`. Taqqoslash case-insensitive, lekin fayl nomi `appsettings.{EnvironmentName}.json` aynan shu nom bilan qidiriladi; Linux'da fayl tizimi case-sensitive, `appsettings.production.json` o'qilmaydi.

`launchSettings.json` faqat `dotnet run` va IDE uchun; `dotnet publish` natijasiga kirmaydi. Container'da muhit `ENV ASPNETCORE_ENVIRONMENT=Production` yoki deployment manifest orqali beriladi.

### 2.3 reloadOnChange va change token

`AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)` `PhysicalFileProvider` orqali `FileSystemWatcher` ochadi. Fayl o'zgarganda provider lug'atni qayta yuklaydi va `IChangeToken`ni ishga tushiradi. `IOptionsMonitor<T>` shu tokenga obuna: keshni tozalaydi, `OnChange` chaqiradi. `IConfiguration`ning o'zi darhol yangi qiymatni qaytaradi; `IOptions<T>` esa yo'q (u bir marta hisoblangan).

Container nozikliklari: Kubernetes ConfigMap volume sifatida mount qilinsa, yangilanish atomik symlink almashinuvi orqali bo'ladi; `FileSystemWatcher` ba'zi muhitlarda (NFS, Docker Desktop volume) hodisani olmaydi. `DOTNET_USE_POLLING_FILE_WATCHER=1` env var polling'ga o'tkazadi (4 soniyada bir tekshiradi). `subPath` bilan mount qilingan fayl umuman yangilanmaydi: Kubernetes uni nusxalaydi, symlink emas.

Yana bir ma'lum xulq: `OnChange` bitta o'zgarish uchun ikki marta chaqirilishi mumkin (ba'zi muharrirlar faylni ikki bosqichda yozadi). Callback idempotent bo'lsin.

### 2.4 Binding: IConfiguration'dan obyektga

`config.GetSection("Sms").Get<SmsOptions>()` yoki `Bind(obj)` reflection bilan public property'larni kalitlar bo'yicha to'ldiradi: `Sms:TimeoutSeconds` → `TimeoutSeconds`. Tip konvertatsiyasi `TypeConverter` orqali: `"5"` → `int`, `"00:00:30"` → `TimeSpan`, `"true"` → `bool`, enum nomi → enum. Massiv/ro'yxat `0`, `1`, ... indekslaridan. Private setter'lar default'da to'ldirilmaydi (`BindNonPublicProperties = true` bilan mumkin). `init` property'lar to'ldiriladi (binder reflection bilan yozadi).

.NET 8+ da `ConfigurationBinder` source generator (`<EnableConfigurationBindingGenerator>true</EnableConfigurationBindingGenerator>`) reflection'siz kod yaratadi; Native AOT uchun shart, oddiy ilovada ixtiyoriy.

## 3. Asosiy tushunchalar

### 3.1 Env var nomlash

- **Ta'rif:** Env var'da `:` ko'p shell'larda ishlamaydi, shuning uchun ierarxiya `__` (ikki pastki chiziq) bilan beriladi. `Sms__ApiKey` → `Sms:ApiKey`. Massiv: `Sms__Providers__0__Name`.
- **Connection string prefikslari:** `ConnectionStrings__Orders` oddiy usul. Azure merosi: `SQLAZURECONNSTR_Orders`, `MYSQLCONNSTR_`, `CUSTOMCONNSTR_` prefiksli env var'lar avtomatik `ConnectionStrings:Orders`ga aylanadi.
- **Prefiksli provider:** `AddEnvironmentVariables("WMS_")` faqat `WMS_` bilan boshlanganlarni oladi va prefiksni olib tashlaydi: `WMS_Sms__ApiKey` → `Sms:ApiKey`. Bir server'da bir nechta ilova bo'lsa to'qnashuvni oldini oladi.

```bash
# Linux/Kubernetes
export ConnectionStrings__Orders="Host=pg;Database=orders;Username=app;Password=..."
export Sms__TimeoutSeconds=10
export Orders__AllowedWarehouses__0=TAS-01
export Orders__AllowedWarehouses__1=SAM-02
```

- **Tipik xato:** `Sms:ApiKey` deb env var yozish (Linux'da shell qabul qilmaydi yoki ilova ko'rmaydi); `.NET` konfiguratsiya `:`ni ham taniydi, lekin shell `:` bilan env var yaratishga ruxsat bermaydi.

### 3.2 User Secrets

- **Ta'rif:** Development uchun sirlarni loyiha papkasidan tashqarida, `~/.microsoft/usersecrets/<UserSecretsId>/secrets.json` faylida saqlash. Loyihaga faqat `<UserSecretsId>` GUID'i yoziladi (csproj'da), sirning o'zi emas.
- **Nega kerak:** `appsettings.Development.json` repo'da; unga parol yozilsa, tarixda abadiy qoladi.

```bash
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Orders" "Host=localhost;Database=orders;Username=dev;Password=dev"
dotnet user-secrets set "Sms:ApiKey" "test-key"
dotnet user-secrets list
```

Faqat `Development` muhitida o'qiladi (`CreateBuilder` shunday sozlaydi). Shifrlanmagan: bu repo'dan ajratish vositasi, xavfsiz saqlash vositasi emas.

- **Tipik xato:** User secrets'ni CI yoki production'da kutish. U yerda `UserSecretsId` papkasi yo'q, provider jim o'tib ketadi (optional), qiymatlar `null`.

### 3.3 Production sirlari: Key Vault, Vault, Kubernetes Secrets

- **Azure Key Vault:** `Azure.Extensions.AspNetCore.Configuration.Secrets` paketi. Sir nomida `:` bo'lmaydi, `--` ishlatiladi: `Sms--ApiKey` → `Sms:ApiKey`. Autentifikatsiya `DefaultAzureCredential` (managed identity, lokalda `az login`).

```csharp
if (!builder.Environment.IsDevelopment())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri($"https://{builder.Configuration["KeyVault:Name"]}.vault.azure.net/"),
        new DefaultAzureCredential());
}
```

- **HashiCorp Vault:** ikki yo'l. (1) Vault Agent / CSI driver sirlarni pod ichida fayl sifatida mount qiladi, ilova `AddKeyPerFile` yoki `AddJsonFile` bilan o'qiydi, kod Vault haqida bilmaydi. (2) `VaultSharp` kutubxonasi bilan ilova o'zi Vault API'ga murojaat qiladi va `IConfigurationProvider` yozasiz. Birinchi yo'l soddaroq va ilovani Vault'ga bog'lamaydi.
- **Kubernetes Secrets:** `envFrom: secretRef` bilan env var (eng oddiy) yoki volume sifatida `/run/secrets/<name>` fayllar. Fayl varianti uchun `builder.Configuration.AddKeyPerFile("/run/secrets", optional: true)`: har fayl nomi kalit, ichi qiymat; `__` fayl nomida `:`ga aylanadi. Volume varianti yangilanadi (env var yangilanmaydi, pod restart kerak).

```csharp
builder.Configuration.AddKeyPerFile("/run/secrets", optional: true, reloadOnChange: true);
```

Kubernetes Secret default'da faqat base64, shifrlanmagan: etcd'da encryption at rest yoqilganini va RBAC cheklanganini tekshiring; yoki External Secrets Operator orqali Vault/Key Vault'dan sinxronlang.

### 3.4 Strongly-typed options va startup validatsiyasi

- **Ta'rif:** `IConfiguration["Sms:ApiKey"]` kodning har joyida tarqalib ketadi va kalit xatosi runtime'da `null` beradi. Options klassi bitta joyda bog'lanadi, validatsiya qilinadi, DI orqali keladi.
- **Nega startup'da:** Validatsiya `IOptions<T>.Value`da (birinchi so'rovda) ishlasa, deploy "muvaffaqiyatli" ko'rinadi va birinchi mijoz 500 oladi. `ValidateOnStart()` xatoni host ishga tushishida chiqaradi: pod `CrashLoopBackOff`ga tushadi, rollout to'xtaydi, eski versiya ishlashda davom etadi.

```csharp
public sealed class PaymeOptions
{
    public const string Section = "Payments:Payme";
    [Required, MinLength(8)] public string MerchantId { get; init; } = "";
    [Required] public string SecretKey { get; init; } = "";
    [Required, Url] public string CheckoutUrl { get; init; } = "";
    [Range(1, 300)] public int CallbackTimeoutSeconds { get; init; } = 30;
    public bool TestMode { get; init; }
}

builder.Services.AddOptions<PaymeOptions>()
    .BindConfiguration(PaymeOptions.Section)
    .ValidateDataAnnotations()
    .Validate(o => builder.Environment.IsDevelopment() || !o.TestMode,
              "Production'da Payme TestMode yoqiq bo'lishi mumkin emas")
    .ValidateOnStart();
```

Murakkab qoidalar (bir nechta option o'rtasida, DI'dagi servisga bog'liq) uchun `IValidateOptions<T>` klassi:

```csharp
public sealed class PaymeOptionsValidator(IHostEnvironment env) : IValidateOptions<PaymeOptions>
{
    public ValidateOptionsResult Validate(string? name, PaymeOptions o)
    {
        var errors = new List<string>();
        if (env.IsProduction() && o.CheckoutUrl.Contains("test", StringComparison.OrdinalIgnoreCase))
            errors.Add("Production'da test checkout URL ishlatilmoqda");
        if (o.SecretKey.Length < 16) errors.Add("SecretKey 16 belgidan qisqa");
        return errors.Count == 0 ? ValidateOptionsResult.Success : ValidateOptionsResult.Fail(errors);
    }
}
builder.Services.AddSingleton<IValidateOptions<PaymeOptions>, PaymeOptionsValidator>();
```

- **Tipik xato:** Options klassida `string ApiKey { get; set; } = null!;` va `[Required]` yo'q: bo'sh qiymat jim o'tadi, tashqi API 401 qaytaradi, sabab topish soatlab.

### 3.5 Feature flags: Microsoft.FeatureManagement

- **Ta'rif:** Funksiyani kodda qoldirib, konfiguratsiya orqali yoqish/o'chirish. Deploy va release ajratiladi: kod chiqdi, lekin flag o'chiq; keyin flag yoqiladi (yoki foizlab, yoki ma'lum tenant'larga).
- **Paket:** `Microsoft.FeatureManagement.AspNetCore` (4.x). Konfiguratsiya bo'limi `FeatureManagement`; `IFeatureManager.IsEnabledAsync("name")`; `[FeatureGate("name")]` atributi controller/action'ga (flag o'chiq bo'lsa 404); Minimal API'da endpoint filter yozasiz yoki handler ichida tekshirasiz.
- **Filterlar:** `Percentage` (foiz), `TimeWindow` (vaqt oralig'i), `Targeting` (foydalanuvchi/guruh bo'yicha), o'zingizning `IFeatureFilter`ingiz (masalan, tenant bo'yicha).

```json
{
  "FeatureManagement": {
    "NewCheckout": false,
    "KeysetPagination": true,
    "ClickPayments": {
      "EnabledFor": [
        { "Name": "Percentage", "Parameters": { "Value": 20 } }
      ]
    }
  }
}
```

```csharp
builder.Services.AddFeatureManagement(); // "FeatureManagement" bo'limini o'qiydi, reloadOnChange ishlaydi

app.MapPost("/checkout", async (CheckoutRequest req, IFeatureManager features, ICheckoutService svc, CancellationToken ct) =>
    await features.IsEnabledAsync("NewCheckout")
        ? await svc.CheckoutV2Async(req, ct)
        : await svc.CheckoutAsync(req, ct));
```

Tenant bo'yicha filter:

```csharp
[FilterAlias("Tenant")]
public sealed class TenantFeatureFilter(ITenantContext tenant) : IFeatureFilter
{
    public Task<bool> EvaluateAsync(FeatureFilterEvaluationContext ctx)
    {
        var allowed = ctx.Parameters.Get<string[]>("Tenants") ?? [];
        return Task.FromResult(allowed.Contains(tenant.Code, StringComparer.OrdinalIgnoreCase));
    }
}
builder.Services.AddFeatureManagement().AddFeatureFilter<TenantFeatureFilter>();
```

- **Tipik xato:** Flag'ni yillab o'chirmaslik: har flag ikki yo'l, ikki test. Flag yaratilganda o'chirish sanasi (yoki "100% ga chiqqach 2 sprint") belgilansin.

### 3.6 Environment-specific sozlamalar

- Fayllar: `appsettings.json` (umumiy default'lar, sirsiz), `appsettings.Development.json` (lokal URL'lar, verbose log), `appsettings.Staging.json`, `appsettings.Production.json` (log darajasi, timeout'lar). Sirlar hech bir faylda emas.
- Kodda tarmoqlanish: `if (env.IsDevelopment())` faqat infratuzilma uchun (developer exception page, user secrets, Swagger UI). Biznes mantiq muhitga qarab tarmoqlanmasin: "staging'da SMS yuborma" degan shart konfiguratsiya (`Sms:Enabled=false` yoki fake provider) orqali.
- Muhit bo'yicha DI: `if (env.IsDevelopment()) services.AddSingleton<ISmsSender, ConsoleSmsSender>(); else services.AddHttpClient<ISmsSender, EskizSmsSender>();` qabul qilinadi, lekin yaxshisi `Sms:Provider` kaliti: `"Console"` yoki `"Eskiz"`.

### 3.7 Sirlarni log'ga tushirmaslik

- `IConfiguration`ni to'liq log'ga chiqarmang. Debug uchun `((IConfigurationRoot)config).GetDebugView(ctx => ctx.Key.Contains("Password") || ctx.Key.Contains("Key") || ctx.Key.Contains("Secret") ? "***" : ctx.Value)` (.NET 7+ overload) ishlating.
- Options obyektini `{@Options}` bilan destructure qilib log'ga yozmang: Serilog hamma property'ni yozadi. Sir bo'lgan property'ga `[JsonIgnore]` foyda bermaydi (Serilog boshqa mexanizm); Serilog uchun `Destructurama.Attributed` `[NotLogged]` yoki alohida "ko'rinadigan" DTO.
- Connection string'ni xato xabarida chiqarmang: `NpgsqlException` xabari parolni o'z ichiga olmaydi, lekin sizning `throw new Exception($"Ulanib bo'lmadi: {cs}")` oladi.
- EF Core `EnableSensitiveDataLogging()` faqat Development'da: u SQL parametr qiymatlarini (mijoz telefoni, karta so'nggi raqamlari) log'ga yozadi.
- HTTP logging (`UseHttpLogging`) `Authorization` sarlavhasini default'da yashiradi, lekin `X-Api-Key` kabi custom sarlavhalarni yashirmaydi: `HttpLoggingOptions.RequestHeaders` ro'yxatini aniq bering.

### 3.8 Konfiguratsiya xatolarini erta topish

| Usul | Qachon xato chiqadi | Nimani tutadi |
|---|---|---|
| `ValidateOnStart()` | Host start | Bo'sh/noto'g'ri qiymat, diapazon, custom qoida |
| `GetRequiredSection("Sms")` | Chaqirilganda (Program.cs'da bo'lsa start) | Bo'lim umuman yo'q |
| `GetConnectionString("X") ?? throw` | Program.cs'da | Connection string yo'q |
| Health check (`AddNpgSql`, `AddUrlGroup`) | Readiness probe | Qiymat bor, lekin ulanib bo'lmaydi |
| `ValidateOnBuild` (DI) | Build | Options'ga bog'liq servis yaratilmaydi |
| Integratsion test `WebApplicationFactory` | CI | Noto'g'ri bind (kalit nomi xato) |

```csharp
// Program.cs: yo'q bo'lsa start'da yiqilsin, birinchi so'rovda emas
var ordersCs = builder.Configuration.GetConnectionString("Orders")
    ?? throw new InvalidOperationException("ConnectionStrings:Orders sozlanmagan");
builder.Configuration.GetRequiredSection("Payments:Payme"); // bo'lim yo'q bo'lsa exception
```

## 4. Amaliy misol

Buyurtma servisining to'liq konfiguratsiya qatlami: fayllar, env var, sirlar, options, flag, health check va startup tekshiruvi.

```json
// appsettings.json: sirsiz default'lar
{
  "Logging": { "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" } },
  "Orders": { "MaxLinesPerOrder": 50, "AbandonedAfter": "02:00:00", "Currency": "UZS" },
  "Sms": { "Provider": "Console", "Sender": "4546", "TimeoutSeconds": 5 },
  "Payments": { "Payme": { "CheckoutUrl": "https://test.paycom.uz", "TestMode": true, "CallbackTimeoutSeconds": 30 } },
  "FeatureManagement": { "NewCheckout": false }
}
```

```json
// appsettings.Production.json: faqat farqlar, sirsiz
{
  "Logging": { "LogLevel": { "Default": "Warning" } },
  "Sms": { "Provider": "Eskiz", "BaseUrl": "https://notify.eskiz.uz" },
  "Payments": { "Payme": { "CheckoutUrl": "https://checkout.paycom.uz", "TestMode": false } }
}
```

```yaml
# Kubernetes deployment (qisqartirilgan): sirlar Secret'dan, env nomi aniq
env:
  - name: ASPNETCORE_ENVIRONMENT
    value: Production
  - name: DOTNET_USE_POLLING_FILE_WATCHER
    value: "1"
  - name: ConnectionStrings__Orders
    valueFrom: { secretKeyRef: { name: orders-secrets, key: orders-cs } }
  - name: Sms__ApiKey
    valueFrom: { secretKeyRef: { name: orders-secrets, key: sms-api-key } }
  - name: Payments__Payme__MerchantId
    valueFrom: { secretKeyRef: { name: orders-secrets, key: payme-merchant } }
  - name: Payments__Payme__SecretKey
    valueFrom: { secretKeyRef: { name: orders-secrets, key: payme-secret } }
volumeMounts:
  - name: feature-flags
    mountPath: /app/config          # ConfigMap: appsettings.Flags.json, symlink yangilanadi
```

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Qo'shimcha fayl: flag'lar ConfigMap orqali, deploy'siz o'zgaradi
builder.Configuration.AddJsonFile("/app/config/appsettings.Flags.json", optional: true, reloadOnChange: true);
// Env var'lar bundan keyin yana o'qilsin, aks holda Flags fayli env var'ni bosib qo'yadi
builder.Configuration.AddEnvironmentVariables();

// --- Options: har biri bind + validatsiya + start'da tekshiruv ---
builder.Services.AddOptions<OrderOptions>().BindConfiguration("Orders")
    .ValidateDataAnnotations().ValidateOnStart();
builder.Services.AddOptions<SmsOptions>().BindConfiguration("Sms")
    .ValidateDataAnnotations()
    .Validate(o => o.Provider != "Eskiz" || (!string.IsNullOrEmpty(o.ApiKey) && !string.IsNullOrEmpty(o.BaseUrl)),
              "Sms:Provider=Eskiz uchun ApiKey va BaseUrl shart")
    .ValidateOnStart();
builder.Services.AddOptions<PaymeOptions>().BindConfiguration(PaymeOptions.Section)
    .ValidateDataAnnotations().ValidateOnStart();
builder.Services.AddSingleton<IValidateOptions<PaymeOptions>, PaymeOptionsValidator>();

// --- Provider tanlovi konfiguratsiyadan, muhit nomidan emas ---
switch (builder.Configuration["Sms:Provider"])
{
    case "Console": builder.Services.AddSingleton<ISmsSender, ConsoleSmsSender>(); break;
    case "Eskiz":
        builder.Services.AddHttpClient<ISmsSender, EskizSmsSender>((sp, c) =>
        {
            var o = sp.GetRequiredService<IOptions<SmsOptions>>().Value;
            c.BaseAddress = new Uri(o.BaseUrl!);
            c.Timeout = TimeSpan.FromSeconds(o.TimeoutSeconds);
        }).AddStandardResilienceHandler();
        break;
    default: throw new InvalidOperationException($"Noma'lum Sms:Provider: {builder.Configuration["Sms:Provider"]}");
}

var ordersCs = builder.Configuration.GetConnectionString("Orders")
    ?? throw new InvalidOperationException("ConnectionStrings:Orders yo'q");
builder.Services.AddDbContext<OrdersDbContext>(o => o.UseNpgsql(ordersCs)
    .EnableSensitiveDataLogging(builder.Environment.IsDevelopment())); // faqat dev'da parametr qiymatlari

builder.Services.AddFeatureManagement();

// Readiness: qiymat bor, lekin ulanadimi?
builder.Services.AddHealthChecks()
    .AddNpgSql(ordersCs, name: "orders-db", tags: ["ready"]);

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    // Sirlarni yashirib konfiguratsiya manzarasini ko'rsatish: faqat dev
    app.MapGet("/debug/config", (IConfiguration cfg) => ((IConfigurationRoot)cfg).GetDebugView(ctx =>
        ctx.Key.Contains("Key", StringComparison.OrdinalIgnoreCase)
        || ctx.Key.Contains("Secret", StringComparison.OrdinalIgnoreCase)
        || ctx.Key.Contains("Password", StringComparison.OrdinalIgnoreCase)
        || ctx.Key.StartsWith("ConnectionStrings", StringComparison.OrdinalIgnoreCase)
            ? "***" : ctx.Value));
}

app.MapHealthChecks("/health/live");
app.MapHealthChecks("/health/ready", new() { Predicate = r => r.Tags.Contains("ready") });

app.MapPost("/api/orders", async (CreateOrderRequest req, IFeatureManager flags, IOrderService svc, CancellationToken ct) =>
    await flags.IsEnabledAsync("NewCheckout") ? await svc.CreateV2Async(req, ct) : await svc.CreateAsync(req, ct));

app.Run();
```

Nega shunday: `AddEnvironmentVariables()` ikkinchi marta chaqirilgan, chunki qo'shimcha JSON fayl ro'yxat oxiriga tushib env var'larni bosib qo'ygan bo'lardi; provider'ni ikki marta qo'shish arzon. `Sms:Provider` kaliti muhit nomi o'rniga ishlatilgan: staging'da ham real Eskiz'ni, dev'da ham sinab ko'rish mumkin. Sirlar faqat env var orqali, fayl va repo'da yo'q. `GetDebugView` maskalangan va faqat Development'da ochiq.

## 5. Tipik xatolar va anti-patternlar

### 5.1 Sirlar appsettings'da

```json
// ❌ appsettings.Development.json repo'da
{ "ConnectionStrings": { "Orders": "Host=prod-db;Password=Qwerty123" } }
```

Nega yomon: git tarixidan o'chirib bo'lmaydi; `git log -p` bilan har kim ko'radi; secret scanner'lar repo'ni bloklaydi.

```bash
# ✅ dev: user secrets; prod: Secret/Key Vault
dotnet user-secrets set "ConnectionStrings:Orders" "Host=localhost;..."
```

### 5.2 Muhit nomiga bog'liq biznes mantiq

```csharp
// ❌
if (env.EnvironmentName == "Prod") await sms.SendAsync(...); // "Production" yozilsa yubormaydi, hech kim bilmaydi
```

```csharp
// ✅ Konfiguratsiya kaliti, provider almashtirish
builder.Services.AddSingleton<ISmsSender>(sp => cfg["Sms:Provider"] == "Console" ? new ConsoleSmsSender() : ...);
```

### 5.3 Tarqoq `IConfiguration["..."]`

```csharp
// ❌ 14 joyda kalit; bittasida "Sms:Timeout" deb yozilgan, null, default 0, TaskCanceledException
var timeout = int.Parse(_config["Sms:TimeoutSeconds"]!);
```

```csharp
// ✅ Bitta options klassi, validatsiya, DI
public sealed class EskizSmsSender(IOptions<SmsOptions> o) { private readonly int _timeout = o.Value.TimeoutSeconds; }
```

### 5.4 Validatsiyasiz options

```csharp
// ❌ ApiKey bo'sh, ilova ko'tarildi, SMS 401, 20 daqiqa keyin bilindi
builder.Services.Configure<SmsOptions>(builder.Configuration.GetSection("Sms"));
```

```csharp
// ✅
builder.Services.AddOptions<SmsOptions>().BindConfiguration("Sms").ValidateDataAnnotations().ValidateOnStart();
```

### 5.5 Provider tartibini buzish

```csharp
// ❌ Key Vault env var'dan keyin: lokal override imkonsiz, testda haqiqiy Key Vault'ga urish
builder.Configuration.AddAzureKeyVault(...);
```

Nega yomon: keyin qo'shilgan g'olib; env var bilan testda qiymatni almashtirib bo'lmaydi.

```csharp
// ✅ Key Vault, keyin env var yana bir marta (yoki Key Vault faqat prod'da)
builder.Configuration.AddAzureKeyVault(...);
builder.Configuration.AddEnvironmentVariables();
```

### 5.6 Butun konfiguratsiyani log'ga yozish

```csharp
// ❌ Startup'da "debug uchun"
foreach (var kv in config.AsEnumerable()) logger.LogInformation("{Key}={Value}", kv.Key, kv.Value);
```

Nega yomon: parollar, API key'lar log agregatoriga (Elastic, Seq) ketadi va u yerda saqlanadi, kim ko'rishini nazorat qilib bo'lmaydi.

```csharp
// ✅ Maskalangan GetDebugView, faqat Development, faqat so'rov bo'yicha (4-bo'lim)
```

### 5.7 Feature flag'ni `IConfiguration` orqali qo'lda o'qish

```csharp
// ❌ bool.Parse(config["Features:NewCheckout"]) har joyda, foiz/targeting yo'q
```

```csharp
// ✅ IFeatureManager: filterlar, reload, bir xil semantika
```

## 6. Best practices checklist

- [ ] Sirlar hech qachon repo'da: dev'da user secrets, prod'da Secret manager/Key Vault/Kubernetes Secret; `.gitignore`da `*.local.json`.
- [ ] Har sozlama bo'limi uchun options klassi + `ValidateDataAnnotations` + `ValidateOnStart`: noto'g'ri deploy birinchi mijozdan oldin yiqiladi.
- [ ] `ASPNETCORE_ENVIRONMENT` qiymati standart nomlardan (`Development`/`Staging`/`Production`): `IsProduction()` ishlashi va fayl nomi mos kelishi uchun.
- [ ] `appsettings.{Env}.json`da faqat farqlar: default'lar bitta joyda, diff o'qiladigan bo'ladi.
- [ ] Env var ierarxiyasi `__` bilan; connection string `ConnectionStrings__Name`: boshqa belgilar shell'da ishlamaydi.
- [ ] Qo'shimcha provider (Key Vault, extra JSON) qo'shgandan keyin `AddEnvironmentVariables()` yana chaqiring: env var override saqlansin.
- [ ] Biznes mantiq muhit nomiga qaramasin; provider/xulq konfiguratsiya kaliti orqali tanlansin.
- [ ] Reload kerak bo'lgan sozlama `IOptionsMonitor` (singleton) yoki `IOptionsSnapshot` (scoped) orqali; container'da `DOTNET_USE_POLLING_FILE_WATCHER=1` va `subPath`siz mount.
- [ ] Feature flag'lar `Microsoft.FeatureManagement` orqali, har flag'ga o'chirish muddati: ikki yo'lli kod to'planmasin.
- [ ] `GetDebugView` maskalangan va faqat Development'da; `{@Options}` destructure log'da yo'q; `EnableSensitiveDataLogging` faqat dev.
- [ ] Readiness health check tashqi bog'liqliklarni (DB, Redis, to'lov API) tekshiradi: qiymat to'g'ri, lekin tarmoq yo'q holatini ko'rsatadi.
- [ ] `GetRequiredSection`/`?? throw` `Program.cs`da: yo'q bo'lim start'da aniqlanadi.
- [ ] Integratsion test `WebApplicationFactory` bilan Production konfiguratsiya shablonini (sirsiz) yuklab ko'radi: kalit nomi xatolari CI'da chiqadi.

## 7. Suhbat savollari va javoblar

**1. Bir xil kalit `appsettings.json` va env var'da bo'lsa, qaysi biri g'olib?**
Env var, chunki `CreateBuilder` uni JSON fayllardan keyin qo'shadi va `ConfigurationRoot` provider'larni oxiridan boshiga qidiradi. Tartib: `appsettings.json` → `appsettings.{Env}.json` → user secrets (dev) → env var → command line. Tuzoq: siz `builder.Configuration.AddJsonFile(...)` yoki Key Vault qo'shsangiz, u ro'yxat oxiriga tushadi va env var'ni bosib o'tadi; yechim `AddEnvironmentVariables()`ni qayta chaqirish.

**2. Connection string'ni qayerda saqlaysiz?**
Dev: `dotnet user-secrets` (fayl `~/.microsoft/usersecrets/`, repo'dan tashqarida). CI: pipeline secret'lari env var sifatida. Prod: Kubernetes Secret → env var `ConnectionStrings__Orders`, yoki Key Vault/Vault provider; ideal holda parolsiz (managed identity, IAM auth) ulanish. `appsettings.*.json`da hech qachon, hatto private repo'da ham. Tuzoq: `appsettings.Development.json` ham repo'da, u ham sir uchun joy emas.

**3. Konfiguratsiya o'zgarganda ilovani qayta ishga tushirmasdan qanday o'qiysiz?**
`AddJsonFile(..., reloadOnChange: true)` `FileSystemWatcher` ochadi, o'zgarishda provider qayta yuklanadi va `IChangeToken` ishga tushadi. Iste'molchi `IOptionsMonitor<T>.CurrentValue` (singleton'da) yoki `IOptionsSnapshot<T>` (scoped, keyingi so'rovda yangi) ishlatishi kerak; `IOptions<T>` hech qachon yangilanmaydi. Container'da watcher ishlamasligi mumkin: polling env var va symlink mount. Env var o'zgarishi esa faqat restart bilan ko'rinadi.

**4. Nega env var'da `Sms__ApiKey`, `Sms:ApiKey` emas?**
Konfiguratsiya ierarxiyasi `:` bilan ajratiladi, lekin bash va ko'p shell'lar `:` bo'lgan env var nomini qabul qilmaydi. `EnvironmentVariablesConfigurationProvider` `__`ni `:`ga almashtiradi. Qo'shimcha: `ASPNETCORE_` va `DOTNET_` prefiksli env var'lar host konfiguratsiyasi (environment, URL'lar) uchun, prefiks olib tashlanib o'qiladi.

**5. Options validatsiyasini nega startup'da qilasiz, birinchi so'rovda emas?**
Rolling deploy'da yangi pod ko'tariladi, readiness o'tadi, trafik keladi, keyin birinchi so'rov 500 beradi: mijoz xatoni ko'rdi, eski pod allaqachon o'chirilgan. `ValidateOnStart` bilan pod ishga tushishda yiqiladi, Kubernetes rollout'ni to'xtatadi, eski versiya trafikni ushlab turadi, mijoz hech narsa sezmaydi. Startup narxi bir necha millisekund.

**6. `IConfiguration` ichida ierarxiya qanday saqlanadi, massivlar qanday?**
Tekis kalit-qiymat: `Sms:Providers:0:Name`. JSON provider ichma-ich obyektlarni va massivlarni shu shaklga tekislaydi, massiv indeksi kalitning bir bo'lagi. Binder esa `List<T>`/`T[]`ni indekslardan yig'adi. Tuzoq: ikki provider massivni birlashtiradi, almashtirmaydi: `appsettings.json`da 3 element, env var'da `0` indeks berilsa, natija 3 element (0-chisi o'zgargan), 1 element emas. Massivni to'liq almashtirish uchun qolganlarini ham bo'sh qilib berish kerak.

**7. Feature flag va konfiguratsiya kaliti farqi nima, nega alohida kutubxona?**
Oddiy `bool` kalit yoqish/o'chirishni beradi, lekin foizlab chiqarish (`Percentage`), vaqt oynasi, foydalanuvchi/tenant targeting, flag nomlarini markazlashtirish, `[FeatureGate]` va reload semantikasi yo'q. `Microsoft.FeatureManagement` shularni standart beradi va Azure App Configuration bilan deploy'siz boshqariladi. Tuzoq: flag doimiy konfiguratsiya emas, u vaqtinchalik; o'chirish rejasiz flag'lar texnik qarz.

**8. Production'da `GetDebugView` chaqirilsa nima xavf?**
`GetDebugView` har kalitning qiymatini va qaysi provider'dan kelganini chiqaradi: parollar, API key'lar, connection string'lar. Ochiq endpoint orqali chiqsa, bu to'liq sir oqishi. .NET 7+ dagi overload maskalash imkonini beradi, lekin maskalash ro'yxati to'liq bo'lmasligi mumkin; shuning uchun faqat Development va faqat autentifikatsiyalangan diagnostika uchun.

**9. Kubernetes'da sirni env var va fayl sifatida berish farqi?**
Env var: oddiy, `__` nomlash, lekin pod restart'siz yangilanmaydi va `/proc/<pid>/environ` orqali container ichidagi har qanday jarayonga ko'rinadi, crash dump'ga tushadi. Fayl (volume): `AddKeyPerFile` bilan o'qiladi, Secret yangilansa fayl ham yangilanadi (symlink, `subPath`siz), `reloadOnChange` ishlaydi, fayl ruxsatlari bilan cheklash mumkin. Yuqori xavfsizlik talabida fayl yoki External Secrets + Vault CSI.

**10. Konfiguratsiya binding reflection ishlatadi; Native AOT'da nima bo'ladi?**
Reflection-based `Bind` AOT'da trimming tufayli property'larni topa olmasligi mumkin. .NET 8+ da `EnableConfigurationBindingGenerator` source generator `Bind`/`Get<T>`/`BindConfiguration` chaqiruvlarini kompilyatsiya vaqtida konkret kodga almashtiradi. AOT loyihada bu shart, oddiy loyihada startup'ni biroz tezlashtiradi.

## 8. Qo'shimcha manbalar

- Configuration in ASP.NET Core: https://learn.microsoft.com/aspnet/core/fundamentals/configuration/
- Configuration providers (.NET): https://learn.microsoft.com/dotnet/core/extensions/configuration-providers
- Environments: https://learn.microsoft.com/aspnet/core/fundamentals/environments
- Safe storage of app secrets in development: https://learn.microsoft.com/aspnet/core/security/app-secrets
- Azure Key Vault configuration provider: https://learn.microsoft.com/aspnet/core/security/key-vault-configuration
- Options pattern va validatsiya: https://learn.microsoft.com/dotnet/core/extensions/options
- Configuration binding source generator: https://learn.microsoft.com/dotnet/core/extensions/configuration-generator
- Microsoft.FeatureManagement: https://learn.microsoft.com/azure/azure-app-configuration/feature-management-dotnet-reference
- FeatureManagement GitHub: https://github.com/microsoft/FeatureManagement-Dotnet
- Kubernetes Secrets: https://kubernetes.io/docs/concepts/configuration/secret/
- HashiCorp Vault Agent Injector: https://developer.hashicorp.com/vault/docs/platform/k8s/injector
- Andrew Lock, konfiguratsiya va options haqida maqolalar: https://andrewlock.net/tag/configuration/
