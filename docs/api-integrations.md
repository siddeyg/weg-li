# API and Integrations Documentation

## REST API

### Overview

weg-li provides a comprehensive REST API for programmatic access to core functionality. The API follows RESTful conventions and includes Swagger/OpenAPI documentation.

**Base URL**: `https://www.weg.li/api`

**Documentation**: `https://www.weg.li/api-docs`

### Authentication

#### API Key Authentication

All API requests require authentication via the `X-API-KEY` header.

```bash
curl -H "X-API-KEY: your-api-token-here" https://www.weg.li/api/notices
```

**Getting Your API Token**:
1. Log in to your account
2. Visit your profile page
3. Find your API token in the API section
4. Click "Generate" if no token exists

**Token Format**: 64-character hexadecimal string

**Token Security**:
- Store securely (never commit to version control)
- Regenerate if compromised
- One token per user
- No expiration (manual regeneration only)

### API Endpoints

#### Notices

**List Notices**
```
GET /api/notices
```

Query Parameters:
- `page` (integer): Page number (default: 1)
- `per_page` (integer): Results per page (default: 25, max: 100)
- `status` (string): Filter by status (open, shared, analyzing, disabled)

Response:
```json
{
  "notices": [
    {
      "id": 12345,
      "token": "abc123xyz",
      "registration": "B-AB 1234",
      "brand": "Mercedes-Benz",
      "color": "silver",
      "street": "Hauptstrasse",
      "city": "Berlin",
      "zip": "10115",
      "latitude": 52.520008,
      "longitude": 13.404954,
      "charge": "112454",
      "tbnr": "112454",
      "date": "2024-11-21",
      "start_date": "2024-11-21T14:30:00Z",
      "end_date": "2024-11-21T15:30:00Z",
      "note": "Blocking bike lane",
      "status": "open",
      "photos_count": 3,
      "created_at": "2024-11-21T14:45:00Z",
      "sent_at": null
    }
  ],
  "meta": {
    "current_page": 1,
    "total_pages": 10,
    "total_count": 245
  }
}
```

**Get Notice**
```
GET /api/notices/:id
```

**Create Notice**
```
POST /api/notices
```

Request Body:
```json
{
  "notice": {
    "registration": "B-AB 1234",
    "brand": "Mercedes",
    "color": "black",
    "street": "Hauptstrasse 1",
    "city": "Berlin",
    "zip": "10115",
    "charge": "112454",
    "start_date": "2024-11-21T14:30:00",
    "note": "Blocking bike lane",
    "photo_ids": ["signed_id_1", "signed_id_2"]
  }
}
```

**Update Notice**
```
PATCH /api/notices/:id
```

**Delete Notice**
```
DELETE /api/notices/:id
```

**Mail Notice**
```
POST /api/notices/:id/mail
```

Sends the notice to the district authority.

Request Body:
```json
{
  "send_via": "email",
  "recipient_email": "authority@example.com"
}
```

#### Uploads

**Create Signed Upload**
```
POST /api/uploads
```

Request Body:
```json
{
  "upload": {
    "filename": "violation.jpg",
    "byte_size": 2048576,
    "checksum": "abc123...",
    "content_type": "image/jpeg"
  }
}
```

Response:
```json
{
  "signed_id": "eyJfcmFpbHMi...",
  "direct_upload": {
    "url": "https://storage.googleapis.com/weg-li-production/...",
    "headers": {
      "Content-Type": "image/jpeg",
      "Content-MD5": "abc123..."
    }
  }
}
```

**Upload Flow**:
1. POST /api/uploads with file metadata → get signed_id and GCS URL
2. PUT to GCS URL with file data
3. POST /api/notices with photo_ids containing signed_id

#### Districts

**List Districts**
```
GET /api/districts
```

Query Parameters:
- `q` (string): Search by name
- `zip` (string): Filter by ZIP code
- `state` (string): Filter by German state

Response:
```json
{
  "districts": [
    {
      "name": "Berlin Mitte",
      "zip": "10115",
      "email": "anzeige@berlin.de",
      "prefixes": ["B"],
      "state": "Berlin",
      "latitude": 52.520008,
      "longitude": 13.404954
    }
  ]
}
```

#### Charges

**List Charges**
```
GET /api/charges
```

