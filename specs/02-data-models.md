# 02 - Domain Models & Data Entity Specification

```
+--------------------+            +------------------------+
|   ApplicationUser  | 1        * |       InviteToken      |
|--------------------|------------|------------------------|
| Id (PK)            |            | Id (PK)                |
| Username           |            | TokenHash              |
| PasswordHash       |            | CreatedByUserId (FK)   |
| Role (Admin/User)  |            | ExpiresUtc             |
| InstapaperToken    |            | ConsumedUtc            |
| InstapaperSecret   |            +------------------------+
+--------------------+
          | 1
          |
          | *
+--------------------+
|  FeedSubscription  |
|--------------------|
| Id (PK)            |
| UserId (FK)        |
| Title              |
| FeedUrl            |
| PollIntervalMins   |
| PreferFeedContent  |
| FallbackToUrl      |
| LastPolledUtc      |
| NextPollUtc        |
| HealthStatus       |
| LastErrorMessage   |
+--------------------+
          | 1
          |
          | *
+--------------------+
|    FeedArticle     |
|--------------------|
| Id (PK)            |
| FeedId (FK)        |
| ItemId (Hash/GUID) |
| ArticleUrl         |
| Title              |
| PublishedUtc       |
| SyncStatus         |
| DeliveryType       |
| RetryCount         |
| NextRetryUtc       |
| ErrorMessage       |
+--------------------+
```

## Entity Classes

### ApplicationUser
```csharp
public enum UserRole { Admin = 1, User = 2 }

public class ApplicationUser
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Username { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public UserRole Role { get; set; } = UserRole.User;
    
    // Encrypted via IDataProtector (Base64 ciphertext)
    public string? EncryptedInstapaperToken { get; set; }
    public string? EncryptedInstapaperTokenSecret { get; set; }
    
    public DateTime CreatedUtc { get; set; } = DateTime.UtcNow;

    public ICollection<FeedSubscription> Feeds { get; set; } = new List<FeedSubscription>();
    public ICollection<InviteToken> CreatedInvites { get; set; } = new List<InviteToken>();
}
```

### InviteToken
```csharp
public class InviteToken
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string TokenHash { get; set; } = string.Empty; // SHA256 of plain token string
    public Guid CreatedByUserId { get; set; }
    public ApplicationUser CreatedByUser { get; set; } = null!;
    public DateTime CreatedUtc { get; set; } = DateTime.UtcNow;
    public DateTime ExpiresUtc { get; set; }
    public DateTime? ConsumedUtc { get; set; }
    public Guid? ConsumedByUserId { get; set; }
}
```

### AppSettings
```csharp
public class AppSettings
{
    public int Id { get; set; } = 1; // Singleton row
    public bool IsProvisioned { get; set; } = false;
    public string EncryptedConsumerKey { get; set; } = string.Empty;
    public string EncryptedConsumerSecret { get; set; } = string.Empty;
    public DateTime UpdatedUtc { get; set; } = DateTime.UtcNow;
}
```

### FeedSubscription
```csharp
public enum FeedHealthStatus { Healthy = 1, Warning = 2, Error = 3 }

public class FeedSubscription
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public Guid UserId { get; set; }
    public ApplicationUser User { get; set; } = null!;
    
    public string Title { get; set; } = string.Empty;
    public string FeedUrl { get; set; } = string.Empty; // Private RSS URL
    
    public int PollIntervalMinutes { get; set; } = 60;
    public bool PreferFeedContent { get; set; } = false;
    public bool FallbackToUrlIfContentMissing { get; set; } = true;
    
    public DateTime? LastPolledUtc { get; set; }
    public DateTime NextPollUtc { get; set; } = DateTime.UtcNow;
    
    public FeedHealthStatus HealthStatus { get; set; } = FeedHealthStatus.Healthy;
    public string? LastErrorMessage { get; set; }
    public int ConsecutiveFailureCount { get; set; } = 0;

    public ICollection<FeedArticle> Articles { get; set; } = new List<FeedArticle>();
}
```

### FeedArticle
```csharp
public enum SyncStatus { SkippedBacklog = 1, Pending = 2, Pushed = 3, Failed = 4 }
public enum DeliveryMethod { None = 0, DirectContent = 1, UrlScrapeFallback = 2 }

public class FeedArticle
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public Guid FeedSubscriptionId { get; set; }
    public FeedSubscription FeedSubscription { get; set; } = null!;
    
    public string ArticleDeduplicationHash { get; set; } = string.Empty; // SHA256(FeedId + Guid/Url)
    public string ArticleUrl { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;
    public DateTime PublishedUtc { get; set; }
    
    public SyncStatus Status { get; set; } = SyncStatus.Pending;
    public DeliveryMethod DeliveryType { get; set; } = DeliveryMethod.None;
    
    public int RetryCount { get; set; } = 0;
    public DateTime? NextRetryUtc { get; set; }
    public string? LastErrorMessage { get; set; }
    public DateTime ProcessedUtc { get; set; } = DateTime.UtcNow;
}
```
