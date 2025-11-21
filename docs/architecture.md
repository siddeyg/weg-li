# Architecture Documentation

## Overview

weg-li is a Ruby on Rails 7.2 application designed to empower German citizens to report parking violations through an intelligent, automated platform. The system combines photo evidence, AI-powered analysis, and direct integration with local authorities.

## Technology Stack

### Backend
- **Ruby**: 3.2.9
- **Rails**: 7.2.2.2
- **Database**: PostgreSQL with PostGIS extension
- **Background Jobs**: Redis + Sidekiq
- **Web Server**: Puma

### Frontend
- **Asset Pipeline**: Webpacker (Webpack 5)
- **JavaScript**: Stimulus controllers, Turbo Rails
- **CSS**: Bootstrap 5, Sass
- **Templates**: Slim

### External Services
- **Google Cloud Platform**:
  - Cloud Vision API (license plate recognition)
  - Cloud Storage (photo storage)
- **Geocoding**: Nominatim/OpenCage
- **Email**: Sendgrid (sending), Action Mailbox (receiving)
- **OAuth Providers**: GitHub, Twitter, Google, Telegram, Amazon, Apple

### Infrastructure
- **Deployment**: Docker support, Render.com, Heroku
- **Storage**: Google Cloud Storage buckets
- **Monitoring**: Slack webhooks
- **Scheduling**: Sidekiq-scheduler

## System Architecture

```
┌─────────────┐
│   Browser   │
└──────┬──────┘
       │
       ├─── HTTP/HTML ───────────────┐
       │                             │
       └─── REST API ────────────┐   │
                                 ▼   ▼
                         ┌──────────────────┐
                         │  Rails Server    │
                         │  (Puma)          │
                         └────────┬─────────┘
                                  │
                  ┌───────────────┼───────────────┐
                  │               │               │
                  ▼               ▼               ▼
          ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
          │  PostgreSQL  │ │    Redis     │ │   Sidekiq    │
          │  + PostGIS   │ │              │ │  (Workers)   │
          └──────────────┘ └──────────────┘ └──────┬───────┘
                                                    │
                  ┌─────────────────────────────────┼─────────────────┐
                  │                                 │                 │
                  ▼                                 ▼                 ▼
          ┌──────────────┐                 ┌──────────────┐  ┌──────────────┐
          │ Google Cloud │                 │  Geocoding   │  │ Action       │
          │ Vision API   │                 │  Services    │  │ Mailbox      │
          └──────────────┘                 └──────────────┘  └──────────────┘
                  │
                  ▼
          ┌──────────────┐
          │ Google Cloud │
          │   Storage    │
          └──────────────┘
```

## Application Structure

### MVC Pattern

**Models** (`app/models/`):
- Domain objects representing core entities
- Active Record for database interaction
- Concerns for shared behavior (Incompletable, Blockable, Bitfield)
- Validations and business logic

**Controllers** (`app/controllers/`):
- Handle HTTP requests and responses
- Authentication and authorization
- RESTful resource management
- API endpoints with Swagger documentation

**Views** (`app/views/`):
- Slim templates for HTML rendering
- Partials for reusable components
- Mailer templates
- JSON builders for API responses

### Service Layer (`app/lib/`)

Business logic extracted from models/controllers:
- `Annotator`: Google Cloud Vision API wrapper
- `ExifAnalyzer`: Photo metadata extraction
- `Vehicle`: License plate parsing and validation
- `PdfGenerator`: Report PDF generation
- `XmlGenerator`: WiNoWiG/OWI21 export
- `Colorizor`: Color matching
- `Token`: JWT token handling

### Background Jobs (`app/jobs/`)

**Core Jobs**:
- `AnalyzerJob`: Photo analysis pipeline (EXIF + Vision API)
- `PhotosDownloadJob`: Google Photos import
- `UserExportJob`: GDPR data export

**Scheduled Jobs** (`app/jobs/scheduled/`):
- `MaterializedViewUpdaterJob`: Update homepage stats (every 5 minutes)
- `AnalyticsJob`: Daily analytics processing (6am)
- `ActivationReminderJob`: Remind users to complete notices (Mon 8:10am)
- `UsageReminderJob`: Encourage bulk upload usage (Mon 8:20am)
- `ExpiringReminderJob`: Remind about old notices (Mon 8:30am)
- `GeocodingCleanupJob`: Fix geocoding issues (7am)
- `DataCleanupJob`: Remove old data (5am)
- `NoticeArchiverJob`: Archive old notices (4am)
- `ExportJob`: Weekly exports (Mon 3am)
- `StuckJob`: Monitor/kill stuck processes (hourly)

## Data Flow

### Notice Creation Flow

```
1. User uploads photos
   ↓
2. BulkUpload created (status: initial)
   ↓
3. Photos attached via Active Storage → Google Cloud Storage
   ↓
4. Notice created (status: analyzing)
   ↓
5. AnalyzerJob enqueued
   ├─→ EXIF extraction (GPS, timestamp)
   │   ├─→ Reverse geocoding (address)
   │   └─→ Proximity analysis (nearby violations)
   ├─→ Google Vision API call
   │   ├─→ License plate OCR
   │   ├─→ Brand recognition
   │   └─→ Color detection
   └─→ DataSets created with results
   ↓
6. Notice auto-filled based on user preferences
   ↓
7. Status changed to "open"
   ↓
8. User reviews and edits
   ↓
9. User submits (status: shared)
   ├─→ Email to district authority
   ├─→ PDF generation (optional)
   └─→ XML export (OWI21/WiNoWiG)
   ↓
10. Authority replies → Action Mailbox → Reply record
```

