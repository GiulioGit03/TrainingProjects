# Guida Completa: API REST con Repository Pattern, Caching e FluentValidation

## Indice
1. [Introduzione](#introduzione)
2. [Architettura del Progetto](#architettura-del-progetto)
3. [Setup Iniziale](#setup-iniziale)
4. [Implementazione Passo-Passo](#implementazione-passo-passo)
5. [Testing](#testing)
6. [Best Practices](#best-practices)

---

## Introduzione

Questa guida ti accompagna nella creazione di un'API REST completa utilizzando:
- **Repository Pattern** per separazione delle responsabilità
- **Three-Layer Architecture** (Presentation, Business Logic, Data Access)
- **In-Memory Caching** per performance ottimizzate
- **FluentValidation** per validazione robusta dei dati
- **Entity Framework Core** per accesso ai dati

---

## Architettura del Progetto

### Struttura delle Cartelle

```
SampleApi/
│
├── 📁 Controllers/              # PRESENTATION LAYER
│   ├── UserController.cs
│   └── AuthController.cs
│
├── 📁 Services/                 # BUSINESS LOGIC LAYER
│   ├── 📁 Interfaces/
│   │   ├── IUserService.cs
│   │   └── IAuthService.cs
│   ├── UserService.cs
│   └── AuthService.cs
│
├── 📁 Repositories/             # DATA ACCESS LAYER
│   ├── 📁 Interfaces/
│   │   ├── IUserRepository.cs
│   │   └── IAuthRepository.cs
│   ├── UserRepository.cs
│   └── AuthRepository.cs
│
├── 📁 Data/                     # DATABASE CONTEXT
│   └── ApplicationDbContext.cs
│
├── 📁 Models/                   # DOMAIN ENTITIES
│   ├── User.cs
│   └── LoginRequest.cs
│
├── 📁 DTOs/                     # DATA TRANSFER OBJECTS
│   ├── UserDto.cs
│   └── CreateUserDto.cs
│
├── 📁 Validators/               # FLUENT VALIDATION
│   ├── CreateUserValidator.cs
│   └── LoginRequestValidator.cs
│
├── 📁 Helpers/                  # UTILITIES
│   └── CacheHelper.cs
│
├── Program.cs
└── appsettings.json
```

### Diagramma dei Layer

```
┌─────────────────────────────────────────────┐
│         CLIENT (Browser, Postman)           │
└──────────────────┬──────────────────────────┘
                   │ HTTP Request
                   ▼
┌─────────────────────────────────────────────┐
│       PRESENTATION LAYER (Controllers)      │
│  • Routing HTTP                             │
│  • Validazione input                        │
│  • Restituzione status codes                │
└──────────────────┬──────────────────────────┘
                   │ Chiama Service
                   ▼
┌─────────────────────────────────────────────┐
│      BUSINESS LOGIC LAYER (Services)        │
│  • Business logic                           │
│  • Validazioni business rules               │
│  • Mapping Entity ↔ DTO                     │
└──────────────────┬──────────────────────────┘
                   │ Chiama Repository
                   ▼
┌─────────────────────────────────────────────┐
│      DATA ACCESS LAYER (Repositories)       │
│  • CRUD Operations                          │
│  • Caching                                  │
│  • Query Database                           │
└──────────────────┬──────────────────────────┘
                   │ Entity Framework
                   ▼
┌─────────────────────────────────────────────┐
│            DATABASE (SQL Server)            │
└─────────────────────────────────────────────┘
```

---

## Setup Iniziale

### Passo 1: Creazione Progetto

```bash
# Crea nuovo progetto Web API
dotnet new webapi -n SampleApi
cd SampleApi

# Apri in VS Code
code .
```

### Passo 2: Installazione Pacchetti NuGet

```bash
# Entity Framework Core
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.EntityFrameworkCore.Design

# FluentValidation
dotnet add package FluentValidation
dotnet add package FluentValidation.AspNetCore

# Caching (già incluso in ASP.NET Core)
# dotnet add package Microsoft.Extensions.Caching.Memory

# Opzionale: Redis per Distributed Cache
# dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

### Passo 3: Configurazione appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=SampleApiDb;Trusted_Connection=true;MultipleActiveResultSets=true"
  },
  "CacheSettings": {
    "DefaultExpirationMinutes": 10,
    "UserCacheMinutes": 15,
    "ShortCacheMinutes": 5
  },
  "JwtSettings": {
    "SecretKey": "your-super-secret-key-min-32-chars",
    "Issuer": "SampleApi",
    "Audience": "SampleApiUsers",
    "ExpirationMinutes": 60
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "SampleApi.Repositories": "Debug"
    }
  },
  "AllowedHosts": "*"
}
```

---

## Implementazione Passo-Passo

### STEP 1: Models - Domain Entities

#### `Models/User.cs`
```csharp
namespace SampleApi.Models
{
    public class User
    {
        public int Id { get; set; }
        public string Username { get; set; } = string.Empty;
        public string Email { get; set; } = string.Empty;
        public string Password { get; set; } = string.Empty;
        public string Role { get; set; } = "User";
        public string? JobTitle { get; set; }
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        public DateTime? UpdatedAt { get; set; }
    }
}
```

#### `Models/LoginRequest.cs`
```csharp
namespace SampleApi.Models
{
    public class LoginRequest
    {
        public string Username { get; set; } = string.Empty;
        public string Password { get; set; } = string.Empty;
    }
}
```

---

### STEP 2: DTOs - Data Transfer Objects

#### `DTOs/UserDto.cs`
```csharp
namespace SampleApi.DTOs
{
    public class UserDto
    {
        public int Id { get; set; }
        public string Username { get; set; } = string.Empty;
        public string Email { get; set; } = string.Empty;
        public string Role { get; set; } = string.Empty;
        public string? JobTitle { get; set; }
        public DateTime CreatedAt { get; set; }
    }
}
```

#### `DTOs/CreateUserDto.cs`
```csharp
namespace SampleApi.DTOs
{
    public class CreateUserDto
    {
        public string Username { get; set; } = string.Empty;
        public string Email { get; set; } = string.Empty;
        public string Password { get; set; } = string.Empty;
        public string? JobTitle { get; set; }
    }
}
```

#### `DTOs/UpdateUserDto.cs`
```csharp
namespace SampleApi.DTOs
{
    public class UpdateUserDto
    {
        public string? Username { get; set; }
        public string? Email { get; set; }
        public string? Password { get; set; }
        public string? Role { get; set; }
        public string? JobTitle { get; set; }
    }
}
```

#### `DTOs/LoginResponseDto.cs`
```csharp
namespace SampleApi.DTOs
{
    public class LoginResponseDto
    {
        public string Token { get; set; } = string.Empty;
        public UserDto User { get; set; } = null!;
    }
}
```

---

### STEP 3: Database Context

#### `Data/ApplicationDbContext.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using SampleApi.Models;

namespace SampleApi.Data
{
    public class ApplicationDbContext : DbContext
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }

        public DbSet<User> Users { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            // Configurazione User
            modelBuilder.Entity<User>(entity =>
            {
                entity.HasKey(e => e.Id);
                
                entity.Property(e => e.Username)
                    .IsRequired()
                    .HasMaxLength(50);
                
                entity.Property(e => e.Email)
                    .IsRequired()
                    .HasMaxLength(100);
                
                entity.Property(e => e.Password)
                    .IsRequired()
                    .HasMaxLength(255);
                
                entity.Property(e => e.Role)
                    .IsRequired()
                    .HasMaxLength(20)
                    .HasDefaultValue("User");
                
                entity.Property(e => e.JobTitle)
                    .HasMaxLength(100);

                // Indici
                entity.HasIndex(e => e.Username).IsUnique();
                entity.HasIndex(e => e.Email).IsUnique();
            });

            // Seed Data (opzionale)
            modelBuilder.Entity<User>().HasData(
                new User
                {
                    Id = 1,
                    Username = "admin",
                    Email = "admin@example.com",
                    Password = "Admin123!", // In produzione deve essere hashata
                    Role = "Admin",
                    JobTitle = "System Administrator",
                    CreatedAt = DateTime.UtcNow
                }
            );
        }
    }
}
```

---

### STEP 4: Repository Layer - Data Access

#### `Repositories/Interfaces/IUserRepository.cs`
```csharp
using SampleApi.Models;

namespace SampleApi.Repositories.Interfaces
{
    public interface IUserRepository
    {
        Task<User?> GetByIdAsync(int id);
        Task<User?> GetByUsernameAsync(string username);
        Task<User?> GetByEmailAsync(string email);
        Task<IEnumerable<User>> GetAllAsync();
        Task<User> CreateAsync(User user);
        Task<User> UpdateAsync(User user);
        Task DeleteAsync(int id);
        Task<bool> ExistsAsync(int id);
    }
}
```

#### `Repositories/UserRepository.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Caching.Memory;
using SampleApi.Data;
using SampleApi.Models;
using SampleApi.Repositories.Interfaces;

namespace SampleApi.Repositories
{
    public class UserRepository : IUserRepository
    {
        private readonly ApplicationDbContext _context;
        private readonly IMemoryCache _cache;
        private readonly ILogger<UserRepository> _logger;
        
        // Cache Keys
        private const string CACHE_KEY_PREFIX = "user_";
        private const string CACHE_KEY_ALL = "users_all";
        private const string CACHE_KEY_USERNAME = "user_username_";
        private const string CACHE_KEY_EMAIL = "user_email_";
        
        // Cache Options
        private readonly MemoryCacheEntryOptions _cacheOptions;

        public UserRepository(
            ApplicationDbContext context, 
            IMemoryCache cache,
            ILogger<UserRepository> logger,
            IConfiguration configuration)
        {
            _context = context;
            _cache = cache;
            _logger = logger;

            var cacheMinutes = configuration.GetValue<int>("CacheSettings:UserCacheMinutes", 15);
            
            _cacheOptions = new MemoryCacheEntryOptions()
                .SetSlidingExpiration(TimeSpan.FromMinutes(cacheMinutes))
                .SetAbsoluteExpiration(TimeSpan.FromMinutes(cacheMinutes * 2))
                .SetPriority(CacheItemPriority.Normal)
                .RegisterPostEvictionCallback((key, value, reason, state) =>
                {
                    _logger.LogDebug("Cache evicted - Key: {Key}, Reason: {Reason}", key, reason);
                });
        }

        public async Task<User?> GetByIdAsync(int id)
        {
            string cacheKey = $"{CACHE_KEY_PREFIX}{id}";

            if (_cache.TryGetValue(cacheKey, out User? cachedUser))
            {
                _logger.LogDebug("Cache HIT - User ID: {UserId}", id);
                return cachedUser;
            }

            _logger.LogDebug("Cache MISS - User ID: {UserId}, querying database", id);

            var user = await _context.Users
                .AsNoTracking()
                .FirstOrDefaultAsync(u => u.Id == id);

            if (user != null)
            {
                _cache.Set(cacheKey, user, _cacheOptions);
                _logger.LogDebug("User ID: {UserId} cached", id);
            }

            return user;
        }

        public async Task<User?> GetByUsernameAsync(string username)
        {
            string cacheKey = $"{CACHE_KEY_USERNAME}{username.ToLower()}";

            if (_cache.TryGetValue(cacheKey, out User? cachedUser))
            {
                _logger.LogDebug("Cache HIT - Username: {Username}", username);
                return cachedUser;
            }

            _logger.LogDebug("Cache MISS - Username: {Username}", username);

            var user = await _context.Users
                .AsNoTracking()
                .FirstOrDefaultAsync(u => u.Username == username);

            if (user != null)
            {
                _cache.Set(cacheKey, user, _cacheOptions);
            }

            return user;
        }

        public async Task<User?> GetByEmailAsync(string email)
        {
            string cacheKey = $"{CACHE_KEY_EMAIL}{email.ToLower()}";

            if (_cache.TryGetValue(cacheKey, out User? cachedUser))
            {
                _logger.LogDebug("Cache HIT - Email: {Email}", email);
                return cachedUser;
            }

            var user = await _context.Users
                .AsNoTracking()
                .FirstOrDefaultAsync(u => u.Email == email);

            if (user != null)
            {
                _cache.Set(cacheKey, user, _cacheOptions);
            }

            return user;
        }

        public async Task<IEnumerable<User>> GetAllAsync()
        {
            if (_cache.TryGetValue(CACHE_KEY_ALL, out IEnumerable<User>? cachedUsers))
            {
                _logger.LogDebug("Cache HIT - All users");
                return cachedUsers!;
            }

            _logger.LogDebug("Cache MISS - All users");

            var users = await _context.Users
                .AsNoTracking()
                .OrderBy(u => u.Username)
                .ToListAsync();

            _cache.Set(CACHE_KEY_ALL, users, _cacheOptions);

            return users;
        }

        public async Task<User> CreateAsync(User user)
        {
            _context.Users.Add(user);
            await _context.SaveChangesAsync();

            _logger.LogInformation("User created - ID: {UserId}, Username: {Username}", 
                user.Id, user.Username);

            // Invalida cache lista
            InvalidateListCache();

            return user;
        }

        public async Task<User> UpdateAsync(User user)
        {
            user.UpdatedAt = DateTime.UtcNow;
            
            _context.Users.Update(user);
            await _context.SaveChangesAsync();

            _logger.LogInformation("User updated - ID: {UserId}", user.Id);

            // Invalida tutte le cache relative a questo utente
            InvalidateUserCache(user.Id, user.Username, user.Email);

            return user;
        }

        public async Task DeleteAsync(int id)
        {
            var user = await _context.Users.FindAsync(id);

            if (user != null)
            {
                var username = user.Username;
                var email = user.Email;

                _context.Users.Remove(user);
                await _context.SaveChangesAsync();

                _logger.LogInformation("User deleted - ID: {UserId}", id);

                InvalidateUserCache(id, username, email);
            }
        }

        public async Task<bool> ExistsAsync(int id)
        {
            string cacheKey = $"{CACHE_KEY_PREFIX}{id}";
            
            if (_cache.TryGetValue(cacheKey, out User? _))
            {
                return true;
            }

            return await _context.Users.AnyAsync(u => u.Id == id);
        }

        // Helper Methods per Invalidazione Cache
        private void InvalidateUserCache(int userId, string username, string email)
        {
            _cache.Remove($"{CACHE_KEY_PREFIX}{userId}");
            _cache.Remove($"{CACHE_KEY_USERNAME}{username.ToLower()}");
            _cache.Remove($"{CACHE_KEY_EMAIL}{email.ToLower()}");
            InvalidateListCache();
            
            _logger.LogDebug("Cache invalidated for user ID: {UserId}", userId);
        }

        private void InvalidateListCache()
        {
            _cache.Remove(CACHE_KEY_ALL);
            _logger.LogDebug("All users cache invalidated");
        }
    }
}
```

#### `Repositories/Interfaces/IAuthRepository.cs`
```csharp
using SampleApi.Models;

namespace SampleApi.Repositories.Interfaces
{
    public interface IAuthRepository
    {
        Task<User?> ValidateCredentialsAsync(string username, string password);
        Task<bool> IsUsernameAvailableAsync(string username);
        Task<bool> IsEmailAvailableAsync(string email);
    }
}
```

#### `Repositories/AuthRepository.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using SampleApi.Data;
using SampleApi.Models;
using SampleApi.Repositories.Interfaces;

namespace SampleApi.Repositories
{
    public class AuthRepository : IAuthRepository
    {
        private readonly ApplicationDbContext _context;

        public AuthRepository(ApplicationDbContext context)
        {
            _context = context;
        }

        public async Task<User?> ValidateCredentialsAsync(string username, string password)
        {
            // Nota: In produzione la password deve essere verificata con hash
            return await _context.Users
                .AsNoTracking()
                .FirstOrDefaultAsync(u => u.Username == username && u.Password == password);
        }

        public async Task<bool> IsUsernameAvailableAsync(string username)
        {
            return !await _context.Users.AnyAsync(u => u.Username == username);
        }

        public async Task<bool> IsEmailAvailableAsync(string email)
        {
            return !await _context.Users.AnyAsync(u => u.Email == email);
        }
    }
}
```

---

### STEP 5: Service Layer - Business Logic

#### `Services/Interfaces/IUserService.cs`
```csharp
using SampleApi.DTOs;
using SampleApi.Models;

namespace SampleApi.Services.Interfaces
{
    public interface IUserService
    {
        Task<UserDto?> GetUserByIdAsync(int id);
        Task<UserDto?> GetUserByUsernameAsync(string username);
        Task<IEnumerable<UserDto>> GetAllUsersAsync();
        Task<UserDto> CreateUserAsync(CreateUserDto createUserDto);
        Task<UserDto> UpdateUserAsync(int id, UpdateUserDto updateUserDto);
        Task DeleteUserAsync(int id);
        Task<bool> UserExistsAsync(int id);
    }
}
```

#### `Services/UserService.cs`
```csharp
using SampleApi.DTOs;
using SampleApi.Models;
using SampleApi.Repositories.Interfaces;
using SampleApi.Services.Interfaces;

namespace SampleApi.Services
{
    public class UserService : IUserService
    {
        private readonly IUserRepository _userRepository;
        private readonly ILogger<UserService> _logger;

        public UserService(IUserRepository userRepository, ILogger<UserService> logger)
        {
            _userRepository = userRepository;
            _logger = logger;
        }

        public async Task<UserDto?> GetUserByIdAsync(int id)
        {
            var user = await _userRepository.GetByIdAsync(id);
            return user == null ? null : MapToDto(user);
        }

        public async Task<UserDto?> GetUserByUsernameAsync(string username)
        {
            var user = await _userRepository.GetByUsernameAsync(username);
            return user == null ? null : MapToDto(user);
        }

        public async Task<IEnumerable<UserDto>> GetAllUsersAsync()
        {
            var users = await _userRepository.GetAllAsync();
            return users.Select(MapToDto);
        }

        public async Task<UserDto> CreateUserAsync(CreateUserDto createUserDto)
        {
            // Business Logic: verifica username univoco
            var existingUser = await _userRepository.GetByUsernameAsync(createUserDto.Username);
            if (existingUser != null)
            {
                throw new InvalidOperationException($"Username '{createUserDto.Username}' is already taken");
            }

            // Business Logic: verifica email univoca
            var existingEmail = await _userRepository.GetByEmailAsync(createUserDto.Email);
            if (existingEmail != null)
            {
                throw new InvalidOperationException($"Email '{createUserDto.Email}' is already registered");
            }

            var user = new User
            {
                Username = createUserDto.Username,
                Email = createUserDto.Email,
                Password = HashPassword(createUserDto.Password), // Hash password
                JobTitle = createUserDto.JobTitle,
                Role = "User", // Default role
                CreatedAt = DateTime.UtcNow
            };

            var createdUser = await _userRepository.CreateAsync(user);
            
            _logger.LogInformation("User created successfully - Username: {Username}", createdUser.Username);
            
            return MapToDto(createdUser);
        }

        public async Task<UserDto> UpdateUserAsync(int id, UpdateUserDto updateUserDto)
        {
            var existingUser = await _userRepository.GetByIdAsync(id);
            
            if (existingUser == null)
            {
                throw new KeyNotFoundException($"User with ID {id} not found");
            }

            // Business Logic: aggiorna solo campi forniti
            if (!string.IsNullOrWhiteSpace(updateUserDto.Username))
            {
                // Verifica che il nuovo username non sia già usato
                var userWithUsername = await _userRepository.GetByUsernameAsync(updateUserDto.Username);
                if (userWithUsername != null && userWithUsername.Id != id)
                {
                    throw new InvalidOperationException($"Username '{updateUserDto.Username}' is already taken");
                }
                existingUser.Username = updateUserDto.Username;
            }

            if (!string.IsNullOrWhiteSpace(updateUserDto.Email))
            {
                var userWithEmail = await _userRepository.GetByEmailAsync(updateUserDto.Email);
                if (userWithEmail != null && userWithEmail.Id != id)
                {
                    throw new InvalidOperationException($"Email '{updateUserDto.Email}' is already registered");
                }
                existingUser.Email = updateUserDto.Email;
            }

            if (!string.IsNullOrWhiteSpace(updateUserDto.Password))
            {
                existingUser.Password = HashPassword(updateUserDto.Password);
            }

            if (!string.IsNullOrWhiteSpace(updateUserDto.Role))
            {
                existingUser.Role = updateUserDto.Role;
            }

            if (updateUserDto.JobTitle != null)
            {
                existingUser.JobTitle = updateUserDto.JobTitle;
            }

            var updatedUser = await _userRepository.UpdateAsync(existingUser);

            _logger.LogInformation("User updated successfully - ID: {UserId}", id);

            return MapToDto(updatedUser);
        }

        public async Task DeleteUserAsync(int id)
        {
            var exists = await _userRepository.ExistsAsync(id);
            
            if (!exists)
            {
                throw new KeyNotFoundException($"User with ID {id} not found");
            }

            await _userRepository.DeleteAsync(id);
            
            _logger.LogInformation("User deleted successfully - ID: {UserId}", id);
        }

        public async Task<bool> UserExistsAsync(int id)
        {
            return await _userRepository.ExistsAsync(id);
        }

        // Helper Methods
        private UserDto MapToDto(User user)
        {
            return new UserDto
            {
                Id = user.Id,
                Username = user.Username,
                Email = user.Email,
                Role = user.Role,
                JobTitle = user.JobTitle,
                CreatedAt = user.CreatedAt
            };
        }

        private string HashPassword(string password)
        {
            // TODO: Implementa BCrypt o PBKDF2
            // using BCrypt.Net;
            // return BCrypt.HashPassword(password);
            
            // PLACEHOLDER - NON USARE IN PRODUZIONE
            return password;
        }
    }
}
```

#### `Services/Interfaces/IAuthService.cs`
```csharp
using SampleApi.DTOs;
using SampleApi.Models;

namespace SampleApi.Services.Interfaces
{
    public interface IAuthService
    {
        Task<LoginResponseDto?> AuthenticateAsync(LoginRequest loginRequest);
        Task<bool> IsUsernameAvailableAsync(string username);
    }
}
```

#### `Services/AuthService.cs`
```csharp
using SampleApi.DTOs;
using SampleApi.Models;
using SampleApi.Repositories.Interfaces;
using SampleApi.Services.Interfaces;

namespace SampleApi.Services
{
    public class AuthService : IAuthService
    {
        private readonly IAuthRepository _authRepository;
        private readonly ILogger<AuthService> _logger;

        public AuthService(IAuthRepository authRepository, ILogger<AuthService> logger)
        {
            _authRepository = authRepository;
            _logger = logger;
        }

        public async Task<LoginResponseDto?> AuthenticateAsync(LoginRequest loginRequest)
        {
            var hashedPassword = HashPassword(loginRequest.Password);
            
            var user = await _authRepository.ValidateCredentialsAsync(
                loginRequest.Username, 
                hashedPassword
            );

            if (user == null)
            {
                _logger.LogWarning("Failed login attempt - Username: {Username}", loginRequest.Username);
                return null;
            }

            var token = GenerateJwtToken(user);

            _logger.LogInformation("User authenticated - Username: {Username}", loginRequest.Username);

            return new LoginResponseDto
            {
                Token = token,
                User = new UserDto
                {
                    Id = user.Id,
                    Username = user.Username,
                    Email = user.Email,
                    Role = user.Role,
                    JobTitle = user.JobTitle,
                    CreatedAt = user.CreatedAt
                }
            };
        }

        public async Task<bool> IsUsernameAvailableAsync(string username)
        {
            return await _authRepository.IsUsernameAvailableAsync(username);
        }

        private string HashPassword(string password)
        {
            // TODO: Implementa hashing sicuro
            return password;
        }

        private string GenerateJwtToken(User user)
        {
            // TODO: Implementa generazione JWT token
            return $"token_for_{user.Username}";
        }
    }
}
```

---

### STEP 6: FluentValidation - Validators

#### `Validators/CreateUserValidator.cs`
```csharp
using FluentValidation;
using SampleApi.DTOs;

namespace SampleApi.Validators
{
    public class CreateUserValidator : AbstractValidator<CreateUserDto>
    {
        public CreateUserValidator()
        {
            RuleFor(x => x.Username)
                .NotEmpty().WithMessage("Username is required")
                .MinimumLength(3).WithMessage("Username must be at least 3 characters")
                .MaximumLength(50).WithMessage("Username cannot exceed 50 characters")
                .Matches(@"^[a-zA-Z0-9_]+$").WithMessage("Username can only contain letters, numbers, and underscores");

            RuleFor(x => x.Email)
                .NotEmpty().WithMessage("Email is required")
                .EmailAddress().WithMessage("Invalid email format")
                .MaximumLength(100).WithMessage("Email cannot exceed 100 characters");

            RuleFor(x => x.Password)
                .NotEmpty().WithMessage("Password is required")
                .MinimumLength(8).WithMessage("Password must be at least 8 characters")
                .Matches(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)")
                .WithMessage("Password must contain at least one uppercase letter, one lowercase letter, and one digit");

            RuleFor(x => x.JobTitle)
                .MaximumLength(100).WithMessage("Job title cannot exceed 100 characters")
                .When(x => !string.IsNullOrEmpty(x.JobTitle));
        }
    }
}
```

#### `Validators/UpdateUserValidator.cs`
```csharp
using FluentValidation;
using SampleApi.DTOs;

namespace SampleApi.Validators
{
    public class UpdateUserValidator : AbstractValidator<UpdateUserDto>
    {
        public UpdateUserValidator()
        {
            RuleFor(x => x.Username)
                .MinimumLength(3).WithMessage("Username must be at least 3 characters")
                .MaximumLength(50).WithMessage("Username cannot exceed 50 characters")
                .Matches(@"^[a-zA-Z0-9_]+$").WithMessage("Username can only contain letters, numbers, and underscores")
                .When(x => !string.IsNullOrWhiteSpace(x.Username));

            RuleFor(x => x.Email)
                .EmailAddress().WithMessage("Invalid email format")
                .MaximumLength(100).WithMessage("Email cannot exceed 100 characters")
                .When(x => !string.IsNullOrWhiteSpace(x.Email));

            RuleFor(x => x.Password)
                .MinimumLength(8).WithMessage("Password must be at least 8 characters")
                .Matches(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)")
                .WithMessage("Password must contain at least one uppercase letter, one lowercase letter, and one digit")
                .When(x => !string.IsNullOrWhiteSpace(x.Password));

            RuleFor(x => x.Role)
                .Must(role => new[] { "User", "Admin", "Moderator" }.Contains(role))
                .WithMessage("Role must be one of: User, Admin, Moderator")
                .When(x => !string.IsNullOrWhiteSpace(x.Role));

            RuleFor(x => x.JobTitle)
                .MaximumLength(100).WithMessage("Job title cannot exceed 100 characters")
                .When(x => !string.IsNullOrEmpty(x.JobTitle));
        }
    }
}
```

#### `Validators/LoginRequestValidator.cs`
```csharp
using FluentValidation;
using SampleApi.Models;

namespace SampleApi.Validators
{
    public class LoginRequestValidator : AbstractValidator<LoginRequest>
    {
        public LoginRequestValidator()
        {
            RuleFor(x => x.Username)
                .NotEmpty().WithMessage("Username is required")
                .MinimumLength(3).WithMessage("Username must be at least 3 characters");

            RuleFor(x => x.Password)
                .NotEmpty().WithMessage("Password is required")
                .MinimumLength(8).WithMessage("Password must be at least 8 characters");
        }
    }
}
```

---

### STEP 7: Controllers - Presentation Layer

#### `Controllers/UserController.cs`
```csharp
using Microsoft.AspNetCore.Mvc;
using SampleApi.DTOs;
using SampleApi.Services.Interfaces;

namespace SampleApi.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    [Produces("application/json")]
    public class UserController : ControllerBase
    {
        private readonly IUserService _userService;
        private readonly ILogger<UserController> _logger;

        public UserController(IUserService userService, ILogger<UserController> logger)
        {
            _userService = userService;
            _logger = logger;
        }

        /// <summary>
        /// Ottiene tutti gli utenti
        /// </summary>
        /// <returns>Lista di utenti</returns>
        [HttpGet]
        [ProducesResponseType(typeof(IEnumerable<UserDto>), StatusCodes.Status200OK)]
        public async Task<ActionResult<IEnumerable<UserDto>>> GetAllUsers()
        {
            _logger.LogInformation("Getting all users");
            
            var users = await _userService.GetAllUsersAsync();
            return Ok(users);
        }

        /// <summary>
        /// Ottiene un utente per ID
        /// </summary>
        /// <param name="id">ID dell'utente</param>
        /// <returns>Dettagli utente</returns>
        [HttpGet("{id}")]
        [ProducesResponseType(typeof(UserDto), StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<ActionResult<UserDto>> GetUser(int id)
        {
            _logger.LogInformation("Getting user with ID: {UserId}", id);

            var user = await _userService.GetUserByIdAsync(id);

            if (user == null)
            {
                _logger.LogWarning("User not found - ID: {UserId}", id);
                return NotFound(new { message = $"User with ID {id} not found" });
            }

            return Ok(user);
        }

        /// <summary>
        /// Ottiene un utente per username
        /// </summary>
        /// <param name="username">Username dell'utente</param>
        /// <returns>Dettagli utente</returns>
        [HttpGet("username/{username}")]
        [ProducesResponseType(typeof(UserDto), StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<ActionResult<UserDto>> GetUserByUsername(string username)
        {
            _logger.LogInformation("Getting user by username: {Username}", username);

            var user = await _userService.GetUserByUsernameAsync(username);

            if (user == null)
            {
                _logger.LogWarning("User not found - Username: {Username}", username);
                return NotFound(new { message = $"User '{username}' not found" });
            }

            return Ok(user);
        }

        /// <summary>
        /// Crea un nuovo utente
        /// </summary>
        /// <param name="createUserDto">Dati del nuovo utente</param>
        /// <returns>Utente creato</returns>
        [HttpPost]
        [ProducesResponseType(typeof(UserDto), StatusCodes.Status201Created)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        public async Task<ActionResult<UserDto>> CreateUser([FromBody] CreateUserDto createUserDto)
        {
            _logger.LogInformation("Creating new user - Username: {Username}", createUserDto.Username);

            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }

            try
            {
                var createdUser = await _userService.CreateUserAsync(createUserDto);

                return CreatedAtAction(
                    nameof(GetUser),
                    new { id = createdUser.Id },
                    createdUser
                );
            }
            catch (InvalidOperationException ex)
            {
                _logger.LogWarning(ex, "Failed to create user - Username: {Username}", createUserDto.Username);
                return BadRequest(new { message = ex.Message });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error creating user - Username: {Username}", createUserDto.Username);
                return StatusCode(500, new { message = "An error occurred while creating the user" });
            }
        }

        /// <summary>
        /// Aggiorna un utente esistente
        /// </summary>
        /// <param name="id">ID dell'utente da aggiornare</param>
        /// <param name="updateUserDto">Nuovi dati dell'utente</param>
        /// <returns>Utente aggiornato</returns>
        [HttpPut("{id}")]
        [ProducesResponseType(typeof(UserDto), StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        public async Task<ActionResult<UserDto>> UpdateUser(int id, [FromBody] UpdateUserDto updateUserDto)
        {
            _logger.LogInformation("Updating user - ID: {UserId}", id);

            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }

            try
            {
                var updatedUser = await _userService.UpdateUserAsync(id, updateUserDto);
                return Ok(updatedUser);
            }
            catch (KeyNotFoundException ex)
            {
                _logger.LogWarning(ex, "User not found - ID: {UserId}", id);
                return NotFound(new { message = ex.Message });
            }
            catch (InvalidOperationException ex)
            {
                _logger.LogWarning(ex, "Invalid operation while updating user - ID: {UserId}", id);
                return BadRequest(new { message = ex.Message });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error updating user - ID: {UserId}", id);
                return StatusCode(500, new { message = "An error occurred while updating the user" });
            }
        }

        /// <summary>
        /// Elimina un utente
        /// </summary>
        /// <param name="id">ID dell'utente da eliminare</param>
        /// <returns>No content</returns>
        [HttpDelete("{id}")]
        [ProducesResponseType(StatusCodes.Status204NoContent)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<IActionResult> DeleteUser(int id)
        {
            _logger.LogInformation("Deleting user - ID: {UserId}", id);

            try
            {
                await _userService.DeleteUserAsync(id);
                return NoContent();
            }
            catch (KeyNotFoundException ex)
            {
                _logger.LogWarning(ex, "User not found - ID: {UserId}", id);
                return NotFound(new { message = ex.Message });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error deleting user - ID: {UserId}", id);
                return StatusCode(500, new { message = "An error occurred while deleting the user" });
            }
        }

        /// <summary>
        /// Verifica se un utente esiste
        /// </summary>
        /// <param name="id">ID dell'utente</param>
        /// <returns>True se esiste, altrimenti False</returns>
        [HttpHead("{id}/exists")]
        [HttpGet("{id}/exists")]
        [ProducesResponseType(StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<IActionResult> UserExists(int id)
        {
            var exists = await _userService.UserExistsAsync(id);

            if (exists)
            {
                return Ok(new { exists = true });
            }

            return NotFound(new { exists = false });
        }
    }
}
```

#### `Controllers/AuthController.cs`
```csharp
using Microsoft.AspNetCore.Mvc;
using SampleApi.DTOs;
using SampleApi.Models;
using SampleApi.Services.Interfaces;

namespace SampleApi.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    [Produces("application/json")]
    public class AuthController : ControllerBase
    {
        private readonly IAuthService _authService;
        private readonly ILogger<AuthController> _logger;

        public AuthController(IAuthService authService, ILogger<AuthController> logger)
        {
            _authService = authService;
            _logger = logger;
        }

        /// <summary>
        /// Effettua il login
        /// </summary>
        /// <param name="loginRequest">Credenziali di login</param>
        /// <returns>Token JWT e dati utente</returns>
        [HttpPost("login")]
        [ProducesResponseType(typeof(LoginResponseDto), StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status401Unauthorized)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        public async Task<ActionResult<LoginResponseDto>> Login([FromBody] LoginRequest loginRequest)
        {
            _logger.LogInformation("Login attempt - Username: {Username}", loginRequest.Username);

            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }

            try
            {
                var result = await _authService.AuthenticateAsync(loginRequest);

                if (result == null)
                {
                    _logger.LogWarning("Failed login - Invalid credentials for username: {Username}", 
                        loginRequest.Username);
                    return Unauthorized(new { message = "Invalid username or password" });
                }

                _logger.LogInformation("Successful login - Username: {Username}", loginRequest.Username);
                
                return Ok(result);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error during login - Username: {Username}", loginRequest.Username);
                return StatusCode(500, new { message = "An error occurred during login" });
            }
        }

        /// <summary>
        /// Verifica disponibilità username
        /// </summary>
        /// <param name="username">Username da verificare</param>
        /// <returns>True se disponibile</returns>
        [HttpGet("check-username/{username}")]
        [ProducesResponseType(StatusCodes.Status200OK)]
        public async Task<ActionResult<object>> CheckUsername(string username)
        {
            var isAvailable = await _authService.IsUsernameAvailableAsync(username);
            
            return Ok(new { username, available = isAvailable });
        }
    }
}
```

---

### STEP 8: Program.cs - Dependency Injection

#### `Program.cs`
```csharp
using FluentValidation;
using FluentValidation.AspNetCore;
using Microsoft.EntityFrameworkCore;
using SampleApi.Data;
using SampleApi.Repositories;
using SampleApi.Repositories.Interfaces;
using SampleApi.Services;
using SampleApi.Services.Interfaces;

var builder = WebApplication.CreateBuilder(args);

// ============================================
// CONFIGURAZIONE SERVIZI
// ============================================

// Controllers
builder.Services.AddControllers();

// Database Context
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions => sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(30),
            errorNumbersToAdd: null
        )
    )
);

// Memory Cache
builder.Services.AddMemoryCache();

// Repositories (Data Access Layer)
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IAuthRepository, AuthRepository>();

// Services (Business Logic Layer)
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IAuthService, AuthService>();

// FluentValidation
builder.Services.AddValidatorsFromAssemblyContaining<Program>();
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddFluentValidationClientsideAdapters();

// Swagger/OpenAPI
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new Microsoft.OpenApi.Models.OpenApiInfo
    {
        Title = "Sample API",
        Version = "v1",
        Description = "API REST con Repository Pattern, Caching e FluentValidation",
        Contact = new Microsoft.OpenApi.Models.OpenApiContact
        {
            Name = "Il Tuo Nome",
            Email = "tuaemail@example.com"
        }
    });

    // Includi commenti XML
    var xmlFile = $"{System.Reflection.Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    if (File.Exists(xmlPath))
    {
        options.IncludeXmlComments(xmlPath);
    }
});

// CORS (opzionale)
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

// Logging
builder.Logging.ClearProviders();
builder.Logging.AddConsole();
builder.Logging.AddDebug();

var app = builder.Build();

// ============================================
// CONFIGURAZIONE PIPELINE HTTP
// ============================================

// Swagger in Development
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "Sample API v1");
        options.RoutePrefix = string.Empty; // Swagger alla root
    });
}

