# 16. Microservices va integratsiya

> **Bir jumlada:** Servislar orasidagi tarmoq ishonchsiz: xabar yo'qoladi, ikki marta keladi, tartibsiz keladi, javob kechikadi; outbox, idempotent consumer, saga va resilience pattern'lari shu haqiqat bilan yashash usullari.
> **Nega Middle uchun:** Middle dasturchi ko'pincha "boshqa servisga xabar yuboradigan" kodni yozadi va aynan shu kod ishlab chiqarishda birinchi bo'lib sinadi: bazaga yozildi, broker'ga yetmadi; yoki xabar ikki marta kelib, mijozdan ikki marta pul yechildi. Pattern'larni bilmaslik bu xatolarni "tasodifiy" deb qoldiradi.
> **O'qish vaqti:** ~50 daqiqa

## Mundarija

- [1. Muammo va yechim](#1-muammo-va-yechim)
- [2. Qanday ishlaydi](#2-qanday-ishlaydi)
- [3. Asosiy tushunchalar](#3-asosiy-tushunchalar)
  - [3.1 Sync vs async integratsiya](#31-sync-vs-async-integratsiya)
  - [3.2 Broker tanlovi: RabbitMQ, Kafka, Azure Service Bus](#32-broker-tanlovi-rabbitmq-kafka-azure-service-bus)
  - [3.3 MassTransit](#33-masstransit)
  - [3.4 Outbox pattern](#34-outbox-pattern)
  - [3.5 Inbox va idempotent consumer](#35-inbox-va-idempotent-consumer)
  - [3.6 Saga: choreography vs orchestration](#36-saga-choreography-vs-orchestration)
  - [3.7 Resilience: retry, timeout, circuit breaker, bulkhead](#37-resilience-retry-timeout-circuit-breaker-bulkhead)
  - [3.8 gRPC contract-first](#38-grpc-contract-first)
  - [3.9 API Gateway: YARP](#39-api-gateway-yarp)
  - [3.10 Service discovery](#310-service-discovery)
  - [3.11 Event schema versioning](#311-event-schema-versioning)
  - [3.12 Distributed tracing bog'lanishi](#312-distributed-tracing-boglanishi)
- [4. Amaliy misol](#4-amaliy-misol)
- [5. Tipik xatolar va anti-patternlar](#5-tipik-xatolar-va-anti-patternlar)
- [6. Best practices checklist](#6-best-practices-checklist)
- [7. Suhbat savollari va javoblar](#7-suhbat-savollari-va-javoblar)
- [8. Qo'shimcha manbalar](#8-qoshimcha-manbalar)

## 1. Muammo va yechim

Buyurtma servisi `SaveChangesAsync` dan keyin RabbitMQ'ga `OrderSubmitted` yuboradi, Ombor servisi uni tinglab zaxira band qiladi. Juma kuni RabbitMQ 40 soniya qayta ishga tushdi. Shu vaqt ichida 37 ta buyurtma bazaga yozildi, lekin `Publish` exception bilan yiqildi; exception ushlanib log'ga yozildi, mijozga 200 qaytdi. Dushanba kuni ombor "bizda bu buyurtmalar yo'q" dedi. Aksincha holat ham bo'ldi: tarmoq uzilishida broker `ack` ni olmadi va bitta xabarni ikki marta yetkazdi; Ombor ikki marta band qildi, zaxira "minus"ga tushdi.

Ikkala holat bitta ildizdan: ikki mustaqil tizimga (baza va broker) yozish atomik emas. Buni **dual write** muammosi deyiladi. Yechim: faqat bitta tizimga (bazaga) atomik yozish, broker'ga esa bazadan alohida jarayon yuboradi (outbox). Broker tomonida esa "aynan bir marta" kafolati yo'q; bor-yo'g'i "kamida bir marta" (at-least-once). Shuning uchun consumer dublikatni tanib, ikkinchi marta hech narsa qilmasligi (idempotent) kerak.

Sinxron HTTP chaqiruvlarda esa muammo boshqacha: Payme API 8 soniya javob bermasa, bizning 200 ta so'rov thread pool'ni band qilib, butun servis javob bermay qoladi (cascading failure). Retry, timeout, circuit breaker va bulkhead shu holatlarni cheklaydi.

## 2. Qanday ishlaydi

**Dual write muammosi** diagrammada:

```mermaid
sequenceDiagram
    participant S as Orders servis
    participant DB as PostgreSQL
    participant B as RabbitMQ
    S->>DB: INSERT order; COMMIT
    S--xB: Publish(OrderSubmitted) (tarmoq xatosi)
    Note over S,B: Buyurtma bor, event yo'q. Teskari tartib ham xavfli:<br/>avval Publish, keyin COMMIT yiqilsa, event bor, buyurtma yo'q.
```

**Outbox** buni quyidagicha yechadi:

```mermaid
sequenceDiagram
    participant S as Orders servis
    participant DB as PostgreSQL
    participant W as Outbox worker
    participant B as RabbitMQ
    participant C as Warehouse consumer
    S->>DB: BEGIN; INSERT order; INSERT outbox_messages; COMMIT
    loop har 1 s
        W->>DB: SELECT ... FROM outbox_messages WHERE sent_at IS NULL FOR UPDATE SKIP LOCKED
        W->>B: Publish (MessageId = outbox.id)
        W->>DB: UPDATE outbox_messages SET sent_at = now()
    end
    B->>C: Deliver (ehtimol ikki marta)
    C->>DB: BEGIN; INSERT inbox(message_id) [dublikat bo'lsa: skip]; biznes; COMMIT
    C->>B: ack
```

Atomiklik faqat bazaga yoziladigan ikki qatorga (`orders` va `outbox_messages`) tayanadi; ularni bitta tranzaksiya kafolatlaydi. Worker yuborishdan keyin `UPDATE` qila olmasa (crash), keyingi aylanishda yana yuboradi: shu yerda at-least-once paydo bo'ladi. Consumer `MessageId` ni `inbox` jadvaliga yozadi; ikkinchi yetkazishda PK konflikti bo'ladi va consumer biznes ishni o'tkazib yuboradi, lekin broker'ga `ack` qaytaradi.

Broker darajasida `ack`: consumer xabarni qayta ishlab bo'lgach, broker'ga tasdiq yuboradi; tasdiq kelmasa (consumer crash) broker xabarni boshqa consumer'ga qayta beradi. MassTransit `Consume` metodi exception'siz tugaganda `ack`, exception'da retry siyosatiga qarab qayta urinish yoki `_error` navbatiga ko'chirish qiladi.

## 3. Asosiy tushunchalar

### 3.1 Sync vs async integratsiya

- **Ta'rif:** Sinxron (HTTP/gRPC): chaqiruvchi javobni kutadi, ikkala servis bir vaqtda tirik bo'lishi shart. Asinxron (broker): chaqiruvchi xabarni navbatga qo'yadi va davom etadi, qabul qiluvchi keyinroq qayta ishlaydi.
- **Nega kerak:** Sinxron chaqiruv oddiy va darhol javob beradi, lekin servislar availability'sini ko'paytiradi: A 99.9%, B 99.9% bo'lsa, A->B zanjiri 99.8%. Asinxron esa vaqtinchalik uzilishga chidamli, lekin "hozir nima holatda" degan savolga darhol javob bermaydi.
- **Qaror mezoni:**
  - Foydalanuvchi javobni hozir kutyaptimi (narx hisoblash, zaxira tekshirish)? Sync.
  - Natija keyinroq kelsa ham bo'ladimi (SMS, hisobot, ombor band qilish)? Async.
  - Bitta event'ni bir necha servis tinglaydimi? Async (publish/subscribe).
- **Tipik xato:** Async xabar yuborib, darhol boshqa servisdan natijani HTTP bilan so'rash ("yubordim, endi bormi?"). Bu ikkala modelning kamchiliklarini birlashtiradi.

### 3.2 Broker tanlovi: RabbitMQ, Kafka, Azure Service Bus

| Mezon | RabbitMQ | Apache Kafka | Azure Service Bus |
|---|---|---|---|
| Model | Navbat (queue) va exchange, xabar iste'mol qilingach o'chadi | Log (append-only), xabar retention muddatigacha saqlanadi, consumer offset bilan o'qiydi | Navbat va topic/subscription, boshqariladigan (PaaS) |
| Routing | Exchange turlari (direct, topic, fanout, headers), moslashuvchan | Topic va partition; routing kalit bo'yicha partition | Topic + subscription filter (SQL filter) |
| Tartib kafolati | Bitta navbat ichida, bitta consumer bo'lsa | Partition ichida qat'iy tartib | Session bo'lsa, session ichida |
| Qayta o'qish (replay) | Yo'q (iste'mol qilingan xabar yo'q) | Ha, offset'ni orqaga surib | Yo'q (dead-letter bundan mustasno) |
| Throughput | O'n minglab msg/s | Millionlab msg/s, batch | O'n minglab msg/s, tier'ga bog'liq |
| Competing consumers | Tabiiy (bir navbatdan ko'p consumer) | Partition soni bilan cheklangan (consumer soni <= partition soni) | Tabiiy |
| Operatsion narx | O'rtacha, o'zi hosting | Yuqori (ZooKeeper/KRaft, partition rejalashtirish), yoki boshqariladigan Confluent/MSK | Past (PaaS), Azure'ga bog'liq |
| Qachon | Command/event, ish navbati, moslashuvchan routing, kichik-o'rta hajm | Event stream, audit log, yuqori hajm, replay, bir necha mustaqil consumer guruhi | Azure ekotizimi, dead-letter, scheduled message, sessiya kerak bo'lganda |

Ko'p enterprise web tizimlar uchun RabbitMQ (yoki Azure'da Service Bus) yetarli. Kafka "event'larni tarixiy log sifatida saqlash va qayta o'qish" kerak bo'lganda yoki hajm yuz minglab msg/s bo'lganda oqlanadi.


### 3.3 MassTransit

- **Ta'rif:** .NET uchun messaging abstraksiyasi: consumer, retry, outbox, saga, request/response bir xil API bilan RabbitMQ, Azure Service Bus, Amazon SQS, Kafka (rider) ustida ishlaydi.
- **Nega kerak:** Broker SDK'si bilan to'g'ridan-to'g'ri ishlash serializatsiya, exchange/queue topologiyasi, retry, dead-letter, tracing'ni qo'lda yozishni talab qiladi. MassTransit buni standartlashtiradi.
- **Litsenziya:** MassTransit v9 (2025-yil e'loni) tijoriy litsenziyaga o'tdi; v8 ochiq kodli (Apache 2.0) bo'lib qoladi va ma'lum muddat qo'llab-quvvatlanadi. Loyiha boshida versiya va litsenziya qarorini yozib qo'ying. Muqobillar: Wolverine (JasperFx, ochiq kod), Rebus, broker SDK to'g'ridan-to'g'ri.
- **Kod:**

```csharp
// Program.cs
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderSubmittedConsumer>();

    // Outbox + inbox EF Core ustida (3.4, 3.5)
    x.AddEntityFrameworkOutbox<WarehouseDbContext>(o =>
    {
        o.UsePostgres();
        o.UseBusOutbox();                               // Publish bazaga yoziladi, worker yuboradi
        o.QueryDelay = TimeSpan.FromSeconds(1);
        o.DuplicateDetectionWindow = TimeSpan.FromMinutes(30);  // inbox: MessageId shu vaqt saqlanadi
    });

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(builder.Configuration["RabbitMq:Host"], "/", h =>
        {
            h.Username(builder.Configuration["RabbitMq:User"]!);
            h.Password(builder.Configuration["RabbitMq:Password"]!);
        });

        // Har consumer uchun navbat va exchange nomlari konvensiya bo'yicha yaratiladi
        cfg.ConfigureEndpoints(ctx);
    });

    // Har endpoint'ga inbox (dublikat filtri) va retry
    x.AddConfigureEndpointsCallback((context, name, cfg) =>
    {
        cfg.UseEntityFrameworkOutbox<WarehouseDbContext>(context);
        cfg.UseMessageRetry(r => r.Exponential(5,
            TimeSpan.FromSeconds(1), TimeSpan.FromMinutes(2), TimeSpan.FromSeconds(2)));
    });
});

// Consumer
public sealed class OrderSubmittedConsumer(WarehouseDbContext db) : IConsumer<OrderSubmitted>
{
    public async Task Consume(ConsumeContext<OrderSubmitted> ctx)
    {
        foreach (var item in ctx.Message.Items)
        {
            var stock = await db.Stocks.FirstAsync(s => s.ProductId == item.ProductId, ctx.CancellationToken);
            stock.Reserve(ctx.Message.OrderId, item.Quantity);
        }
        // Publish ham outbox orqali: biznes o'zgarish va event bitta tranzaksiyada
        await ctx.Publish(new StockReserved(ctx.Message.OrderId), ctx.CancellationToken);
        await db.SaveChangesAsync(ctx.CancellationToken);
    }
}
```

Ichida nima bo'ladi: `ConfigureEndpoints` consumer nomidan navbat nomini (`order-submitted`) yasaydi, `OrderSubmitted` tipi uchun fanout exchange yaratadi va navbatni unga bog'laydi. `Publish` xabarni tip nomi bo'yicha exchange'ga yuboradi; kim tinglashini publisher bilmaydi. Xabar JSON (System.Text.Json) bilan envelope ichida (`messageId`, `conversationId`, `headers`, `message`) ketadi.

- **Tipik xato:** Consumer ichida `ctx.Publish` ni `SaveChangesAsync` dan keyin, outbox'siz chaqirish: dual write qaytib keladi.

### 3.4 Outbox pattern

- **Ta'rif:** Chiquvchi xabarlar biznes ma'lumot bilan bitta tranzaksiyada `outbox` jadvaliga yoziladi; alohida worker jadvalni o'qib broker'ga yuboradi va yuborilganini belgilaydi.
- **Nega kerak:** Dual write muammosini yo'q qiladi: yoki ikkalasi (buyurtma va event) saqlanadi, yoki hech biri.
- **Kod (o'z implementatsiyasi, MassTransit'siz):**

```csharp
// Jadval
public sealed class OutboxMessage
{
    public Guid Id { get; init; } = Guid.CreateVersion7();   // MessageId sifatida ham ishlatiladi
    public string Type { get; init; } = null!;               // "OrderSubmitted"
    public string Payload { get; init; } = null!;            // JSON
    public DateTimeOffset CreatedAt { get; init; } = DateTimeOffset.UtcNow;
    public DateTimeOffset? SentAt { get; set; }
    public int Attempts { get; set; }
    public string? LastError { get; set; }
}

// DbContext: domain event'lar SaveChanges paytida outbox'ga aylanadi
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    var events = ChangeTracker.Entries<IHasDomainEvents>()
        .SelectMany(e => e.Entity.DomainEvents).ToList();
    foreach (var e in ChangeTracker.Entries<IHasDomainEvents>()) e.Entity.ClearDomainEvents();

    foreach (var e in events.OfType<IIntegrationEvent>())
        Outbox.Add(new OutboxMessage
        {
            Type = e.GetType().Name,
            Payload = JsonSerializer.Serialize(e, e.GetType(), JsonOptions)
        });

    return await base.SaveChangesAsync(ct);   // orders + outbox bitta INSERT tranzaksiyasida
}

// Worker: ko'p instansda xavfsiz, SKIP LOCKED tufayli ikki worker bir qatorni olmaydi
public sealed class OutboxProcessor(IServiceScopeFactory scopes, IPublishEndpoint publisher, ILogger<OutboxProcessor> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(1));
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            using var scope = scopes.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

            await using var tx = await db.Database.BeginTransactionAsync(stoppingToken);
            var batch = await db.Outbox
                .FromSql($"""
                    SELECT * FROM orders.outbox_messages
                    WHERE sent_at IS NULL AND attempts < 10
                    ORDER BY created_at
                    LIMIT 50
                    FOR UPDATE SKIP LOCKED
                    """)
                .ToListAsync(stoppingToken);

            foreach (var msg in batch)
            {
                try
                {
                    var type = MessageTypes.Resolve(msg.Type);      // nom -> Type lug'ati
                    var body = JsonSerializer.Deserialize(msg.Payload, type, JsonOptions)!;
                    await publisher.Publish(body, type, p => p.MessageId = msg.Id, stoppingToken);
                    msg.SentAt = DateTimeOffset.UtcNow;
                }
                catch (Exception ex) when (ex is not OperationCanceledException)
                {
                    msg.Attempts++;
                    msg.LastError = ex.Message;
                    log.LogWarning(ex, "Outbox {Id} yuborilmadi, urinish {Attempts}", msg.Id, msg.Attempts);
                }
            }

            await db.SaveChangesAsync(stoppingToken);
            await tx.CommitAsync(stoppingToken);
        }
    }
}
```

`FOR UPDATE SKIP LOCKED`: PostgreSQL qatorni tranzaksiya tugaguncha qulflaydi; boshqa worker qulflangan qatorlarni o'tkazib yuboradi. Bu ikki instans bitta xabarni bir vaqtda yubormasligini kafolatlaydi, lekin crash'dan keyin qayta yuborishni emas: `Publish` muvaffaqiyatli, `SaveChanges` yiqilsa, xabar yana yuboriladi. Shuning uchun `MessageId = msg.Id` beriladi, consumer shu Id bo'yicha dublikatni taniydi.

- **Tipik xato:** Worker'ni faqat bitta instansda ishlaydi deb hisoblash va `SKIP LOCKED` siz oddiy `SELECT`: ikki instans bir xabarni ikki marta yuboradi (consumer idempotent bo'lsa zarar yo'q, lekin yuk ikki barobar).

### 3.5 Inbox va idempotent consumer

- **Ta'rif:** At-least-once yetkazish: broker xabarni kamida bir marta yetkazadi, ba'zan ko'proq. Idempotent consumer: bitta xabarni necha marta olsa ham natija bir xil. Inbox: qayta ishlangan `MessageId` larni saqlovchi jadval; biznes o'zgarish va inbox yozuvi bitta tranzaksiyada.
- **Nega kerak:** Exactly-once yetkazish tarmoqda amalda mumkin emas (ack yo'qolsa broker bilmaydi, consumer qayta ishladimi yoki yo'q). Yagona ishonchli yo'l: dublikatni consumer tomonida zararsizlantirish.
- **Kod (qo'lda inbox):**

```csharp
// InboxMessage: (MessageId, Consumer) composite PK, ProcessedAt
public async Task Consume(ConsumeContext<OrderSubmitted> ctx)
{
    var messageId = ctx.MessageId ?? throw new InvalidOperationException("MessageId yo'q");

    await using var tx = await db.Database.BeginTransactionAsync(ctx.CancellationToken);

    // 1) Inbox'ga yozishga urinamiz. PK konflikti = dublikat.
    db.Inbox.Add(new InboxMessage { MessageId = messageId, Consumer = nameof(OrderSubmittedConsumer),
                                    ProcessedAt = DateTimeOffset.UtcNow });
    try
    {
        await db.SaveChangesAsync(ctx.CancellationToken);
    }
    catch (DbUpdateException ex) when (ex.InnerException is PostgresException { SqlState: "23505" })
    {
        return;   // allaqachon qayta ishlangan, ack qaytadi, biznes ishi takrorlanmaydi
    }

    // 2) Biznes ish, inbox yozuvi bilan bitta tranzaksiyada
    foreach (var item in ctx.Message.Items)
    {
        var stock = await db.Stocks.FirstAsync(s => s.ProductId == item.ProductId, ctx.CancellationToken);
        stock.Reserve(ctx.Message.OrderId, item.Quantity);
    }
    await db.SaveChangesAsync(ctx.CancellationToken);
    await tx.CommitAsync(ctx.CancellationToken);
}
```

Ikkinchi yo'l: biznes operatsiyaning o'zini idempotent qilish. `stock.Reserve(orderId, qty)` ichida "bu `orderId` uchun rezerv allaqachon bor" tekshiruvi bo'lsa, inbox jadvali kerak emas. Bu tabiiy kalit (`orderId`) bor joyda afzal; inbox esa universal.

- **Tipik xato:** Dublikatni tekshirish va biznes yozuvni turli tranzaksiyalarda qilish: ikki consumer bir vaqtda tekshiradi, ikkisi ham "yo'q" ko'radi, ikkisi ham yozadi.

### 3.6 Saga: choreography vs orchestration

- **Ta'rif:** Saga bir necha servisni qamrab oluvchi biznes jarayon (buyurtma: zaxira band qilish -> to'lov -> jo'natish), bunda tarqalgan tranzaksiya o'rniga har qadam lokal tranzaksiya, xato bo'lsa oldingi qadamlar kompensatsiya qilinadi (rezervni bo'shatish, to'lovni qaytarish).
- **Choreography:** Markaz yo'q, har servis event tinglab keyingi qadamni o'zi qiladi. Oddiy, 2-3 qadam uchun yaxshi; 6 qadamda "jarayon qayerda qoldi" degan savolga javob yo'q.
- **Orchestration:** Alohida state machine (orchestrator) har qadamga command yuboradi, javob kutadi, holatni saqlaydi. Jarayon ko'rinadigan, timeout va kompensatsiya bir joyda; narxi: qo'shimcha komponent va uning bazasi.
- **Kod (MassTransit state machine, orchestration):**

```csharp
public sealed class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }       // = OrderId
    public string CurrentState { get; set; } = null!;
    public Guid CustomerId { get; set; }
    public decimal Amount { get; set; }
    public DateTimeOffset SubmittedAt { get; set; }
}

public sealed class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public State AwaitingStock { get; private set; } = null!;
    public State AwaitingPayment { get; private set; } = null!;
    public State Completed { get; private set; } = null!;
    public State Failed { get; private set; } = null!;

    public Event<OrderSubmitted> OrderSubmitted { get; private set; } = null!;
    public Event<StockReserved> StockReserved { get; private set; } = null!;
    public Event<StockReservationFailed> StockFailed { get; private set; } = null!;
    public Event<PaymentCompleted> PaymentCompleted { get; private set; } = null!;
    public Event<PaymentFailed> PaymentFailed { get; private set; } = null!;

    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);

        // Hamma event OrderId bo'yicha bitta saga instansiga bog'lanadi
        Event(() => OrderSubmitted, e => e.CorrelateById(m => m.Message.OrderId));
        Event(() => StockReserved, e => e.CorrelateById(m => m.Message.OrderId));
        Event(() => StockFailed, e => e.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentCompleted, e => e.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentFailed, e => e.CorrelateById(m => m.Message.OrderId));

        Initially(
            When(OrderSubmitted)
                .Then(ctx =>
                {
                    ctx.Saga.CustomerId = ctx.Message.CustomerId;
                    ctx.Saga.Amount = ctx.Message.Amount;
                    ctx.Saga.SubmittedAt = DateTimeOffset.UtcNow;
                })
                .Publish(ctx => new ReserveStock(ctx.Saga.CorrelationId, ctx.Message.Items))
                .TransitionTo(AwaitingStock));

        During(AwaitingStock,
            When(StockReserved)
                .Publish(ctx => new ChargeCustomer(ctx.Saga.CorrelationId, ctx.Saga.CustomerId, ctx.Saga.Amount))
                .TransitionTo(AwaitingPayment),
            When(StockFailed)
                .Publish(ctx => new CancelOrder(ctx.Saga.CorrelationId, "Zaxira yetarli emas"))
                .TransitionTo(Failed)
                .Finalize());

        During(AwaitingPayment,
            When(PaymentCompleted)
                .Publish(ctx => new ConfirmOrder(ctx.Saga.CorrelationId))
                .TransitionTo(Completed)
                .Finalize(),
            When(PaymentFailed)
                // Kompensatsiya: rezervni bo'shatish, buyurtmani bekor qilish
                .Publish(ctx => new ReleaseStock(ctx.Saga.CorrelationId))
                .Publish(ctx => new CancelOrder(ctx.Saga.CorrelationId, "To'lov o'tmadi"))
                .TransitionTo(Failed)
                .Finalize());

        SetCompletedWhenFinalized();   // Finalize bo'lgan instans bazadan o'chiriladi
    }
}

// Ro'yxatdan o'tkazish
x.AddSagaStateMachine<OrderStateMachine, OrderState>()
 .EntityFrameworkRepository(r =>
 {
     r.ExistingDbContext<OrdersDbContext>();
     r.UsePostgres();      // saga qatorini FOR UPDATE bilan qulflaydi: bir instans bir vaqtda bir event
 });
```

Ichida: har event kelganda MassTransit `CorrelationId` bo'yicha `OrderState` qatorini pessimistik qulf bilan o'qiydi, `During` blokidagi mos `When` ni bajaradi, yangi holatni saqlaydi. Holatga mos kelmagan event (masalan `AwaitingPayment` da `StockReserved` yana kelsa) default'da e'tiborsiz qoldiriladi, bu dublikatlarga tabiiy himoya.

- **Tipik xato:** Kompensatsiya qadamini unutish. `PaymentFailed` da faqat `CancelOrder` yuborib, `ReleaseStock` yubormaslik: zaxira abadiy band bo'lib qoladi.

### 3.7 Resilience: retry, timeout, circuit breaker, bulkhead

- **Ta'rif:** `Microsoft.Extensions.Http.Resilience` (Polly v8 ustida) `HttpClient` pipeline'iga strategiyalar qo'shadi. Har biri boshqa muammoni yechadi:
  - **Timeout:** javob kelmasa so'rovni uzish; aks holda thread/connection cheksiz band. Ikki turi: attempt timeout (bitta urinish) va total request timeout (hamma urinishlar jami).
  - **Retry + jitter:** vaqtinchalik xatoda (5xx, 408, tarmoq) qayta urinish; eksponensial kutish (1s, 2s, 4s) va jitter (tasodifiy qo'shimcha), chunki 500 ta client bir vaqtda qayta ursa, "retry storm" bo'ladi.
  - **Circuit breaker:** xatolar nisbati chegaradan oshsa, ma'lum vaqt (break duration) so'rovlarni yubormasdan darhol xato qaytarish. Yiqilgan servisga bosim bermaydi va o'z thread'larimizni saqlaydi. Holatlari: Closed (normal) -> Open (hammasi rad) -> Half-open (bitta sinov so'rovi) -> Closed/Open.
  - **Bulkhead (concurrency limiter):** bitta tashqi servisga bir vaqtda N tadan ko'p so'rov yubormaslik; sekin servis butun ilovaning connection'larini yeb qo'ymaydi.
- **Kod:**

```csharp
builder.Services.AddHttpClient<PaymeClient>(c => c.BaseAddress = new Uri("https://checkout.paycom.uz/api"))
    .AddStandardResilienceHandler(o =>
    {
        // Standart pipeline tartibi: RateLimiter -> TotalTimeout -> Retry -> CircuitBreaker -> AttemptTimeout
        o.RateLimiter.DefaultRateLimiterOptions.PermitLimit = 50;   // bulkhead: bir vaqtda 50 ta so'rov
        o.RateLimiter.DefaultRateLimiterOptions.QueueLimit = 20;    // navbatda 20 ta, qolgani darhol xato

        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(20);
        o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(5);

        o.Retry.MaxRetryAttempts = 3;
        o.Retry.BackoffType = DelayBackoffType.Exponential;
        o.Retry.UseJitter = true;
        o.Retry.Delay = TimeSpan.FromMilliseconds(500);
        // Default: 5xx, 408, HttpRequestException, TimeoutRejectedException. POST uchun ehtiyot bo'ling (pastda)

        o.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);  // kamida 2 x AttemptTimeout bo'lishi shart
        o.CircuitBreaker.MinimumThroughput = 20;    // 30 s ichida kamida 20 so'rov bo'lsa baholanadi
        o.CircuitBreaker.FailureRatio = 0.5;        // 50% xato bo'lsa ochiladi
        o.CircuitBreaker.BreakDuration = TimeSpan.FromSeconds(15);
    });
```

Retry va idempotentlik: `POST /payments/charge` ni retry qilish ikki marta pul yechishi mumkin. Yechim: so'rovga `Idempotency-Key` header (buyurtma Id) qo'shish, qabul qiluvchi shu kalit bo'yicha dublikatni rad etadi. Agar tashqi API buni qo'llamasa, POST uchun retry'ni o'chiring: `o.Retry.ShouldHandle` ni faqat GET/PUT uchun sozlang yoki alohida `HttpClient` ro'yxatdan o'tkazing.

- **Tipik xato:** Retry'ni timeout'siz qo'shish: har urinish 100 s (HttpClient default) kutsa, 3 retry 5 daqiqa.

### 3.8 gRPC contract-first

- **Ta'rif:** gRPC HTTP/2 ustida binary (Protocol Buffers) protokol; kontrakt `.proto` faylda yoziladi, server va client kodi undan generatsiya qilinadi (contract-first).
- **Nega kerak:** Ichki servislar orasida JSON/REST'ga nisbatan 3-10 marta kichik payload, qat'iy tiplangan kontrakt, streaming. Tashqi (browser, hamkor) API uchun REST qoladi, chunki gRPC browser'da to'g'ridan-to'g'ri ishlamaydi (gRPC-Web kerak).
- **Kod:**

```protobuf
// Protos/stock.proto
syntax = "proto3";
option csharp_namespace = "Warehouse.Grpc";

service Stock {
  rpc Check (CheckRequest) returns (CheckReply);
}
message CheckRequest { string product_id = 1; int32 quantity = 2; }
message CheckReply   { bool available = 1; int32 on_hand = 2; }
```

```csharp
// Server (Warehouse)
public sealed class StockService(WarehouseDbContext db) : Stock.StockBase
{
    public override async Task<CheckReply> Check(CheckRequest req, ServerCallContext ctx)
    {
        var s = await db.Stocks.AsNoTracking()
            .FirstOrDefaultAsync(x => x.ProductId == Guid.Parse(req.ProductId), ctx.CancellationToken);
        if (s is null) throw new RpcException(new Status(StatusCode.NotFound, "product"));
        return new CheckReply { Available = s.Available >= req.Quantity, OnHand = s.Available };
    }
}
// Program.cs: builder.Services.AddGrpc(); app.MapGrpcService<StockService>();

// Client (Orders): AddGrpcClient<Stock.StockClient>(o => o.Address = new Uri("https://warehouse"))
// Chaqiruv: deadline shart, aks holda cheksiz kutadi
var reply = await client.CheckAsync(new CheckRequest { ProductId = id.ToString(), Quantity = 3 },
    deadline: DateTime.UtcNow.AddSeconds(3), cancellationToken: ct);
```

Kontrakt evolyutsiyasi: maydon raqamlari (`= 1`, `= 2`) hech qachon qayta ishlatilmaydi; yangi maydon qo'shish mos, o'chirish o'rniga `reserved 3;` deb belgilanadi.

- **Tipik xato:** `Guid` ni `bytes` ga emas, `string` ga yozib, keyin ikki tomonda har xil format (`N` vs `D`) ishlatish. Format kontraktda hujjatlanadi.

### 3.9 API Gateway: YARP

- **Ta'rif:** YARP (Yet Another Reverse Proxy) ASP.NET Core ichida ishlaydigan reverse proxy: tashqi so'rovni ichki servisga yo'naltiradi, auth, rate limit, header transformatsiyani bitta joyda qiladi.
- **Nega kerak:** Mijoz (frontend) 12 servisning manzilini bilmasligi kerak; JWT tekshiruvi va rate limit har servisda takrorlanmaydi.
- **Kod:**

```json
{
  "ReverseProxy": {
    "Routes": {
      "orders": {
        "ClusterId": "orders",
        "AuthorizationPolicy": "authenticated",
        "RateLimiterPolicy": "per-user",
        "Match": { "Path": "/api/orders/{**rest}" },
        "Transforms": [ { "PathRemovePrefix": "/api" } ]
      }
    },
    "Clusters": {
      "orders": {
        "LoadBalancingPolicy": "RoundRobin",
        "HealthCheck": { "Active": { "Enabled": true, "Interval": "00:00:10", "Path": "/health" } },
        "Destinations": {
          "a": { "Address": "http://orders-1:8080/" },
          "b": { "Address": "http://orders-2:8080/" }
        }
      }
    }
  }
}
```

```csharp
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));
builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("authenticated", p => p.RequireAuthenticatedUser());
builder.Services.AddRateLimiter(o => o.AddPolicy("per-user", ctx =>
    RateLimitPartition.GetTokenBucketLimiter(ctx.User.Identity?.Name ?? "anon",
        _ => new TokenBucketRateLimiterOptions { TokenLimit = 100, TokensPerPeriod = 100,
                                                  ReplenishmentPeriod = TimeSpan.FromMinutes(1) })));

var app = builder.Build();
app.UseAuthentication(); app.UseAuthorization(); app.UseRateLimiter();
app.MapReverseProxy();
```

- **Tipik xato:** Gateway'ga biznes mantiq (agregatsiya, transformatsiya) qo'shish; u yana "hamma narsa bitta joyda" monolitga aylanadi. Gateway faqat routing/auth/limit.

### 3.10 Service discovery

- **Ta'rif:** Servis manzilini kodga emas, konfiguratsiya yoki registr orqali topish. `Microsoft.Extensions.ServiceDiscovery` paketi `http://orders` kabi mantiqiy nomni konfiguratsiya (`Services:orders:http:0`), DNS yoki Kubernetes service orqali haqiqiy manzilga aylantiradi.
- **Nega kerak:** Instans soni va IP o'zgaradi; Kubernetes'da pod'lar qayta yaratiladi. Kodda `http://10.0.3.17:8080` bo'lishi mumkin emas.
- **Kod:**

```csharp
builder.Services.AddServiceDiscovery();
builder.Services.ConfigureHttpClientDefaults(http =>
{
    http.AddServiceDiscovery();           // "http://orders" nomini resolve qiladi
    http.AddStandardResilienceHandler();
});
builder.Services.AddHttpClient<OrdersApiClient>(c => c.BaseAddress = new Uri("http://orders"));
```

Konfiguratsiya: `"Services": { "orders": { "http": [ "orders-1:8080", "orders-2:8080" ] } }`. Aspire bu konfiguratsiyani AppHost'dan avtomatik beradi; Kubernetes'da `orders` DNS nomi service'ga to'g'ri keladi va alohida resolver kerak emas.

### 3.11 Event schema versioning

- **Ta'rif:** Event kontrakti vaqt bilan o'zgaradi; eski consumer yangi xabarni, yangi consumer eski xabarni (navbatda qolgan) o'qiy olishi kerak.
- **Qoidalar:**
  - Maydon qo'shish mumkin (optional, default qiymat bilan). Consumer "tolerant reader": tanimagan maydonni e'tiborsiz qoldiradi (System.Text.Json default shunday).
  - Maydon o'chirish, nomini o'zgartirish, tipini o'zgartirish mumkin emas. Kerak bo'lsa yangi maydon qo'shiladi, eski deprecated qoladi.
  - Semantika o'zgarsa (masalan `Amount` endi tiyinda), bu yangi event: `OrderSubmittedV2`, alohida tip/navbat. Publisher o'tish davrida ikkalasini chiqaradi, consumer'lar ko'chib bo'lgach eski o'chadi.
  - Xabar header'ida `schema-version` yoki tip nomida versiya; MassTransit tip nomini (`urn:message:Contracts:OrderSubmitted`) kontrakt sifatida ishlatadi, shuning uchun namespace o'zgarishi ham breaking change.
- **Kod:** `public sealed record OrderSubmitted(Guid OrderId, Guid CustomerId, IReadOnlyList<OrderItem> Items, decimal Amount, string? PromoCode = null);` Oxirgi maydon keyin qo'shilgan: eski publisher yubormasa `null`.

- **Tipik xato:** `record` ga `required` maydon qo'shish: navbatdagi eski xabarlar deserializatsiya bo'lmaydi, consumer `_error` navbatiga to'lib ketadi.

### 3.12 Distributed tracing bog'lanishi

- **Ta'rif:** Bitta foydalanuvchi harakati bir necha servis va broker orqali o'tganda, hamma span'lar bitta `trace-id` bilan bog'lanadi. HTTP'da `traceparent` header (W3C Trace Context), MassTransit'da xabar header'i orqali tarqaladi.
- **Nega kerak:** "Buyurtma 2 daqiqa kutdi" degan muammoda qaysi servis sekin ekanini ko'rish; outbox orqali o'tgan xabar ham asl HTTP so'rovga bog'lanadi.
- **Kod:** `builder.Services.AddOpenTelemetry().WithTracing(t => t.AddAspNetCoreInstrumentation().AddHttpClientInstrumentation().AddSource("MassTransit").AddOtlpExporter());` MassTransit o'z `ActivitySource` ini `"MassTransit"` nomi bilan e'lon qiladi.

MassTransit `Publish` paytida joriy `Activity` ning `traceparent` ini xabar header'iga yozadi; consumer tomonida yangi `Activity` shu parent bilan boshlanadi. Outbox worker alohida background ishda bo'lgani uchun MassTransit outbox'i header'larni jadvalda saqlab qo'yadi; o'z outbox'ingizni yozsangiz, `Activity.Current?.Id` ni `OutboxMessage` ga saqlab, yuborishda header'ga qo'ying.

## 4. Amaliy misol

Orders servisi buyurtmani saqlab, outbox orqali `OrderSubmitted` chiqaradi; Warehouse servisi uni inbox bilan idempotent qayta ishlaydi; Payme'ga HTTP chaqiruv resilience bilan. Hammasi MassTransit EF outbox ustida.

```csharp
// ===== Orders servis: Program.cs =====
builder.Services.AddDbContext<OrdersDbContext>(o => o.UseNpgsql(cs));

builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<OrdersDbContext>(o =>
    {
        o.UsePostgres();
        o.UseBusOutbox();                    // IPublishEndpoint.Publish -> OutboxMessage jadvali
    });
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(builder.Configuration["RabbitMq:Host"]);
        cfg.ConfigureEndpoints(ctx);
    });
});

// PaymeClient: 3.7 dagi AddStandardResilienceHandler konfiguratsiyasi (AttemptTimeout 5 s, jami 15 s, 2 retry)

// ===== Orders servis: command handler =====
public sealed class SubmitOrderHandler(OrdersDbContext db, IPublishEndpoint publisher)
    : ICommandHandler<SubmitOrderCommand, Result>
{
    public async Task<Result> Handle(SubmitOrderCommand cmd, CancellationToken ct)
    {
        var order = await db.Orders.Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == cmd.OrderId, ct);
        if (order is null) return Result.NotFound("order");

        try { order.Submit(); }
        catch (DomainException ex) { return Result.Invalid(ex.Message); }

        // UseBusOutbox tufayli bu Publish RabbitMQ'ga emas, OutboxMessage jadvaliga yozadi.
        // Quyidagi SaveChangesAsync orders + outbox ni bitta tranzaksiyada saqlaydi.
        await publisher.Publish(new OrderSubmitted(
            order.Id, order.CustomerId,
            order.Lines.Select(l => new OrderItem(l.ProductId, l.Quantity)).ToList(),
            order.Total.Amount), ct);

        await db.SaveChangesAsync(ct);
        return Result.Ok();
    }
}

// ===== Contracts (alohida NuGet/loyiha, ikkala servis reference beradi) =====
namespace Shop.Contracts.Orders;
public sealed record OrderItem(Guid ProductId, int Quantity);
public sealed record OrderSubmitted(Guid OrderId, Guid CustomerId, IReadOnlyList<OrderItem> Items, decimal Amount);
public sealed record StockReserved(Guid OrderId);
public sealed record StockReservationFailed(Guid OrderId, Guid ProductId, string Reason);

// ===== Warehouse servis: Program.cs =====
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderSubmittedConsumer>();
    x.AddEntityFrameworkOutbox<WarehouseDbContext>(o =>
    {
        o.UsePostgres();
        o.UseBusOutbox();
        o.DuplicateDetectionWindow = TimeSpan.FromHours(1);   // inbox: MessageId shu muddat eslab qolinadi
    });
    x.AddConfigureEndpointsCallback((context, name, cfg) =>
    {
        cfg.UseEntityFrameworkOutbox<WarehouseDbContext>(context);   // consumer inbox + outbox
        cfg.UseMessageRetry(r => r.Exponential(4, TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(30), TimeSpan.FromSeconds(2)));
    });
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(builder.Configuration["RabbitMq:Host"]);
        cfg.PrefetchCount = 16;             // bir instans bir vaqtda 16 xabar oladi
        cfg.ConfigureEndpoints(ctx);
    });
});

// ===== Warehouse servis: consumer =====
public sealed class OrderSubmittedConsumer(WarehouseDbContext db, ILogger<OrderSubmittedConsumer> log)
    : IConsumer<OrderSubmitted>
{
    public async Task Consume(ConsumeContext<OrderSubmitted> ctx)
    {
        var msg = ctx.Message;

        // Inbox MassTransit tomonidan: bu MessageId avval qayta ishlangan bo'lsa, Consume chaqirilmaydi.
        // Qo'shimcha biznes idempotentlik: Reserve ichida orderId bo'yicha mavjud rezerv tekshiriladi.
        foreach (var item in msg.Items)
        {
            var stock = await db.Stocks.FirstOrDefaultAsync(s => s.ProductId == item.ProductId, ctx.CancellationToken);
            if (stock is null || !stock.TryReserve(msg.OrderId, item.Quantity))
            {
                log.LogWarning("Zaxira yetarli emas: order {OrderId}, product {ProductId}", msg.OrderId, item.ProductId);
                await ctx.Publish(new StockReservationFailed(msg.OrderId, item.ProductId, "insufficient"), ctx.CancellationToken);
                await db.SaveChangesAsync(ctx.CancellationToken);   // outbox yozuvi saqlanadi, rezervlar emas (hech biri qilinmagan)
                return;
            }
        }

        await ctx.Publish(new StockReserved(msg.OrderId), ctx.CancellationToken);
        await db.SaveChangesAsync(ctx.CancellationToken);
        // Shu nuqtada: inbox yozuvi + rezervlar + outbox (StockReserved) bitta tranzaksiyada.
        // SaveChanges muvaffaqiyatli bo'lsa, MassTransit ack qaytaradi va outbox'ni yuboradi.
    }
}
```

Nima uchun shunday:

- `UseBusOutbox` command handler'dagi `Publish` ni bazaga yo'naltiradi; `SaveChangesAsync` siz hech narsa ketmaydi. Bu "yozildi, lekin event ketmadi" holatini mumkin qilmaydi.
- Consumer'da `UseEntityFrameworkOutbox` ikki ishni qiladi: inbox (dublikat `MessageId` ni `DuplicateDetectionWindow` ichida rad etadi) va consumer ichidagi `Publish` uchun outbox.
- `stock.TryReserve(orderId, qty)` biznes darajasida idempotent: aggregate ichida `orderId` bo'yicha mavjud rezerv bo'lsa, qayta band qilmaydi. Inbox oynasi (1 soat) tugagandan keyin kelgan dublikat ham shu tufayli zararsiz.
- `PrefetchCount = 16`: broker bir instansga bir vaqtda 16 ta xabar beradi; qiymat katta bo'lsa bitta instans hammasini olib, boshqalari bo'sh turadi; kichik bo'lsa round-trip ko'payadi.

## 5. Tipik xatolar va anti-patternlar

**1. Dual write**

```csharp
// ❌
await db.SaveChangesAsync(ct);
await bus.Publish(new OrderSubmitted(...), ct);   // bu yerda yiqilsa, event yo'q
```

Oqibati: tarmoq uzilishida buyurtma bor, ombor bilmaydi. Yoki teskari tartibda: event bor, buyurtma rollback.

```csharp
// ✅ Outbox: bitta tranzaksiya
await publisher.Publish(new OrderSubmitted(...), ct);   // UseBusOutbox -> jadval
await db.SaveChangesAsync(ct);
```

**2. Idempotent bo'lmagan consumer**

```csharp
// ❌
public async Task Consume(ConsumeContext<PaymentReceived> ctx)
{
    account.Balance += ctx.Message.Amount;   // ikki marta kelsa ikki marta qo'shiladi
    await db.SaveChangesAsync();
}
```

```csharp
// ✅ Tabiiy kalit bo'yicha
if (await db.Payments.AnyAsync(p => p.ExternalId == ctx.Message.PaymentId, ctx.CancellationToken)) return;
db.Payments.Add(new Payment(ctx.Message.PaymentId, ctx.Message.Amount));   // ExternalId UNIQUE index
account.Balance += ctx.Message.Amount;
await db.SaveChangesAsync(ctx.CancellationToken);
```

**3. Retry'ni timeout va circuit breaker'siz**

```csharp
// ❌ Har urinish 100 s default timeout, 5 retry: 8 daqiqa kutish, thread band
.AddResilienceHandler("p", b => b.AddRetry(new HttpRetryStrategyOptions { MaxRetryAttempts = 5 }));
```

Oqibati: tashqi servis yiqilganda bizning servis ham yiqiladi (cascading failure).

```csharp
// ✅ Standart handler: timeout + retry + breaker + limiter
.AddStandardResilienceHandler(o => { o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(5); });
```

**4. Sinxron zanjir (A -> B -> C -> D HTTP).** Availability'lar ko'paytiriladi, eng sekin servis hammaning latency'sini belgilaydi; faqat darhol javob kerak qismni sync, qolganini event bilan.

**5. Shared database (ikki servis bitta jadvalga).** Schema o'zgarishi ikkala servisni buzadi, bu tarqalgan monolit; har servis o'z bazasi/schema'si.

**6. Event'ga butun entity'ni qo'yish**

```csharp
// ❌
public record OrderChanged(Order Order);   // 40 maydon, EF navigation'lar, siklik havola
```

Oqibati: serializatsiya xatosi, kontrakt har EF o'zgarishida sinadi. Yechim: event'da faqat consumer'ga kerak maydonlar, tekis tiplar.

**7. `_error` navbatini kuzatmaslik.** Retry tugagach MassTransit xabarni `{queue}_error` ga ko'chiradi va jim qoladi; navbat uzunligiga alert va redelivery tartibi kerak.

## 6. Best practices checklist

- [ ] Har qanday "bazaga yozib, broker'ga yuborish" outbox orqali, chunki ikki tizimga atomik yozib bo'lmaydi.
- [ ] Har consumer idempotent (inbox yoki tabiiy kalit bo'yicha), chunki broker at-least-once yetkazadi.
- [ ] Dublikat tekshiruvi va biznes yozuv bitta tranzaksiyada, chunki alohida bo'lsa race condition dublikatni o'tkazib yuboradi.
- [ ] Har `HttpClient` da attempt timeout va total timeout bor, chunki default 100 s thread'larni band qiladi.
- [ ] Retry faqat idempotent so'rovlar uchun yoki `Idempotency-Key` bilan, chunki POST retry ikki marta pul yechishi mumkin.
- [ ] Retry'da eksponensial backoff va jitter, chunki bir vaqtda qayta urinish yiqilgan servisni yana yiqitadi.
- [ ] Event kontraktlari alohida `Contracts` loyihasida, faqat qo'shimcha o'zgarishlar, chunki navbatda eski xabarlar bor.
- [ ] Semantik o'zgarish yangi event tipi (`V2`), chunki eski consumer'lar bir vaqtda ko'chmaydi.
- [ ] Saga'da har qadam uchun kompensatsiya va timeout belgilangan, chunki "javob kelmadi" ham holat.
- [ ] Har servis o'z bazasi yoki schema'si, chunki shared jadval tarqalgan monolit.
- [ ] `_error`/dead-letter navbatlari monitoring va alert bilan, chunki retry tugagan xabar jimgina yo'qoladi.
- [ ] Broker va HTTP chaqiruvlari OpenTelemetry bilan bitta trace'ga bog'langan, chunki muammo qayerda ekanini boshqa yo'l bilan topib bo'lmaydi.

## 7. Suhbat savollari va javoblar

**1. Bazaga yozib, keyin broker'ga yuborsak nima xavf bor?**

Bu dual write: ikki mustaqil tizimga yozish atomik emas. `SaveChanges` o'tib, `Publish` yiqilsa, buyurtma bor, event yo'q; teskari tartibda `Publish` o'tib, commit yiqilsa, boshqa servis mavjud bo'lmagan buyurtma haqida xabar oladi. Yechim outbox: event bazadagi `outbox` jadvaliga biznes ma'lumot bilan bitta tranzaksiyada yoziladi, alohida worker uni broker'ga yuboradi va yuborilganini belgilaydi. Worker crash bo'lsa xabar qayta yuboriladi, shuning uchun consumer idempotent bo'lishi shart. Tuzoq: "`Publish` ni `try/catch` ga olib, xatoda log yozamiz" javobi muammoni yashiradi, yechmaydi.

**2. Exactly-once yetkazish mumkinmi?**

Tarmoq darajasida yo'q. Consumer xabarni qayta ishlab `ack` yuborganda tarmoq uzilsa, broker `ack` ni olmaydi va xabarni qayta yetkazadi; broker "consumer ishladimi yoki yo'q" ni bilolmaydi (ikki general muammosi). Amalda "exactly-once" deganda at-least-once yetkazish plus idempotent qayta ishlash tushuniladi: natija bir marta qo'llangandek bo'ladi. Kafka'ning "exactly-once semantics" ham faqat Kafka ichidagi (consume-transform-produce) oqimlar uchun, tashqi baza bilan emas. Tuzoq: at-most-once (ack'ni qayta ishlashdan oldin yuborish) ham bor, u xabar yo'qolishiga rozi bo'lgan holatlar (metrika) uchun.

**3. Circuit breaker nima va retry'dan farqi?**

Retry bitta so'rovning vaqtinchalik xatosini yashiradi: yana urinib ko'radi. Circuit breaker tizim darajasidagi muammoni taniydi: oxirgi N soniyada xatolar nisbati chegaradan oshsa, "ochiladi" va keyingi so'rovlarni tarmoqqa chiqarmasdan darhol rad etadi. Bu ikki narsani beradi: yiqilgan servisga bosim bermaydi (unga tiklanish uchun vaqt beradi) va bizning thread/connection'larimiz javobsiz so'rovlarda qotib qolmaydi. Break muddati o'tgach half-open holatda bitta sinov so'rovi o'tkaziladi; muvaffaqiyatli bo'lsa yopiladi. Tuzoq: retry breaker'dan oldin turadi (standart pipeline'da shunday), shuning uchun breaker ochiq bo'lganda retry ham darhol `BrokenCircuitException` oladi; bu to'g'ri xatti-harakat.

**4. Kafka va RabbitMQ qachon?**

RabbitMQ xabar navbati: xabar iste'mol qilingach yo'qoladi, routing moslashuvchan (exchange turlari), competing consumers tabiiy, ish navbati va command/event uchun qulay. Kafka tartiblangan log: xabarlar retention muddatigacha saqlanadi, consumer o'z offset'ini boshqaradi, replay mumkin, partition ichida qat'iy tartib, hajm millionlab msg/s. Kafka "event'lar tarixi kerak", "bir necha mustaqil tizim bir oqimni o'qiydi" (analitika + ombor + audit), yoki hajm katta bo'lganda. Oddiy enterprise CRUD tizimi uchun RabbitMQ (Azure'da Service Bus) kam operatsion narx bilan yetadi. Tuzoq: Kafka'da consumer soni partition sonidan oshmaydi; "consumer qo'shsam tezlashadi" ishlamaydi.

**5. Saga nima, choreography va orchestration farqi?**

Saga bir necha servisni qamrab olgan jarayonda tarqalgan tranzaksiya (2PC) o'rniga lokal tranzaksiyalar ketma-ketligi va xatoda kompensatsiya qadamlari. Choreography'da markaz yo'q: Orders `OrderSubmitted` chiqaradi, Warehouse tinglab `StockReserved` chiqaradi, Payments tinglab to'lov qiladi; har servis keyingi qadamni o'zi biladi. Oddiy, lekin 5-6 qadamda jarayon ko'rinmaydi va kompensatsiya tarqalib ketadi. Orchestration'da state machine (MassTransit saga) har qadamga command yuboradi, javob kutadi, holatni bazada saqlaydi, timeout va kompensatsiyani bir joyda boshqaradi. Narxi: qo'shimcha komponent. Tuzoq: orchestrator biznes qoidalarni o'ziga olib, "distributed monolith" bo'lib qolmasin; u faqat qadamlar tartibini biladi, qoidalar servislarda.

**6. At-least-once consumer'ni qanday idempotent qilasiz?**

Ikki yo'l. Birinchisi, inbox: `MessageId` ni alohida jadvalga biznes o'zgarish bilan bitta tranzaksiyada yozish; PK konflikti dublikat belgisi. MassTransit `UseEntityFrameworkOutbox` buni `DuplicateDetectionWindow` bilan avtomatik qiladi. Ikkinchisi, operatsiyani tabiiy kalit bo'yicha idempotent qilish: "orderId uchun rezerv bor bo'lsa, yana qilma", `ExternalId` ga UNIQUE index. Ikkinchisi oyna muddatiga bog'liq emas, shuning uchun ishonchliroq; ko'pincha ikkisi birga ishlatiladi. Tuzoq: tekshiruv va yozuvni turli tranzaksiyada qilish, ikki parallel consumer ikkisi ham "yo'q" ko'rib ikkisi ham yozadi.

**7. Event kontraktini qanday versiyalaysiz?**

Faqat qo'shimcha (additive) o'zgarish: optional maydon qo'shish, default qiymat bilan. O'chirish, nom o'zgartirish, tip o'zgartirish taqiq, chunki navbatda eski xabarlar bor va consumer'lar bir vaqtda deploy bo'lmaydi. Consumer tolerant reader: tanimagan maydonni e'tiborsiz qoldiradi. Semantika o'zgarsa (valyuta birligi), yangi tip `V2`, publisher o'tish davrida ikkalasini chiqaradi. MassTransit tip nomini (namespace bilan) kontrakt sifatida ishlatadi, shuning uchun `record` ni boshqa namespace'ga ko'chirish ham breaking. Tuzoq: `required` yoki non-nullable maydon qo'shish; eski xabar deserializatsiyada yiqiladi.

**8. gRPC'ni qachon, REST'ni qachon?**

gRPC ichki servis-servis chaqiruvlar uchun: binary payload kichik, HTTP/2 multiplexing, contract-first (`.proto` dan kod generatsiya), streaming, deadline o'rnatilgan. REST/JSON tashqi mijozlar (browser, mobil, hamkor) uchun: hamma tushunadi, debug oson, browser to'g'ridan-to'g'ri ishlaydi. gRPC browser'da gRPC-Web proxy talab qiladi. Tuzoq: gRPC "tezroq" degani uchun tashqi API'ni ham gRPC qilish, keyin hamkorlar integratsiya qila olmaydi.

**9. Outbox worker ikki instansda ishlasa nima bo'ladi?**

`FOR UPDATE SKIP LOCKED` siz ikkala worker bir xil qatorlarni o'qib, bir xabarni ikki marta yuboradi. `SKIP LOCKED` bilan PostgreSQL qulflangan qatorlarni ikkinchi worker'ga ko'rsatmaydi, har biri o'z batch'ini oladi. Lekin crash holati baribir qoladi: `Publish` o'tib, `sent_at` yozilmasa, keyingi aylanishda qayta yuboriladi. Shuning uchun outbox dublikatni to'liq yo'q qilmaydi, faqat kamaytiradi; consumer idempotentligi shart. MassTransit'ning EF outbox'i buni `OutboxState` jadvali bilan qiladi, lekin tamoyil bir xil. Tuzoq: worker tranzaksiyasini uzoq ushlab turish (50 ta xabarni ketma-ket 100 ms dan yuborish) qatorlarni 5 s qulflaydi; batch'ni kichik qiling.

## 8. Qo'shimcha manbalar

- Microsoft Learn, Build resilient HTTP apps (Microsoft.Extensions.Http.Resilience): https://learn.microsoft.com/dotnet/core/resilience/http-resilience
- Microsoft Learn, Service discovery in .NET: https://learn.microsoft.com/dotnet/core/extensions/service-discovery
- Microsoft Learn, gRPC on .NET: https://learn.microsoft.com/aspnet/core/grpc/
- Microsoft Learn, YARP reverse proxy: https://learn.microsoft.com/aspnet/core/fundamentals/servers/yarp/yarp-overview
- Microsoft Learn, Saga pattern: https://learn.microsoft.com/azure/architecture/reference-architectures/saga/saga
- Microsoft Learn, Transactional Outbox pattern: https://learn.microsoft.com/azure/architecture/best-practices/transactional-outbox-cosmos
- MassTransit docs, Transactional Outbox: https://masstransit.io/documentation/patterns/transactional-outbox
- MassTransit docs, Saga state machine: https://masstransit.io/documentation/patterns/saga/state-machine
- Polly v8 docs: https://www.pollydocs.org/
- Chris Richardson, microservices.io, Idempotent Consumer: https://microservices.io/patterns/communication-style/idempotent-consumer.html
- W3C Trace Context: https://www.w3.org/TR/trace-context/
