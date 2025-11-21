# Claude AI Assistant Guide for weg-li

This document provides guidance for AI assistants (Claude, GPT, etc.) working with the weg-li codebase.

## Project Overview

**weg-li** is a Ruby on Rails 7.2 civic engagement platform for reporting parking violations in Germany.

**Core Purpose**: Enable citizens to document parking violations with photos, automatically analyze them using AI, and submit reports to local authorities.

## Quick Context

### Technology Stack
- **Backend**: Ruby 3.2.9, Rails 7.2.2.2
- **Database**: PostgreSQL 12+ with PostGIS
- **Jobs**: Redis + Sidekiq
- **Storage**: Google Cloud Storage
- **AI**: Google Cloud Vision API (license plate OCR, brand recognition)
- **Frontend**: Webpacker, Stimulus, Turbo Rails

### Key Models
- **Notice**: Parking violation report (main entity)
- **User**: Account with OAuth authentication
- **District**: German municipality (3377+ records)
- **Charge**: Violation code from TBNR catalog
- **Brand**: Vehicle manufacturer
- **DataSet**: Analysis results (polymorphic)

### Critical Files
- `app/models/notice.rb` (439 lines) - Core violation model
- `app/jobs/analyzer_job.rb` (131 lines) - Photo analysis pipeline
- `app/lib/annotator.rb` (98 lines) - Google Vision API wrapper
- `app/lib/vehicle.rb` (101 lines) - License plate parsing
- `app/controllers/notices_controller.rb` (384 lines) - Main controller

## Common Tasks

### Understanding Code

When asked about functionality:
1. Check `docs/` directory first (comprehensive documentation)
2. Read model files for business logic
3. Check controllers for user-facing features
4. Review jobs for background processing

**Example Questions**:
- "How does license plate recognition work?" → Read `app/lib/vehicle.rb` and `docs/features-workflows.md#license-plate-recognition`
- "What are the notice statuses?" → Check `app/models/notice.rb` enum
- "How is geocoding handled?" → See `app/jobs/analyzer_job.rb` and `docs/api-integrations.md#geocoding-integration`

### Making Changes

**Before modifying code**:
1. Understand the current implementation
2. Check for related tests in `spec/`
3. Consider impact on background jobs
4. Review concerns for shared behavior

**Testing changes**:
```bash
# Run specific test
bundle exec rspec spec/models/notice_spec.rb

# Run all tests
bundle exec rspec

# Check code style
bundle exec rubocop
```

### Adding Features

**Typical flow**:
1. Add model changes (migrations if needed)
2. Update controllers/routes
3. Add views (Slim templates)
4. Write tests (RSpec + FactoryBot)
5. Update documentation

**Example - Adding a new notice field**:
```bash
# 1. Migration
rails g migration AddFieldToNotices field_name:type

# 2. Update model
# app/models/notice.rb - add validation, scope, method

# 3. Update controller
# app/controllers/notices_controller.rb - permit parameter

# 4. Update view
# app/views/notices/_form.html.slim - add form field

# 5. Test
# spec/models/notice_spec.rb - add test cases
```

## Code Conventions

### Ruby Style
- Follow RuboCop configuration (`.rubocop.yml`)
- 2-space indentation
- Max line length: 180 characters
- Use descriptive method names

### Rails Patterns
- **Concerns**: Shared behavior (Incompletable, Blockable, Bitfield)
- **Services**: Complex business logic in `app/lib/`
- **Jobs**: Background processing in `app/jobs/`
- **Scopes**: Query methods in models
- **Enums**: Status fields with integer backing

### Database
- Use migrations for schema changes
- Add indexes for query optimization
- Use PostGIS for spatial queries
- Maintain referential integrity

### Testing
- Use FactoryBot for test data
- Write request specs for controllers
- Write model specs for business logic
- Mock external APIs (VCR for HTTP)

## Important Concepts

### Incompletable Concern

Allows saving invalid records (drafts):
```ruby
notice.save_incomplete  # Bypasses validation
notice.incomplete?      # Check if valid
```

Used for draft notices that users are still editing.

### Bitfields

Efficient boolean storage:
```ruby
# In model
bitfield :flags, vehicle_empty: 1, hazard_lights: 2

# Usage
notice.vehicle_empty?
notice.vehicle_empty = true
```

### Analysis Pipeline

1. User uploads photos → Notice created (status: analyzing)
2. `AnalyzerJob` enqueued
3. EXIF extraction (GPS, timestamp)
4. Geocoding (address lookup)
5. Proximity analysis (nearby violations)
6. Google Vision API (OCR, brand, color)
7. DataSets created with results
8. Notice auto-filled, status → open

### German Specifics

- **TBNR**: 6-digit violation codes (Tatbestandsnummer)
- **Districts**: Identified by ZIP code
- **License plates**: Format PREFIX-LETTERS DIGITS (e.g., "B-AB 1234")
- **WiNoWiG/OWI21**: XML export formats for authorities

