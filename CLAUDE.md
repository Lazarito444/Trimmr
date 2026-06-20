\# CLAUDE.md — Trimmr



\## Project overview



Appointment management SaaS for LATAM barbershops. Barbers create a digital space for their shop; clients discover nearby barbershops, book slots, and leave reviews.



\*\*Target market:\*\* Latin America  

\*\*Business model:\*\* SaaS — monthly subscription per barbershop (Free / Pro / Business)



\---



\## Architecture



Clean Architecture — dependency rule is strict: Domain ← Application ← Infrastructure ← Api.



```

BarberApp.sln

├── src/

│   ├── BarberApp.Domain/          # Entities, enums, value objects. Zero external deps.

│   ├── BarberApp.Application/     # CQRS handlers, interfaces, DTOs

│   ├── BarberApp.Infrastructure/  # EF Core, Npgsql, FCM, S3, Quartz.NET

│   └── BarberApp.Api/             # Endpoints, middleware, Program.cs

└── tests/

&#x20;   ├── BarberApp.Domain.Tests/

&#x20;   └── BarberApp.Application.Tests/

```



\---



\## Tech stack



| Layer | Technology |

|---|---|

| API | .NET 10, ASP.NET Core Minimal APIs, FluentValidation |

| ORM | EF Core 10 + Npgsql + NetTopologySuite (PostGIS) |

| Database | PostgreSQL + PostGIS |

| Auth | ASP.NET Identity + JWT Bearer |

| Background jobs | Quartz.NET (PostgreSQL store) |

| Push notifications | Firebase Cloud Messaging (FirebaseAdmin SDK) |

| File storage | AWS S3 or Azure Blob — abstracted behind IStorageService |

| Mobile apps | Flutter (two apps: client app + barber/owner app) |



\---



\## Key domain rules



\- \*\*4-hour rule:\*\* cancellations and rescheduling require at least 4 hours before the slot. Enforced in domain entity methods, not in handlers or controllers.

\- \*\*12-hour reminder:\*\* push notification sent to client 12h before appointment. Quartz job runs hourly, queries confirmed appointments in the 11–12h window, sets `reminder\_sent = true` after sending.

\- \*\*Max 20 photos per barbershop.\*\* Validated in application layer before calling `IStorageService`.

\- \*\*Rescheduling flow:\*\* barber/owner proposes new slot → client receives push → client accepts or cancels. New slot must also be 4h+ from now.

\- \*\*Reviews:\*\* triggered automatically when appointment is marked `Completed`. One review per appointment (1:1).



\---



\## Coding conventions



\### General

\- Language: C# 14, nullable enabled, implicit usings on

\- Prefer `record` for DTOs and value objects

\- Prefer `Result<T>` over exceptions for expected business failures

\- Exceptions are for truly exceptional cases (IO, DB down, etc.)

\- No business logic in controllers or handlers — handlers coordinate, domain decides



\### Naming

\- Commands: `VerbNounCommand` (e.g. `CancelAppointmentCommand`)

\- Queries: `GetNounByXQuery` (e.g. `GetAvailableSlotsQuery`)

\- Handlers: same name + `Handler` suffix

\- EF configurations: `NounConfiguration` in `Infrastructure/Persistence/Configurations/`



\### Enums

All enums are stored as `SMALLINT` in PostgreSQL. Always add `.HasConversion<short>()` in the entity configuration. Never rely on EF inferring this from the enum base type.



```csharp

public enum UserRole : short

{

&#x20;   Owner    = 0,

&#x20;   Barber   = 1,

&#x20;   Customer = 2

}



public enum AppointmentStatus : short

{

&#x20;   Pending      = 0,

&#x20;   Confirmed    = 1,

&#x20;   Completed    = 2,

&#x20;   Cancelled    = 3,

&#x20;   Rescheduled  = 4

}



public enum CancelledBy : short

{

&#x20;   Client = 0,

&#x20;   Barber = 1,

&#x20;   Owner  = 2

}



public enum DayOfWeek : short

{

&#x20;   Monday    = 0,

&#x20;   Tuesday   = 1,

&#x20;   Wednesday = 2,

&#x20;   Thursday  = 3,

&#x20;   Friday    = 4,

&#x20;   Saturday  = 5,

&#x20;   Sunday    = 6

}



public enum PlanTier : short

{

&#x20;   Free     = 0,

&#x20;   Pro      = 1,

&#x20;   Business = 2

}



public enum SubscriptionStatus : short

{

&#x20;   Active    = 0,

&#x20;   PastDue   = 1,

&#x20;   Cancelled = 2,

&#x20;   Trialing  = 3

}

```