Query Parameters:
- `classification` (integer): Filter by category (5 = parking)
- `favorites` (boolean): Only show common charges

Response:
```json
{
  "charges": [
    {
      "tbnr": "112454",
      "description": "Parken im eingeschränkten Halteverbot",
      "fine": "25.00",
      "classification": 5
    }
  ]
}
```

#### Brands

**List Brands**
```
GET /api/brands
```

Query Parameters:
- `q` (string): Search by name
- `kind` (string): Filter by vehicle type (car, truck, bike)

---

## Google Cloud Integration

### Google Cloud Vision API

**Purpose**: Automated license plate recognition, brand detection, and color identification from photos.

#### Setup

1. **Create Google Cloud Project**:
   - Visit https://console.cloud.google.com
   - Create new project or select existing
   - Enable Cloud Vision API
   - Enable Cloud Storage API

2. **Create Service Account**:
   - Navigate to IAM & Admin → Service Accounts
   - Create service account with roles:
     - Storage Object Admin
     - Cloud Vision API User
   - Generate JSON key file

3. **Configure Environment**:
```bash
export GOOGLE_CLOUD_PROJECT=your-project-id
export GOOGLE_CLOUD_KEYFILE=/path/to/service-account-key.json
```

#### API Usage

**File**: `app/lib/annotator.rb`

**Features Used**:
- `DOCUMENT_TEXT_DETECTION`: OCR for license plates
- `IMAGE_PROPERTIES`: Dominant color extraction

**Request Flow**:
```ruby
# In AnalyzerJob
annotator = Annotator.new
result = annotator.annotate_object(photo.key)

# API call to Vision API
# gs://weg-li-production/abc123.jpg
```

**API Request Example**:
```json
{
  "requests": [{
    "image": {
      "source": {
        "imageUri": "gs://weg-li-production/abc123.jpg"
      }
    },
    "features": [
      {"type": "DOCUMENT_TEXT_DETECTION"},
      {"type": "IMAGE_PROPERTIES"}
    ],
    "imageContext": {
      "languageHints": ["de"]
    }
  }]
}
```

**Response Processing**:
```ruby
# Extract license plates
text_annotations = response['text_annotations']
plates = text_annotations.map { |annotation|
  Vehicle.plate?(annotation['description'])
}.compact

# Extract colors
colors = response['image_properties']['dominant_colors']['colors']
dominant_color = Colorizor.closest_color(colors.first)

# Extract brands
brands = text_annotations.map { |annotation|
  Brand.fuzzy_search(annotation['description'])
}.compact.first
```

#### Rate Limits and Costs

**Pricing** (as of 2024):
- First 1,000 requests/month: Free
- Additional requests: $1.50 per 1,000

**Rate Limits**:
- Default: 1,800 requests/minute
- Burst: 600 requests/minute

**Optimization Strategies**:
- Results cached in DataSet records
- One-time analysis per photo
- Manual re-analysis available if needed
- Alternative car-ml service as fallback

---

### Google Cloud Storage

**Purpose**: Photo storage with direct upload capability.

#### Configuration

**File**: `config/storage.yml`

```yaml
google_production:
  service: GCS
  project: <%= ENV['GOOGLE_CLOUD_PROJECT'] %>
  credentials: <%= ENV['GOOGLE_CLOUD_KEYFILE'] %>
  bucket: weg-li-production

google_development:
  service: GCS
  project: <%= ENV['GOOGLE_CLOUD_PROJECT'] %>
  credentials: <%= ENV['GOOGLE_CLOUD_KEYFILE'] %>
  bucket: weg-li-development
```

#### Bucket Structure

```
weg-li-production/
  ├── abc123xyz.jpg          # Notice photos
  ├── def456uvw.png
  └── [secure_random_base36].[ext]
```

**Naming Convention**:
- `SecureRandom.base36(28)` + file extension
- No user-controlled paths (security)
- Unique across all uploads

#### Direct Upload Flow

1. **Client requests upload URL**:
```javascript
POST /api/uploads
{
  "filename": "photo.jpg",
  "byte_size": 2048576,
  "checksum": "md5-base64",
  "content_type": "image/jpeg"
}
```

2. **Server generates signed URL**:
```ruby
blob = ActiveStorage::Blob.create_before_direct_upload!(params)
signed_id = blob.signed_id
upload_url = blob.service_url_for_direct_upload
```