// HTTPS Redirection
app.UseHttpsRedirection();

// CORS
app.UseCors("AllowAll");

// Authorization
app.UseAuthorization();

// Map Controllers
app.MapControllers();

// ============================================
// DATABASE MIGRATION (opzionale in dev)
// ============================================
using (var scope = app.Services.CreateScope())
{
    var services = scope.ServiceProvider;
    try
    {
        var context = services.GetRequiredService<ApplicationDbContext>();
        
        if (app.Environment.IsDevelopment())
        {
            // Applica automaticamente le migrations in development
            context.Database.Migrate();
            
            var logger = services.GetRequiredService<ILogger<Program>>();
            logger.LogInformation("Database migrated successfully");
        }
    }
    catch (Exception ex)
    {
        var logger = services.GetRequiredService<ILogger<Program>>();
        logger.LogError(ex, "An error occurred while migrating the database");
    }
}

app.Run();
```

---

## Testing

### Migrations e Database

```bash
# Crea migration
dotnet ef migrations add InitialCreate

# Aggiorna database
dotnet ef database update

# Rimuovi ultima migration (se necessario)
dotnet ef migrations remove
```

### Esecuzione Applicazione

```bash
# Run
dotnet run

# Watch (auto-reload)
dotnet watch run
```

### Testing con Postman/curl

#### 1. GET - Tutti gli utenti
```bash
curl -X GET https://localhost:5001/api/user
```

#### 2. GET - Utente per ID
```bash
curl -X GET https://localhost:5001/api/user/1
```

#### 3. POST - Crea utente
```bash
curl -X POST https://localhost:5001/api/user \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john_doe",
    "email": "john@example.com",
    "password": "Password123!",
    "jobTitle": "Software Developer"
  }'