### Authentication Flow

```
1. User clicks OAuth provider button
   ↓
2. Redirect to provider (GitHub/Google/Twitter/etc.)
   ↓
3. Provider callback to /auth/:provider/callback
   ↓
4. SessionsController#create
   ├─→ Find or create User
   ├─→ Create Authorization record
   └─→ Set session[:user_id]
   ↓
5. User authenticated
```

**Email Authentication (Passwordless)**:
```
1. User enters email → /sessions/email
   ↓
2. JWT token generated (expires 1 hour)
   ↓
3. Magic link email sent
   ↓
4. User clicks link → /auth/validation/:token
   ↓
5. Token validated → User authenticated
```

## Database Design

### Key Tables

**notices**:
- Primary entity for violation reports
- 30+ columns including location, violation details, metadata
- PostGIS geography column (`lonlat`) for spatial queries
- Status enum: open, disabled, analyzing, shared
- Bitfield flags: vehicle_empty, hazard_lights, expired_tuv, etc.

**users**:
- Authentication and profile data
- Access levels: admin (42), community (1), user (0), disabled (-99)
- Bitfield preferences for auto-fill behavior
- Has signature attachment (Active Storage)

**districts**:
- German municipalities (3377+ records)
- Unique by ZIP code
- Email addresses for submitting violations
- License plate prefixes
- Config types for special integrations

**charges**:
- Traffic violation catalog (TBNR codes)
- 6-character codes with descriptions, fines, points
- Classification system (5 = parking violations)

**data_sets**:
- Polymorphic storage for analysis results
- Kinds: google_vision, exif, geocoder, proximity, car_ml
- JSON data column for flexible schema

**Materialized Views**:
- `homepages`: Aggregated public statistics
- `leaders`: Weekly leaderboard data

## Security Architecture

### Authentication
- Multi-provider OAuth via OmniAuth
- Passwordless email authentication with JWT
- Session-based authentication for web
- API token authentication (X-API-KEY header)

### Authorization
- Access level system (admin, community, user)
- Resource ownership checks
- Admin-only routes and features
- API rate limiting via Rack::Attack

### Data Protection
- Private file storage (Google Cloud Storage with signed URLs)
- Email validation and blocklist for disposable addresses
- MX record validation
- Personal email anonymization option
- GDPR compliance with data export

### Security Headers
- Content Security Policy
- CORS configured for API
- Secure session cookies
- CSRF protection

## Scalability Considerations

### Performance Optimization
- Materialized views for homepage statistics
- Database indexes on frequently queried columns
- Active Storage with CDN-ready GCS
- Background job processing for heavy operations
- Fragment caching for public pages

### Background Processing
- Sidekiq with Redis for job queue
- Separate queues for different priorities
- Job retry logic with exponential backoff
- Monitoring via Sidekiq web UI

### Database
- PostgreSQL connection pooling
- PostGIS for efficient spatial queries
- Prepared statements
- Index optimization for common queries

### External API Management
- Result caching in data_sets table
- Graceful degradation if APIs unavailable
- Rate limiting awareness
- Alternative providers (car-ml as Vision API backup)

## Configuration Management

### Environment Variables
All sensitive configuration stored in environment variables:
- OAuth credentials (GITHUB_CONSUMER_KEY, etc.)
- Google Cloud credentials (GOOGLE_CLOUD_PROJECT, GOOGLE_CLOUD_KEYFILE)
- API keys and tokens
- Service URLs

### Rails Environments
- **Development**: Local disk storage, letter_opener for emails
- **Test**: In-memory storage, fixture data
- **Production**: GCS storage, Sendgrid emails, error tracking

### Feature Flags
- User-level bitfields for autosuggest preferences
- District-level config types for integrations
- Environment-based feature toggles

## Deployment Architecture

### Docker Support
- `Dockerfile`: Application container
- `docker-compose.yml`: Multi-container setup (app, db, redis)
- Entrypoint scripts for initialization

### Platform Support
- Render.com (render.yaml configuration)
- Heroku (Procfile, buildpacks)
- Self-hosted Docker deployment

### Database Migrations
- Versioned migrations in `db/migrate/`
- Schema versioning
- PostGIS extension requirement
- Materialized view management

## Monitoring and Maintenance

### Logging
- Rails logger with configurable levels
- Sidekiq job logging
- Error tracking integration
- Verbose logging option (VERBOSE_LOGGING env var)

### Health Checks
- `StuckJob`: Monitors Sidekiq for stuck processes
- Database connection monitoring
- External service availability checks

### Scheduled Maintenance
- Data cleanup jobs remove old records
- Geocoding cleanup fixes address issues
- Archive job moves old notices
- Export job generates backups

## Code Organization Principles

### Concerns
- `Incompletable`: Allows saving invalid records (drafts)
- `Blockable`: Email validation and blocklist
- Shared across multiple models

### Bitfields
- Compact storage for boolean flags
- User preferences, notice flags, district settings
- Efficient querying with bitwise operations

### Polymorphic Associations
- `DataSet` belongs to any model (Notice, Photo)
- Flexible analysis result storage
- Multiple source types (Vision API, EXIF, geocoder)

### Service Objects
- Extract complex business logic from models
- Single responsibility principle
- Testable and reusable

This architecture enables weg-li to efficiently process thousands of violation reports while maintaining code quality and system reliability.
