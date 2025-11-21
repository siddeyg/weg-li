# weg-li Documentation

Welcome to the comprehensive documentation for **weg-li** - a civic engagement platform for reporting parking violations in Germany.

## 📖 Documentation Index

### Getting Started

**New to weg-li?** Start here:

1. **[Development Setup Guide](development-setup.md)**
   - System requirements and prerequisites
   - Local development setup (Linux, macOS, Docker)
   - Database configuration
   - Google Cloud integration setup
   - Testing environment
   - Troubleshooting guide

### Understanding the System

**Learn about the architecture and design:**

2. **[Architecture Documentation](architecture.md)**
   - Technology stack overview
   - System architecture and data flow
   - Application structure (MVC + services)
   - Background jobs and scheduling
   - Security architecture
   - Scalability considerations
   - Configuration management

### Data Models

**Understand the database and models:**

3. **[Models and Database Documentation](models-database.md)**
   - Core models (Notice, User, District, Charge, Brand)
   - Model relationships and associations
   - Database schema
   - Indexes and optimization
   - Concerns and mixins
   - Validation rules
   - Callbacks and lifecycle

### Integrations

**Learn about external services and APIs:**

4. **[API and Integrations Documentation](api-integrations.md)**
   - REST API reference
   - API authentication
   - Google Cloud Vision API integration
   - Google Cloud Storage setup
   - Geocoding services
   - Email integration (Sendgrid, Action Mailbox)
   - OAuth providers
   - Rate limiting and security

### Features

**Deep dive into functionality:**

5. **[Features and Workflows Documentation](features-workflows.md)**
   - Notice creation and management
   - Bulk upload workflow
   - Intelligent auto-fill system
   - Geographic features and mapping
   - License plate recognition
   - Violation catalog (TBNR system)
   - District management
   - PDF and XML export
   - User management and authentication
   - Reply tracking
   - Scheduled jobs and maintenance

---

## Quick Reference

### For Developers

**Setting up your environment:**
```bash
# Clone repository
git clone https://github.com/siddeyg/weg-li.git
cd weg-li

# Install dependencies
bundle install
yarn install

# Setup database
rails db:setup
rails db:seed
rake dev:data

# Start application
foreman start
```

**Common tasks:**
- Create admin user: `User.last.update(access: :admin)`
- Run tests: `bundle exec rspec`
- Rails console: `bundle exec rails console`
- Sidekiq UI: http://localhost:3000/sidekiq

### For API Users

**Authentication:**
```bash
curl -H "X-API-KEY: your-api-token" https://www.weg.li/api/notices
```

**Key endpoints:**
- `GET /api/notices` - List notices
- `POST /api/notices` - Create notice
- `POST /api/uploads` - Get upload URL
- `GET /api/districts` - List districts
- `GET /api/charges` - List violation codes

**Full API docs:** https://www.weg.li/api-docs

### For Contributors

**Code quality:**
```bash
# Linting
bundle exec rubocop

# Security audit
bundle exec brakeman

# Tests with coverage
COVERAGE=true bundle exec rspec
```

**Before submitting PR:**
1. All tests passing
2. RuboCop clean
3. No security issues
4. Meaningful commit messages
5. Updated documentation if needed

---

## Project Overview

### What is weg-li?

weg-li is a web application that empowers German citizens to report parking violations to local authorities. The platform combines:

- **Photo documentation** with automated analysis
- **AI-powered recognition** of license plates, brands, and colors
- **GPS-based location detection** from photo EXIF data
- **Direct integration** with German municipal authorities
- **Standardized violation codes** (TBNR system)

### Key Statistics

- **3377+ districts** covering all of Germany
- **Multiple authentication** options (OAuth + passwordless email)
- **Automatic analysis** via Google Cloud Vision API
- **Multiple export formats** (Email, PDF, WiNoWiG, OWI21)
- **Background processing** with Sidekiq for scalability

### Technology Highlights

- **Backend**: Ruby 3.2.9, Rails 7.2.2
- **Database**: PostgreSQL with PostGIS (spatial queries)
- **Frontend**: Webpacker, Stimulus, Turbo Rails
- **Cloud**: Google Cloud Platform (Vision API + Storage)
- **Jobs**: Redis + Sidekiq for background processing
- **Email**: Sendgrid + Action Mailbox

---

## Documentation Structure

```
docs/
├── index.md                    # This file - navigation and overview
├── architecture.md             # System design and architecture
├── models-database.md          # Data models and database schema
├── api-integrations.md         # APIs and external services
├── features-workflows.md       # Feature descriptions and workflows
└── development-setup.md        # Setup guide for developers
```