## External Dependencies

### Google Cloud Vision API
- **Purpose**: License plate OCR, brand recognition, color detection
- **Rate limit**: 1,800 requests/minute
- **Cost**: $1.50 per 1,000 requests (after free tier)
- **Alternative**: car-ml service (ENV['CAR_ML_URL'])

### Google Cloud Storage
- **Buckets**: weg-li-production, weg-li-development
- **Access**: Private with signed URLs (7-day expiration)
- **Direct upload**: Via signed URLs from API

### Geocoding
- **Primary**: Nominatim (OpenStreetMap)
- **Alternative**: OpenCage
- **Rate limit**: 1 req/sec (Nominatim)

## Development Workflow

### Starting Development

```bash
# Clone and setup
git clone https://github.com/siddeyg/weg-li.git
cd weg-li
bundle install
yarn install

# Configure
cp .env-example .env
# Edit .env with secrets

# Database
rails db:setup
rails db:seed
rake dev:data

# Start services
foreman start
```

### Making Changes

```bash
# Create branch
git checkout -b feature/description

# Make changes
# ... edit files ...

# Test
bundle exec rspec
bundle exec rubocop

# Commit
git add .
git commit -m "Description of changes"

# Push
git push origin feature/description
```

### Database Changes

```bash
# Create migration
rails g migration MigrationName

# Edit migration file
# db/migrate/TIMESTAMP_migration_name.rb

# Run migration
rails db:migrate

# Test rollback
rails db:rollback
rails db:migrate
```

## Common Pitfalls

### 1. PostGIS Queries

**Wrong**:
```ruby
Notice.where("latitude = ? AND longitude = ?", lat, lng)
```

**Correct**:
```ruby
Notice.where("ST_DWithin(lonlat, ST_SetSRID(ST_MakePoint(?, ?), 4326)::geography, 50)", lng, lat)
```

### 2. Active Storage URLs

**Wrong**:
```ruby
photo.service_url  # Expires, not for background jobs
```

**Correct**:
```ruby
photo.url  # Generates new signed URL
```

### 3. Background Jobs

**Wrong**:
```ruby
AnalyzerJob.perform_now(notice)  # Blocks request
```

**Correct**:
```ruby
AnalyzerJob.perform_later(notice.id)  # Async via Sidekiq
```

### 4. German Characters

**Handle umlauts properly**:
```ruby
# License plates: Ä→AE, Ö→OE, Ü→UE
Vehicle.normalize("MÜ-AB 1234")  # => "MUE-AB 1234"
```

## Security Considerations

- **API keys**: Store in environment variables, never commit
- **User data**: GDPR compliance (export functionality)
- **Email validation**: Blocklist for disposable emails
- **Rate limiting**: Rack::Attack configuration
- **File uploads**: Validate file types and sizes
- **SQL injection**: Use parameterized queries
- **XSS**: Slim templates auto-escape by default

## Performance Optimization

- **Materialized views**: Homepage statistics (refreshed every 5 min)
- **Indexes**: Key columns (user_id, status, zip, registration)
- **Background jobs**: Heavy processing (analysis, geocoding)
- **Caching**: DataSets store API results
- **Pagination**: `kaminari` gem for large result sets

## Documentation

**Comprehensive docs in `docs/` directory**:
- `index.md` - Navigation and overview
- `architecture.md` - System design
- `models-database.md` - Data models
- `api-integrations.md` - External services
- `features-workflows.md` - Feature descriptions
- `development-setup.md` - Setup guide

**Always refer to docs when**:
- Understanding a feature
- Planning changes
- Explaining functionality
- Onboarding new developers

## Useful Commands

```bash
# Rails console
bundle exec rails console

# Database console
psql weg_li_development

# Sidekiq web UI
open http://localhost:3000/sidekiq

# Check dependencies
bundle outdated
yarn outdated

# Code quality
bundle exec rubocop -a
bundle exec brakeman

# Reset database
bundle exec rails db:reset
```

## Getting Help

When stuck:
1. Read `docs/` directory
2. Search codebase for similar functionality
3. Check test files for usage examples
4. Review GitHub issues
5. Ask in Slack (link in docs)

## Contributing Guidelines

- Follow existing code style
- Write tests for new features
- Update documentation
- Run RuboCop before committing
- Write clear commit messages
- Keep PRs focused and small

---

**For more details**: See `docs/index.md` for comprehensive documentation.

**Project Repository**: https://github.com/siddeyg/weg-li

**License**: THE (extended) BEER-WARE LICENSE (Revision 42.0815)

---

**Note for AI Assistants**: This file is specifically for you! When working with this codebase:
- Always check the `docs/` directory first
- Understand the German context (TBNR codes, districts, license plates)
- Consider background job implications
- Test changes thoroughly
- Follow Rails and Ruby best practices
- Be mindful of external API costs (Google Cloud Vision)

Happy coding! 🚲✊