```csharp

// Always explicit in entity configuration — never rely on EF inference

builder.Property(a => a.Status).HasConversion<short>();

```



\### Result pattern

```csharp

// Domain methods return Result, not void — never throw for business rule violations

public Result Cancel(CancelledBy cancelledBy, string? reason = null)

{

&#x20;   if (StartsAt - DateTimeOffset.UtcNow < BusinessRules.MinimumLeadTime)

&#x20;       return Result.Failure("Cannot cancel an appointment with less than 4 hours notice.");



&#x20;   if (Status is AppointmentStatus.Cancelled or AppointmentStatus.Completed)

&#x20;       return Result.Failure($"Cannot cancel an appointment with status {Status}.");



&#x20;   Status = AppointmentStatus.Cancelled;

&#x20;   CancelledBy = cancelledBy;

&#x20;   CancelledAt = DateTimeOffset.UtcNow;

&#x20;   CancellationReason = reason;

&#x20;   UpdatedAt = DateTimeOffset.UtcNow;

&#x20;   return Result.Success();

}

```



\### Value objects over primitive obsession

\- Coordinates → `Coordinates(double Latitude, double Longitude)` — never expose NTS `Point` in Domain

\- NTS `Point` lives only in `Infrastructure/Persistence/Configurations/`



```csharp

// Domain/ValueObjects/Coordinates.cs — no external dependencies

public record Coordinates(double Latitude, double Longitude)

{

&#x20;   public static Coordinates From(double lat, double lng)

&#x20;   {

&#x20;       if (lat is < -90 or > 90)   throw new ArgumentException("Invalid latitude.");

&#x20;       if (lng is < -180 or > 180) throw new ArgumentException("Invalid longitude.");

&#x20;       return new(lat, lng);

&#x20;   }

}



// Infrastructure only — never leak NTS into Domain or Application

builder.Property(b => b.Location)

&#x20;   .HasConversion(

&#x20;       v => new Point(v.Longitude, v.Latitude) { SRID = 4326 },

&#x20;       p => Coordinates.From(p.Y, p.X)

&#x20;   )

&#x20;   .HasColumnType("geography(Point, 4326)");

```



\### Rich domain models

Entities own their behavior. If logic belongs to a single entity, it's a method on that entity — not a service.



```

✓ appointment.Cancel(...)        — belongs to Appointment

✓ appointment.Reschedule(...)    — belongs to Appointment

✗ AppointmentService.Cancel(...) — wrong layer for this logic

```



Handlers coordinate across entities, services, and infrastructure — they don't contain business rules.



```csharp

// Application/Features/Appointments/Commands/CancelAppointmentCommandHandler.cs

public async Task<Result> Handle(CancelAppointmentCommand request, CancellationToken ct)

{

&#x20;   var appointment = await db.Appointments.FindAsync(\[request.AppointmentId], ct)

&#x20;       ?? return Result.Failure("Appointment not found.");



&#x20;   var cancelledBy = currentUser.Role switch

&#x20;   {

&#x20;       UserRole.Customer => CancelledBy.Client,

&#x20;       UserRole.Barber   => CancelledBy.Barber,

&#x20;       UserRole.Owner    => CancelledBy.Owner,

&#x20;       \_                 => return Result.Failure("Unauthorized.")

&#x20;   };



&#x20;   var result = appointment.Cancel(cancelledBy, request.Reason);

&#x20;   if (!result.IsSuccess) return result;



&#x20;   await db.SaveChangesAsync(ct);

&#x20;   await notifications.SendPushAsync(appointment.ClientId, "Appointment cancelled", "...");

&#x20;   return Result.Success();

}

```



\---



\## Domain entities (summary)



| Entity | Key fields |

|---|---|

| `User` | Id, Email, Phone, Name, Role, FcmToken, PasswordHash |

| `Barbershop` | Id, OwnerId, Name, Description, Address, Location (Coordinates), Active |

| `Subscription` | Id, BarbershopId, Plan, Status, StripeCustomerId, StripeSubscriptionId |

