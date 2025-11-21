# Development Setup Guide

## Prerequisites

### Required Software

- **Ruby**: 3.2.9
- **Rails**: 7.2.2.2
- **PostgreSQL**: 12+ with PostGIS extension
- **Node.js**: 16+ (for Webpacker)
- **Yarn**: 1.22+
- **Redis**: 6+ (for Sidekiq)
- **ImageMagick**: 7+ (for image processing)

### Optional Tools

- **Docker**: For containerized development
- **rbenv** or **rvm**: Ruby version management
- **foreman**: Process management (Procfile support)

---

## Local Development Setup

### Linux (Ubuntu/Debian)

#### 1. Install System Dependencies

```bash
# Update package list
sudo apt update

# Install Ruby dependencies
sudo apt install -y git curl libssl-dev libreadline-dev zlib1g-dev \
  autoconf bison build-essential libyaml-dev libreadline-dev \
  libncurses5-dev libffi-dev libgdbm-dev

# Install PostgreSQL with PostGIS
sudo apt install -y postgresql postgresql-contrib postgis

# Install Node.js and Yarn
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
npm install -g yarn

# Install Redis
sudo apt install -y redis-server

# Install ImageMagick
sudo apt install -y imagemagick libmagickwand-dev
```

#### 2. Install Ruby

Using rbenv:
```bash
# Install rbenv
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/main/bin/rbenv-installer | bash

# Add to ~/.bashrc
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc

# Install Ruby 3.2.9
rbenv install 3.2.9
rbenv global 3.2.9

# Verify
ruby -v  # Should show 3.2.9
```

#### 3. Setup PostgreSQL

```bash
# Start PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create PostgreSQL user
sudo -u postgres createuser -s $USER

# Enable PostGIS extension (will be done via Rails migration)
```

#### 4. Clone Repository

```bash
git clone https://github.com/siddeyg/weg-li.git
cd weg-li
```

#### 5. Install Dependencies

```bash
# Install Ruby gems
bundle install

# Install JavaScript packages
yarn install
```

#### 6. Configure Environment

```bash
# Copy example environment file
cp .env-example .env

# Edit .env file
nano .env
```

**Minimal Configuration** (for local development without external services):
```bash
# Database
DATABASE_URL=postgresql://localhost/weg_li_development

# Required for Rails
SECRET_KEY_BASE=your-secret-key-here-run-rails-secret-to-generate

# Optional: OAuth providers (not required for basic functionality)
# GITHUB_CONSUMER_KEY=
# GITHUB_CONSUMER_SECRET=
# GOOGLE_CONSUMER_KEY=
# GOOGLE_CONSUMER_SECRET=

# Optional: Google Cloud (for full functionality)
# GOOGLE_CLOUD_PROJECT=
# GOOGLE_CLOUD_KEYFILE=
```

Generate `SECRET_KEY_BASE`:
```bash
bundle exec rails secret
# Copy output to .env
```

#### 7. Setup Database

```bash
# Create database
bundle exec rails db:create

# Run migrations
bundle exec rails db:migrate

# Load seed data (charges, brands, etc.)
bundle exec rails db:seed

# Generate test data (districts, notices, users)
bundle exec rake dev:data
```

#### 8. Start Services

**Option A: Using individual commands**
```bash
# Terminal 1: Rails server
bundle exec rails server

# Terminal 2: Sidekiq (background jobs)
bundle exec sidekiq

# Terminal 3: Webpack dev server (hot reloading)
bundle exec bin/webpack-dev-server

# Terminal 4: Redis (if not running as service)
redis-server
```

**Option B: Using Foreman (recommended)**
```bash
# Install foreman
gem install foreman

# Start all services
foreman start
```

This will start:
- Rails server on http://localhost:3000
- Sidekiq for background jobs
- Webpack dev server for assets

#### 9. Access Application

Open browser: http://localhost:3000

**Login**:
- Click "Email" authentication
- Enter any email address
- Check terminal/console for magic link
- Click link to authenticate

**Create Admin User**:
```bash
# Open Rails console
bundle exec rails console

# Make your user an admin
user = User.last  # Or find by email: User.find_by(email: 'your@email.com')
user.update(access: :admin)
```

Access admin interface: http://localhost:3000/admin

---

### macOS Setup

#### 1. Install Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

#### 2. Install Dependencies

```bash
# Install Ruby environment manager
brew install rbenv

# Install PostgreSQL with PostGIS
brew install postgresql postgis

# Install Node.js and Yarn
brew install node yarn

# Install Redis
brew install redis

# Install ImageMagick
brew install imagemagick

# Start services
brew services start postgresql
brew services start redis
```

#### 3. Install Ruby

```bash
# Install Ruby 3.2.9
rbenv install 3.2.9
rbenv global 3.2.9

# Add to ~/.zshrc (or ~/.bashrc)
echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc
source ~/.zshrc

# Verify
ruby -v
```

#### 4. Create PostgreSQL User

```bash
# Create user with same name as your macOS user
createuser -s $(whoami)

# Or create 'postgres' superuser
createuser -s postgres
```

#### 5. Follow Steps 4-9 from Linux Setup