3. **Client uploads directly to GCS**:
```javascript
PUT https://storage.googleapis.com/weg-li-production/abc123.jpg
Headers:
  Content-Type: image/jpeg
  Content-MD5: abc123...
Body: [binary photo data]
```

4. **Client attaches to notice**:
```javascript
POST /api/notices
{
  "photo_ids": ["signed_id_from_step1"]
}
```

#### Access Control

**Private by Default**:
- All objects private
- Signed URLs for temporary access (1 week expiration)
- No public listing

**URL Generation**:
```ruby
photo.url  # Returns signed URL valid for 7 days
```

#### Storage Management

**Lifecycle Policies**:
- Photos deleted when notice is deleted (ActiveStorage cascade)
- Orphaned blobs cleaned up via `rails storage:purge_unattached`

**Backup Strategy**:
- GCS versioning enabled (optional)
- Cross-region replication (optional)
- Regular exports via ExportJob

---

## Geocoding Integration

**Purpose**: Convert GPS coordinates to addresses and vice versa.

**Gem**: `geocoder`

**File**: `config/initializers/geocoder.rb`

### Providers

**Primary: Nominatim** (OpenStreetMap):
```ruby
Geocoder.configure(
  lookup: :nominatim,
  timeout: 5,
  units: :km,
  language: :de
)
```

**Alternative: OpenCage**:
```ruby
Geocoder.configure(
  lookup: :opencage,
  api_key: ENV['OPENCAGE_API_KEY'],
  language: :de
)
```

### Usage

**Reverse Geocoding** (Coordinates → Address):
```ruby
result = Geocoder.search([52.520008, 13.404954]).first
address = result.address  # "Hauptstrasse 1, 10115 Berlin"
```

**Forward Geocoding** (Address → Coordinates):
```ruby
result = Geocoder.search("Hauptstrasse 1, Berlin").first
coords = [result.latitude, result.longitude]
```

**In Models**:
```ruby
class Notice < ApplicationRecord
  geocoded_by :location
  reverse_geocoded_by :latitude, :longitude, address: :location

  after_validation :geocode, if: ->(obj) { obj.location_changed? }
  after_validation :reverse_geocode, if: ->(obj) { obj.latitude_changed? }
end
```

### Rate Limits

**Nominatim**:
- 1 request per second
- Must include User-Agent header
- Cache results to minimize requests

**OpenCage**:
- Free tier: 2,500 requests/day
- Paid tiers available

---

## Email Integration

### Outbound Email (Sendgrid)

**Purpose**: Send violation reports to authorities, user notifications.

**Configuration**:
```ruby
# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  address: 'smtp.sendgrid.net',
  port: 587,
  user_name: ENV['SENDGRID_USERNAME'],
  password: ENV['SENDGRID_PASSWORD'],
  authentication: :plain,
  enable_starttls_auto: true
}
```

**Mailers**:
- `NoticeMailer`: Send violation reports
- `UserMailer`: Authentication, reminders, notifications

**Example - Violation Report**:
```ruby
NoticeMailer.charge(notice, recipient_email).deliver_later
```

Email includes:
- Violation details
- PDF attachment (optional)
- Photos attached
- User signature
- Reply-to address: `#{notice.token}@anzeige.weg.li`

### Inbound Email (Action Mailbox)

**Purpose**: Receive replies from authorities.

**Configuration**:
```ruby
# config/environments/production.rb
config.action_mailbox.ingress = :sendgrid
```

**Sendgrid Inbound Parse Setup**:
1. Add MX records pointing to Sendgrid
2. Configure hostname: `anzeige.weg.li`
3. Set webhook URL: `https://www.weg.li/rails/action_mailbox/sendgrid/inbound_emails`

**Email Format**:
```
To: abc123xyz@anzeige.weg.li
```

Where `abc123xyz` is the notice token.

**Processing**:
```ruby
# app/mailboxes/autoreply_mailbox.rb
class AutoreplyMailbox < ApplicationMailbox
  def process
    notice = Notice.find_by(token: mail.to.first.split('@').first)
    Reply.create(notice: notice, inbound_email: inbound_email)

    # Notify user
    UserMailer.reply_notification(notice.user, notice).deliver_later
  end
end
```