| `BarbershopBarber` | Id, BarbershopId, UserId, DisplayName, Active |

| `WorkingHours` | Id, BarbershopId, Day, OpenTime, CloseTime |

| `BarberBlock` | Id, BarbershopId, BarberId, Start, End, Reason |

| `Service` | Id, BarbershopId, Name, DurationMinutes, Price |

| `Appointment` | Id, BarbershopId, ClientId, BarberId, ServiceId, StartsAt, DurationMinutes, PriceSnapshot, Status, ReminderSent, CancelledBy, RescheduledSlot |

| `Review` | Id, AppointmentId, BarbershopId, BarberId, ClientId, Rating (1–5), Comment |

| `BarbershopPhoto` | Id, BarbershopId, StorageKey, OrderIndex, UploadedBy |

| `BarbershopRatingCache` | BarbershopId (PK), TotalReviews, AverageRating |



\---



\## Database



\- PostgreSQL 16 + PostGIS

\- Multi-tenancy: row-level — every table has `barbershop\_id` as tenant discriminator

\- All PKs: `UUID` (`uuid\_generate\_v4()`)

\- All timestamps: `TIMESTAMPTZ` in UTC

\- Soft delete: `deleted\_at TIMESTAMPTZ` — never hard delete barbershops or users

\- Enums: `SMALLINT`

\- Geolocation: `geography(Point, 4326)` with GIST index on `barbershops.location`

\- `barbershop\_rating\_cache`: updated via PostgreSQL trigger on `reviews` — never update manually from application code



\### Proximity query pattern

```csharp

// EF.Functions.IsWithinDistance → Npgsql translates to ST\_DWithin

var nearby = await db.Barbershops

&#x20;   .Where(b => b.Active \&\& b.DeletedAt == null)

&#x20;   .Where(b => EF.Functions.IsWithinDistance(b.Location, userPoint, radiusMeters))

&#x20;   .OrderBy(b => EF.Functions.Distance(b.Location, userPoint))

&#x20;   .ToListAsync(ct);

```



\### Appointment slot availability

Available slots = barbershop working hours − barber blocks − existing confirmed/pending appointments.  

This logic lives in `GetAvailableSlotsQueryHandler` — do not replicate it elsewhere.



\---



\## Background jobs (Quartz.NET)



\- All jobs implement `IJob` and are decorated with `\[DisallowConcurrentExecution]`

\- PostgreSQL job store — never use in-memory store in any environment

\- Job registration lives in `Infrastructure/DependencyInjection.cs`



```csharp

// Reminder job: query window is 11–12h ahead to avoid duplicate sends

// ReminderSent flag prevents re-sending if job overlaps

\[DisallowConcurrentExecution]

public class SendAppointmentRemindersJob(IAppDbContext db, INotificationService notifications) : IJob

{

&#x20;   public async Task Execute(IJobExecutionContext context)

&#x20;   {

&#x20;       var now          = DateTimeOffset.UtcNow;

&#x20;       var windowStart  = now.AddHours(11);

&#x20;       var windowEnd    = now.AddHours(12);



&#x20;       var appointments = await db.Appointments

&#x20;           .Where(a => a.Status == AppointmentStatus.Confirmed

&#x20;                    \&\& !a.ReminderSent

&#x20;                    \&\& a.StartsAt >= windowStart

&#x20;                    \&\& a.StartsAt <= windowEnd)

&#x20;           .ToListAsync();



&#x20;       foreach (var appointment in appointments)

&#x20;       {

&#x20;           await notifications.SendPushAsync(

&#x20;               appointment.ClientId,

&#x20;               "Appointment reminder",

&#x20;               $"You have an appointment tomorrow at {appointment.StartsAt:HH:mm}."

&#x20;           );

&#x20;           appointment.MarkReminderSent();

&#x20;       }



&#x20;       await db.SaveChangesAsync();

&#x20;   }

}

```



\---



\## Auth \& authorization



\- Two JWT claims always present: `userId` and `role` (0=Owner, 1=Barber, 2=Customer)

\- `TenantResolutionMiddleware` extracts `barbershopId` from token and sets it on `ICurrentUserService`

\- Authorization policies:

&#x20; - `OwnerOnly` — barbershop profile edits, photo management, barber invites/removals