Same as Linux: clone repo, install dependencies, configure environment, setup database, start services.

---

## Docker Development Setup

### Prerequisites

- Docker Desktop (Mac/Windows) or Docker Engine (Linux)
- Docker Compose

### Setup

#### 1. Clone Repository

```bash
git clone https://github.com/siddeyg/weg-li.git
cd weg-li
```

#### 2. Configure Environment

```bash
cp .env-example .env
# Edit .env if needed (defaults work for Docker setup)
```

#### 3. Build and Start Containers

```bash
# Build Docker image
docker-compose build

# Start all services (app, db, redis)
docker-compose up
```

Services started:
- **app**: Rails application on port 3000
- **db**: PostgreSQL with PostGIS
- **redis**: Redis for Sidekiq

#### 4. Setup Database (First Time Only)

```bash
# Create database
docker-compose exec app bundle exec rails db:create

# Run migrations
docker-compose exec app bundle exec rails db:migrate

# Load seed data
docker-compose exec app bundle exec rails db:seed

# Generate test data
docker-compose exec app bundle exec rake dev:data
```

#### 5. Access Application

Open browser: http://localhost:3000

#### Common Docker Commands

```bash
# Start services
docker-compose up

# Start in background
docker-compose up -d

# Stop services
docker-compose down

# View logs
docker-compose logs -f app

# Open Rails console
docker-compose exec app bundle exec rails console

# Run migrations
docker-compose exec app bundle exec rails db:migrate

# Run tests
docker-compose exec app bundle exec rspec

# Restart app after code changes
docker-compose restart app
```

---

## Importing Production Data

### Districts

Import real district data from production:

```bash
# Open Rails console
bundle exec rails console

# Download and import districts
districts = JSON.parse(URI.open('https://www.weg.li/districts.json').read)
District.insert_all!(
  districts.map { |d|
    d.except('personal_email').merge({'flags': d['personal_email'] ? 1 : 0})
  }
)
```

### Charges (TBNR Catalog)

Charges are included in seed data:
```bash
bundle exec rails db:seed
```

Or update from production:
```bash
# Rails console
charges = JSON.parse(URI.open('https://www.weg.li/charges.json').read)
Charge.insert_all!(charges)
```

---

## Google Cloud Setup (Optional)

For full functionality (license plate recognition, cloud storage):

### 1. Create Google Cloud Project

1. Visit https://console.cloud.google.com
2. Create new project: "weg-li-dev"
3. Enable APIs:
   - Cloud Vision API
   - Cloud Storage API

### 2. Create Service Account

1. Navigate to: IAM & Admin → Service Accounts
2. Create service account: "weg-li-dev"
3. Grant roles:
   - Storage Object Admin
   - Cloud Vision API User
4. Create key (JSON)
5. Download key file to: `config/gcloud.json`

### 3. Create Storage Bucket

```bash
# Install gcloud CLI
curl https://sdk.cloud.google.com | bash

# Authenticate
gcloud auth login

# Set project
gcloud config set project YOUR_PROJECT_ID

# Create bucket
gsutil mb -l EU gs://weg-li-development
```

### 4. Configure Environment

Add to `.env`:
```bash
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_KEYFILE=config/gcloud.json
```

Or use credentials directly:
```bash
GOOGLE_APPLICATION_CREDENTIALS='{"type":"service_account",...}'
```

### 5. Test Integration

```bash
# Rails console
bundle exec rails console

# Test Vision API
annotator = Annotator.new
result = annotator.annotate_object('test-image.jpg')

# Test storage
blob = ActiveStorage::Blob.create_after_upload!(
  io: File.open('test.jpg'),
  filename: 'test.jpg'
)
blob.url  # Should return GCS URL
```

---

## Running Tests

### Setup Test Database

```bash
# Create test database
RAILS_ENV=test bundle exec rails db:create

# Run migrations
RAILS_ENV=test bundle exec rails db:migrate
```

### Run Tests

```bash
# All tests
bundle exec rspec

# Specific file
bundle exec rspec spec/models/notice_spec.rb

# Specific test
bundle exec rspec spec/models/notice_spec.rb:42

# With coverage report
COVERAGE=true bundle exec rspec
```

### Test Configuration

**File**: `spec/rails_helper.rb`

**Features**:
- FactoryBot for test data
- Database Cleaner for clean state
- Shoulda Matchers for readable assertions
- SimpleCov for coverage reports
- VCR for HTTP request mocking

---

## Development Tools

### Rails Console

```bash
# Start console
bundle exec rails console

# Or with Docker
docker-compose exec app bundle exec rails console
```

**Useful commands**:
```ruby
# Reload code
reload!

# Create test user
user = User.create!(email: 'test@example.com', nickname: 'tester')

# Create test notice
notice = Notice.create!(
  user: user,
  registration: 'B-AB 1234',
  street: 'Hauptstrasse',
  city: 'Berlin',
  zip: '10115',
  charge: '112454'
)

# Trigger analysis
AnalyzerJob.perform_now(notice.id)
```

### Sidekiq Web UI

Access at: http://localhost:3000/sidekiq (admin only)