---

## Use Cases

### For Citizens

1. **Report parking violations**
   - Upload photos from smartphone
   - Automatic license plate recognition
   - One-click submission to authorities

2. **Bulk reporting**
   - Import photos from Google Photos
   - Create multiple reports efficiently
   - Track submissions and replies

3. **Location-based suggestions**
   - Common violations at your location
   - Pre-filled data from previous reports
   - Smart defaults based on context

### For Developers

1. **Build integrations**
   - RESTful API with Swagger docs
   - Direct upload to cloud storage
   - Webhook notifications

2. **Extend functionality**
   - Modular service architecture
   - Well-documented codebase
   - Active community

3. **Deploy your own instance**
   - Docker support
   - Clear setup instructions
   - Flexible configuration

### For Municipalities

1. **Receive violation reports**
   - Standardized format (WiNoWiG, OWI21)
   - Email with photo evidence
   - Structured data for processing

2. **Digital processing**
   - XML import into systems
   - Automated workflow integration
   - Reply tracking

---

## File Locations Reference

### Core Application Files

- **Models**: `app/models/`
- **Controllers**: `app/controllers/`
- **Views**: `app/views/`
- **Services**: `app/lib/`
- **Jobs**: `app/jobs/`
- **Mailers**: `app/mailers/`

### Configuration

- **Environment**: `.env` (local), environment variables (production)
- **Database**: `config/database.yml`
- **Routes**: `config/routes.rb`
- **Storage**: `config/storage.yml`
- **Initializers**: `config/initializers/`

### Data

- **Migrations**: `db/migrate/`
- **Schema**: `db/schema.rb`
- **Seeds**: `db/seeds.rb`
- **Reference data**: `config/data/`

### Tests

- **Specs**: `spec/`
- **Factories**: `spec/factories/`
- **Test config**: `spec/rails_helper.rb`

---

## External Resources

### Official Links

- **Website**: https://www.weg.li
- **GitHub**: https://github.com/weg-li/weg-li
- **API Docs**: https://www.weg.li/api-docs
- **Slack**: Join for discussions and support

### Related Projects

- **weg-li iOS**: Mobile app for iOS
- **weg-li Android**: Mobile app for Android
- **weg-li-imagine**: Image CDN service
- **car-ml**: Alternative ML service for license plate recognition

### Learning Resources

- **Rails Guides**: https://guides.rubyonrails.org
- **PostGIS Docs**: https://postgis.net/documentation
- **Google Cloud Vision**: https://cloud.google.com/vision/docs
- **Sidekiq**: https://github.com/mperham/sidekiq/wiki

---

## Contributing

We welcome contributions! Here's how to get started:

1. **Read the documentation** (you're here!)
2. **Setup development environment** ([Development Setup](development-setup.md))
3. **Understand the architecture** ([Architecture](architecture.md))
4. **Check open issues** on GitHub
5. **Join Slack** for discussions
6. **Submit a PR** with your improvements

### Contribution Guidelines

- Follow Ruby style guide (RuboCop)
- Write tests for new features
- Update documentation
- Keep commits focused and descriptive
- Be respectful and constructive

---

## Support

### Getting Help

- **Documentation**: You're reading it!
- **GitHub Issues**: Report bugs or request features
- **Slack**: Ask questions and discuss ideas
- **Email**: Contact project maintainers

### Reporting Issues

When reporting issues, please include:
- Clear description of the problem
- Steps to reproduce
- Expected vs actual behavior
- Environment details (OS, Ruby version, etc.)
- Relevant log output
- Screenshots if applicable

---

## License

**THE (extended) BEER-WARE LICENSE** (Revision 42.0815)

As long as you retain this notice you can do whatever you want with this stuff. If we meet some day, and you think this stuff is worth it, you can buy me some beers in return.

---

## Changelog

Major updates and version history tracked in GitHub releases.

---

**Last Updated**: November 2024

**Documentation Version**: 1.0

---

## Next Steps

Choose your path:

- **👨‍💻 Developer**: Start with [Development Setup](development-setup.md)
- **🏗️ Architect**: Read [Architecture Documentation](architecture.md)
- **📊 Database**: Explore [Models and Database](models-database.md)
- **🔌 API User**: Check [API and Integrations](api-integrations.md)
- **✨ Feature Tour**: Browse [Features and Workflows](features-workflows.md)

---

Happy reporting! 🚲🚶‍♂️✊