&#x20; - `BarberOrOwner` — mark appointment completed, propose reschedule

&#x20; - Authenticated (any role) — book appointment, cancel own appointment, submit review



\---



\## API conventions



\- Versioning: URL prefix `/api/v1/`

\- Error responses follow RFC 9110 Problem Details (`application/problem+json`)

\- Business rule failures → HTTP 422 Unprocessable Entity

\- Not found → HTTP 404 with problem details

\- Pagination: cursor-based, `?cursor=\&limit=` params

\- Geo list endpoints include `distance\_m` field when `lat`/`lng` are provided in the request



\### Endpoint map

```

Auth

&#x20; POST  /api/v1/auth/register/customer

&#x20; POST  /api/v1/auth/register/barber

&#x20; POST  /api/v1/auth/login

&#x20; POST  /api/v1/auth/refresh



Barbershops

&#x20; GET   /api/v1/barbershops/nearby?lat\&lng\&radius\&minRating\&maxPrice\&availableToday

&#x20; GET   /api/v1/barbershops/search?q\&...filters

&#x20; GET   /api/v1/barbershops/{id}

&#x20; POST  /api/v1/barbershops                              \[OwnerOnly]

&#x20; PUT   /api/v1/barbershops/{id}                         \[OwnerOnly]

&#x20; POST  /api/v1/barbershops/{id}/photos                  \[OwnerOnly]

&#x20; DELETE /api/v1/barbershops/{id}/photos/{photoId}       \[OwnerOnly]



Barbers

&#x20; GET   /api/v1/barbershops/{id}/barbers

&#x20; POST  /api/v1/barbershops/{id}/barbers/invite          \[OwnerOnly]

&#x20; DELETE /api/v1/barbershops/{id}/barbers/{barberId}     \[OwnerOnly]

&#x20; GET   /api/v1/barbershops/{id}/barbers/{barberId}/slots?date



Appointments

&#x20; GET   /api/v1/appointments                             \[Customer → own | Barber → own]

&#x20; POST  /api/v1/appointments                             \[Customer]

&#x20; GET   /api/v1/appointments/{id}

&#x20; POST  /api/v1/appointments/{id}/cancel                 \[Customer | Barber | Owner]

&#x20; POST  /api/v1/appointments/{id}/reschedule             \[BarberOrOwner]

&#x20; POST  /api/v1/appointments/{id}/accept-reschedule      \[Customer]

&#x20; POST  /api/v1/appointments/{id}/complete               \[BarberOrOwner]



Reviews

&#x20; GET   /api/v1/barbershops/{id}/reviews

&#x20; POST  /api/v1/appointments/{id}/review                 \[Customer — only if Completed]



Subscriptions

&#x20; POST  /api/v1/webhooks/stripe                          \[Stripe → updates plan/status]

```



\---



\## Flutter apps



Two separate apps, one shared Dart package for models and API client:



```

/mobile/

├── trimr\_client/    # Customer app

├── trimr\_barber/    # Owner + Barber app (role-conditional UX after JWT decode)

└── trimr\_core/      # Shared: API client, models, theme, FCM setup

```



\- FCM token registered on login, refreshed via `FirebaseMessaging.onTokenRefresh`

\- Role-based UX is handled client-side after decoding the JWT — same API for both apps

\- Deep link on reschedule push notification opens the appointment detail screen



\---



\## Environment variables (required)



```bash

\# Database

DATABASE\_URL=postgresql://user:password@host:5432/barberapp



\# JWT

JWT\_SECRET=

JWT\_ISSUER=

JWT\_AUDIENCE=

JWT\_EXPIRY\_MINUTES=60



\# Firebase

FIREBASE\_CREDENTIALS\_JSON=



\# Storage

AWS\_ACCESS\_KEY\_ID=

AWS\_SECRET\_ACCESS\_KEY=

AWS\_S3\_BUCKET=

AWS\_REGION=



\# Stripe

STRIPE\_SECRET\_KEY=

STRIPE\_WEBHOOK\_SECRET=

```



\---



\## What is NOT in MVP



\- Online payments (Stripe charges) — webhook infrastructure exists for subscription management only

\- WhatsApp / SMS notifications — push only

\- Customer web app — Flutter mobile only

\- Loyalty / points system

\- Paid advertising between barbershops

\- Owner override of the 4-hour cancellation rule (planned for v2)