---

## OAuth Providers

### Supported Providers

1. **GitHub**
2. **Google**
3. **Twitter**
4. **Telegram**
5. **Amazon**
6. **Apple**
7. **Email** (passwordless)

### Configuration

**Gem**: `omniauth` with provider-specific gems

**File**: `config/initializers/omniauth.rb`

```ruby
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :github,
    ENV['GITHUB_CONSUMER_KEY'],
    ENV['GITHUB_CONSUMER_SECRET'],
    scope: 'user:email'

  provider :google_oauth2,
    ENV['GOOGLE_CONSUMER_KEY'],
    ENV['GOOGLE_CONSUMER_SECRET'],
    scope: 'email,profile'

  provider :twitter,
    ENV['TWITTER_CONSUMER_KEY'],
    ENV['TWITTER_CONSUMER_SECRET']

  provider :telegram,
    ENV['TELEGRAM_BOT_NAME'],
    ENV['TELEGRAM_BOT_SECRET']

  # ... other providers
end
```

### Authentication Flow

1. User clicks "Sign in with GitHub"
2. Redirect to `/auth/github`
3. OmniAuth redirects to GitHub OAuth page
4. User authorizes application
5. GitHub redirects to `/auth/github/callback`
6. `SessionsController#create` processes callback:
   ```ruby
   auth = request.env['omniauth.auth']
   user = User.from_omniauth(auth)
   authorization = user.authorizations.find_or_create_by(
     provider: auth.provider,
     uid: auth.uid
   )
   session[:user_id] = user.id
   ```

### Provider Setup Guides

**GitHub**:
1. Visit https://github.com/settings/developers
2. Register OAuth App
3. Set callback URL: `https://www.weg.li/auth/github/callback`
4. Copy Client ID and Secret

**Google**:
1. Visit https://console.cloud.google.com/apis/credentials
2. Create OAuth 2.0 Client ID
3. Set authorized redirect URI: `https://www.weg.li/auth/google_oauth2/callback`
4. Copy Client ID and Secret

**Twitter**:
1. Visit https://developer.twitter.com/en/portal/projects-and-apps
2. Create App
3. Set callback URL: `https://www.weg.li/auth/twitter/callback`
4. Copy API Key and Secret

---

## Alternative Services

### car-ml Service

**Purpose**: Backup for Google Cloud Vision API.

**URL**: `https://weg-li-car-ml.onrender.com`

**Configuration**:
```bash
export CAR_ML_URL=https://weg-li-car-ml.onrender.com
```

**Usage**:
```ruby
# In Annotator
def annotate_yolo(key)
  url = "#{ENV['CAR_ML_URL']}/analyze"
  response = HTTP.post(url, json: { image_url: photo_url(key) })
  JSON.parse(response.body)
end
```

**Response**:
```json
{
  "license_plate_number": "B-AB 1234",
  "make": "Mercedes-Benz",
  "color": "silver"
}
```

---

## Webhook Notifications

### Slack Integration

**Purpose**: Error notifications, monitoring alerts.

**File**: `app/lib/slack.rb`

**Configuration**:
```bash
export SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
```

**Usage**:
```ruby
Slack.notify("Notice #{notice.id} failed to send", channel: '#errors')
```

**Events**:
- Stuck Sidekiq jobs
- Failed email deliveries
- Vision API errors
- Critical application errors

---

## Rate Limiting

**Gem**: `rack-attack`

**File**: `config/initializers/rack_attack.rb`

### Rules

**API Rate Limits**:
```ruby
throttle('api/ip', limit: 100, period: 1.minute) do |req|
  req.ip if req.path.start_with?('/api')
end

throttle('api/key', limit: 1000, period: 1.hour) do |req|
  req.env['HTTP_X_API_KEY'] if req.path.start_with?('/api')
end
```

**Authentication**:
```ruby
throttle('auth/ip', limit: 5, period: 20.seconds) do |req|
  req.ip if req.path.start_with?('/auth')
end
```

**Email Sending**:
```ruby
throttle('mail/user', limit: 10, period: 1.day) do |req|
  req.session[:user_id] if req.path == '/notices/:id/mail'
end
```

---

This comprehensive integration architecture enables weg-li to provide intelligent, automated violation reporting while maintaining security and scalability.