**Features**:
- Monitor job queues
- View failed jobs
- Retry failed jobs
- Real-time stats

### Database Console

```bash
# PostgreSQL console
psql weg_li_development

# Or with Docker
docker-compose exec db psql -U postgres weg_li_development
```

**Useful queries**:
```sql
-- Count notices by status
SELECT status, COUNT(*) FROM notices GROUP BY status;

-- Recent notices with location
SELECT registration, street, city, created_at
FROM notices
ORDER BY created_at DESC
LIMIT 10;

-- Spatial query: notices within 50m
SELECT * FROM notices
WHERE ST_DWithin(
  lonlat,
  ST_SetSRID(ST_MakePoint(13.404954, 52.520008), 4326)::geography,
  50
);
```

### Logs

```bash
# Development log
tail -f log/development.log

# Sidekiq log
tail -f log/sidekiq.log

# Or with Docker
docker-compose logs -f app
```

---

## Common Development Tasks

### Reset Database

```bash
# Drop, create, migrate, seed
bundle exec rails db:reset

# Or step by step
bundle exec rails db:drop
bundle exec rails db:create
bundle exec rails db:migrate
bundle exec rails db:seed
```

### Generate Test Data

```bash
# Generate districts, users, notices
bundle exec rake dev:data

# Customize in lib/tasks/dev.rake
```

### Update Dependencies

```bash
# Ruby gems
bundle update

# JavaScript packages
yarn upgrade

# Check for outdated packages
bundle outdated
yarn outdated
```

### Database Migrations

```bash
# Create migration
bundle exec rails generate migration AddFieldToModel field:type

# Run migrations
bundle exec rails db:migrate

# Rollback last migration
bundle exec rails db:rollback

# Check migration status
bundle exec rails db:migrate:status
```

### Code Quality

```bash
# Run RuboCop (linter)
bundle exec rubocop

# Auto-fix issues
bundle exec rubocop -a

# Run security audit
bundle exec brakeman

# Check for vulnerable dependencies
bundle exec bundle-audit check --update
```

---

## Troubleshooting

### PostgreSQL Connection Issues

**Error**: `could not connect to server`

**Solution**:
```bash
# Check if PostgreSQL is running
sudo systemctl status postgresql  # Linux
brew services list  # macOS

# Start PostgreSQL
sudo systemctl start postgresql  # Linux
brew services start postgresql  # macOS
```

### Redis Connection Issues

**Error**: `Redis::CannotConnectError`

**Solution**:
```bash
# Start Redis
sudo systemctl start redis  # Linux
brew services start redis  # macOS

# Or run in foreground
redis-server
```

### PostGIS Extension Error

**Error**: `PG::UndefinedFile: ERROR: could not open extension control file`

**Solution**:
```bash
# Install PostGIS
sudo apt install postgis  # Linux
brew install postgis  # macOS

# Enable in database
psql weg_li_development
CREATE EXTENSION postgis;
```

### ImageMagick Errors

**Error**: `MiniMagick::Invalid`

**Solution**:
```bash
# Install ImageMagick
sudo apt install imagemagick libmagickwand-dev  # Linux
brew install imagemagick  # macOS

# Check installation
which convert
convert --version
```

### Webpacker Compilation Issues

**Error**: `Webpacker can't find application.js`

**Solution**:
```bash
# Reinstall node modules
rm -rf node_modules yarn.lock
yarn install

# Recompile
bundle exec rails webpacker:compile

# Or run dev server
bundle exec bin/webpack-dev-server
```

### Permission Errors (Linux)

**Error**: `Permission denied`

**Solution**:
```bash
# Fix file permissions
chmod +x bin/*

# Fix bundle permissions
bundle install --path vendor/bundle
```

---

## IDE Setup

### VS Code

**Recommended Extensions**:
- Ruby (Peng Lv)
- Ruby Solargraph
- Rails (Hridoy)
- ERB Helper Tags
- PostgreSQL (Chris Kolkman)
- Docker

**Settings** (`.vscode/settings.json`):
```json
{
  "ruby.intellisense": "rubyLocate",
  "ruby.format": "rubocop",
  "ruby.lint": {
    "rubocop": true
  },
  "editor.formatOnSave": true,
  "[ruby]": {
    "editor.defaultFormatter": "misogi.ruby-rubocop"
  }
}
```

### RubyMine

- Open project directory
- Configure Ruby SDK: Settings → Languages & Frameworks → Ruby SDK
- Enable Rails support: Settings → Languages & Frameworks → Ruby on Rails
- Configure database: Database tool window → + → PostgreSQL

---

## Next Steps

1. ✅ Setup development environment
2. ✅ Run application locally
3. 📖 Read [Architecture Documentation](architecture.md)
4. 📖 Read [Models and Database](models-database.md)
5. 📖 Read [API and Integrations](api-integrations.md)
6. 📖 Read [Features and Workflows](features-workflows.md)
7. 🔨 Start contributing!

---

## Getting Help

- **GitHub Issues**: https://github.com/weg-li/weg-li/issues
- **Slack**: https://join.slack.com/t/weg-li/shared_invite/...
- **Documentation**: This docs folder

Happy coding! 🚀