```

#### 4. PUT - Aggiorna utente
```bash
curl -X PUT https://localhost:5001/api/user/1 \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john_updated",
    "role": "Admin"
  }'
```

#### 5. DELETE - Elimina utente
```bash
curl -X DELETE https://localhost:5001/api/user/1
```

#### 6. POST - Login
```bash
curl -X POST https://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "Admin123!"
  }'
```

### Collezione Postman

Crea una collezione Postman con queste richieste salvate per testing rapido:

**Environment Variables:**
```json
{
  "baseUrl": "https://localhost:5001",
  "userId": "1",
  "token": ""
}
```

---

## Best Practices

### 1. **Separazione delle Responsabilità**
- ✅ Controller: Solo HTTP routing e status codes
- ✅ Service: Business logic e coordinamento
- ✅ Repository: Solo accesso ai dati e caching

### 2. **Sicurezza**
- ⚠️ **MAI** restituire password nei DTO
- ⚠️ Sempre hash delle password (usa BCrypt)
- ⚠️ Implementa JWT per autenticazione
- ⚠️ Usa HTTPS in produzione
- ⚠️ Valida sempre l'input con FluentValidation

### 3. **Caching**
- Cache solo dati read-heavy
- Invalida cache dopo modifiche (Create/Update/Delete)
- Usa expiration appropriata (5-15 minuti)
- Log cache hits/misses per monitoring

### 4. **Error Handling**
- Gestisci eccezioni specifiche (KeyNotFoundException, InvalidOperationException)
- Log sempre gli errori
- Restituisci messaggi user-friendly
- Non esporre stack traces in produzione

### 5. **Logging**
- Log operazioni importanti (Create, Update, Delete)
- Log tentativi di login falliti
- Log cache hits/misses in Debug mode
- Usa livelli appropriati (Debug, Information, Warning, Error)

### 6. **Validazione**
- FluentValidation per input validation
- Business rules validation nel Service Layer
- Model State validation nel Controller

### 7. **Performance**
- Usa AsNoTracking() per query read-only
- Implementa paginazione per liste grandi
- Cache query frequenti
- Usa indici appropriati nel database

### 8. **Testing**
```csharp
// Unit Test esempio
[Fact]
public async Task GetUserById_ReturnsUser_WhenUserExists()
{
    // Arrange
    var mockRepo = new Mock<IUserRepository>();
    mockRepo.Setup(r => r.GetByIdAsync(1))
        .ReturnsAsync(new User { Id = 1, Username = "test" });
    
    var service = new UserService(mockRepo.Object, Mock.Of<ILogger<UserService>>());
    
    // Act
    var result = await service.GetUserByIdAsync(1);
    
    // Assert
    Assert.NotNull(result);
    Assert.Equal("test", result.Username);
}
```

---

## Prossimi Passi

### 1. Implementa JWT Authentication
- Genera token JWT nel login
- Proteggi endpoint con `[Authorize]`
- Gestisci ruoli e claims

### 2. Aggiungi Password Hashing
```bash
dotnet add package BCrypt.Net-Next
```

### 3. Implementa Paginazione
```csharp
public async Task<PagedResult<UserDto>> GetUsersAsync(int page, int pageSize)
```

### 4. Aggiungi Distributed Cache (Redis)
```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

### 5. Unit Testing
```bash
dotnet add package xunit
dotnet add package Moq
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

### 6. API Versioning
```bash
dotnet add package Microsoft.AspNetCore.Mvc.Versioning
```

### 7. Rate Limiting
```bash
dotnet add package AspNetCoreRateLimit
```

---

## Risorse Aggiuntive

- [ASP.NET Core Documentation](https://docs.microsoft.com/aspnet/core)
- [Entity Framework Core](https://docs.microsoft.com/ef/core)
- [FluentValidation](https://docs.fluentvalidation.net)
- [Repository Pattern](https://docs.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design)

---

## Conclusione

Hai ora una struttura completa e scalabile per un'API REST professionale con:
- ✅ Repository Pattern correttamente implementato
- ✅ Caching per performance ottimizzate
- ✅ Validazione robusta con FluentValidation
- ✅ CRUD completo (GET, POST, PUT, DELETE)
- ✅ Logging e error handling
- ✅ Separazione chiara delle responsabilità

Questa architettura è:
- **Testabile** - Ogni layer può essere testato indipendentemente
- **Manutenibile** - Codice organizzato e pulito
- **Scalabile** - Pronta per crescere
- **Professionale** - Segue best practices industry-standard

Buon coding! 🚀
