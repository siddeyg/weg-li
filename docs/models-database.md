# Models and Database Documentation

## Overview

The weg-li database schema is built on PostgreSQL with the PostGIS extension for geographic functionality. The system uses Active Record as the ORM layer.

## Core Models

### Notice

**File**: `app/models/notice.rb` (439 lines)

The central model representing a parking violation report.

#### Associations
- `belongs_to :user`
- `belongs_to :district` (via zip code lookup)
- `belongs_to :charge` (via tbnr code)
- `belongs_to :brand` (optional)
- `belongs_to :bulk_upload` (optional)
- `has_many :photos` (Active Storage attachments)
- `has_many :replies` (responses from authorities)
- `has_many_attached :photos`
- `has_many :data_sets` (analysis results)

#### Key Attributes

**Location**:
- `street` (string): Street name
- `city` (string): City name
- `zip` (string): ZIP code (links to district)
- `location` (string): Formatted address
- `latitude`, `longitude` (float): GPS coordinates
- `lonlat` (geography, Point, SRID 4326): PostGIS geometry for spatial queries

**Vehicle Information**:
- `registration` (string): License plate number
- `brand` (string): Vehicle manufacturer
- `color` (string): Vehicle color
- `charge` (string): TBNR violation code

**Violation Details**:
- `tbnr` (string): Traffic violation code (6 digits)
- `note` (text): Additional details from user
- `date` (date): Violation date
- `start_date`, `end_date` (datetime): Violation time window
- `duration` (integer): Duration in minutes

**Metadata**:
- `token` (string, unique): Public identifier for sharing
- `wegli_email` (string): Reply-to email address
- `status` (integer): Current processing state
- `sent_at` (datetime): When shared with authorities
- `vehicle_empty` (boolean, bitfield)
- `hazard_lights` (boolean, bitfield)
- `expired_tuv` (boolean, bitfield)
- `expired_eco` (boolean, bitfield)
- `over_2_8_tons` (boolean, bitfield)

#### Status Enum
```ruby
enum status: {
  open: 0,        # Ready to be shared
  disabled: 1,    # Hidden/deleted by user
  analyzing: 2,   # Currently being processed
  shared: 3       # Submitted to authorities
}
```

#### Scopes
- `for_public`: Shared notices, not disabled, with photos
- `since(date)`: Created after date
- `by_token(token)`: Find by public token
- `with_page_and_order`: Pagination with default ordering

#### Methods

**Geocoding**:
- `geocode`: Convert address to coordinates
- `reverse_geocode`: Convert coordinates to address
- `coordinates`: Returns [latitude, longitude]

**Validation**:
- `incomplete?`: Check if required fields are missing
- `complete?`: Opposite of incomplete
- `status_required_fields`: Fields needed before sharing

**Analysis**:
- `analyze!`: Trigger AnalyzerJob
- `apply_favorites`: Auto-fill from user's history
- `apply_dataset(data_set)`: Apply analysis results

**Sharing**:
- `district_email`: Get authority email address
- `all_emails`: Primary + alias emails
- `generate_public_token!`: Create shareable URL token

**Queries**:
- `self.nearest_tbnrs(lonlat)`: Find common violations nearby (50m, last 6 months)
- `self.similar(notice)`: Find similar reports

#### Validations
- Registration format via `Vehicle.plate?`
- Date range validation (start_date < end_date)
- Address validation (street, city, zip)
- District existence via ZIP code
- Maximum distance from district center

#### Callbacks
- `before_validation`: Normalize registration, set defaults
- `after_create`: Generate token, set wegli_email
- `after_commit`: Trigger geocoding if address changed

---

### User

**File**: `app/models/user.rb` (238 lines)

User accounts with OAuth authentication and preferences.

#### Associations
- `has_many :notices`
- `has_many :bulk_uploads`
- `has_many :exports`
- `has_many :snippets`
- `has_many :authorizations` (OAuth providers)
- `has_many :replies` (through notices)
- `belongs_to :district` (via zip)
- `has_one_attached :signature`

#### Key Attributes

**Profile**:
- `email` (string, unique, required)
- `nickname` (string, required)
- `name` (string): Full name
- `address` (text): Full address
- `zip` (string): ZIP code (links to district)
- `token` (string): Session token
- `api_token` (string): API authentication token
- `validation_date` (datetime): When email was validated

**Preferences**:
- `access` (integer): Access level
- `flags` (integer): Bitfield for boolean settings
- `autosuggest` (integer): Bitfield for auto-fill preferences

**Timestamps**:
- `created_at`, `updated_at`
- `email_sent_at`: Last reminder email
- `reminder_sent_at`: Last activation reminder

#### Access Levels
```ruby
enum access: {
  to_delete: -100,  # Marked for deletion
  disabled: -99,    # Account disabled
  user: 0,          # Regular user
  community: 1,     # Community moderator
  studi: 2,         # Student/researcher
  admin: 42         # Administrator
}
```

#### Bitfield Flags
```ruby
bitfield :flags,
  hide_public_profile: 1,                  # Don't show on leaderboard
  disable_reminders: 2,                    # No email reminders
  disable_autoreply_notifications: 4       # No reply notifications
```

#### Autosuggest Bitfields
```ruby
bitfield :autosuggest,
  from_exif: 1,         # Use GPS/timestamp from photos
  from_proximity: 2,    # Suggest violations from nearby reports
  from_recognition: 4,  # Use OCR/Vision API results
  from_history: 8       # Auto-fill from previous reports
```

#### Validations
- Email format and uniqueness
- Email not in blocklist (Blockable concern)
- MX record validation for email domain
- Nickname presence
- Address format (if provided)

#### Methods

**Authentication**:
- `self.authenticated_with_token(token)`: Find user by session token
- `generate_api_token!`: Create new API token
- `validate!`: Mark email as validated

**Statistics**:
- `open_notices_count`: Count of open notices
- `shared_notices_count`: Count of shared notices
- `total_photos`: Count of uploaded photos

**Convenience**:
- `display_name`: Returns nickname or email
- `district_name`: Name of user's district
- `admin?`, `community?`: Access level checks

---

### District

**File**: `app/models/district.rb` (124 lines)

German municipalities/jurisdictions that receive violation reports.

#### Associations
- `has_many :notices` (via zip)
- `has_many :users` (via zip)

#### Key Attributes

**Identification**:
- `name` (string, required): Municipality name (e.g., "Berlin Mitte")
- `zip` (string, unique, required): ZIP code (primary key)
- `email` (string): Primary contact email
- `aliases` (string): Alternative email addresses (comma-separated)
- `prefixes` (string): License plate prefixes (comma-separated, e.g., "B,BP")

**Location**:
- `latitude`, `longitude` (float): District center coordinates
- `osm_id` (bigint): OpenStreetMap identifier

**Configuration**:
- `config` (integer): Integration type
- `gmk` (string): Municipality key (Gemeindekennziffer) for OWI21
- `state` (string): German state (Bundesland)
- `status` (integer): Active or proposed
- `flags` (integer): Bitfield for options

#### Config Types
```ruby
enum config: {
  standard: 0,     # Regular email submission
  signature: 1,    # Requires user signature
  munich: 2,       # Special PIS integration
  owi21: 3,        # Friedberg (Hessen) OWI21 format
  hamburg: 4,      # Hamburg-specific format
  winowig: 5       # WiNoWiG XML format
}
```

#### Status Enum
```ruby
enum status: {
  active: 0,    # Accepts reports
  proposed: 1   # Not yet confirmed/active
}
```

#### Bitfield Flags
```ruby
bitfield :flags,
  personal_email: 1  # Anonymize contact info
```

#### Methods

**Email Handling**:
- `all_emails`: Returns array of primary + alias emails
- `display_email`: Returns email with potential anonymization
- `personal?`: Check if personal_email flag is set

**Validation**:
- `accepts?(plate)`: Check if license plate prefix matches district
- `prefix_list`: Parse comma-separated prefixes

**Search**:
- `self.by_name(query)`: Search districts by name
- `self.by_zip(zip)`: Find district by ZIP code
- `self.for_prefix(prefix)`: Find districts accepting a plate prefix

---

### Charge

**File**: `app/models/charge.rb` (92 lines)

Traffic violation catalog based on German TBNR (Tatbestandsnummer) system.

#### Associations
- `has_many :notices`
- `has_many :charge_variants`

#### Key Attributes

**Identification**:
- `tbnr` (string, 6 digits, unique): Violation code
- `description` (text): German description of violation
- `fap` (string): FAP code (Fachverfahren Außergerichtliches Parkmanagement)

**Penalties**:
- `fine` (decimal): Fine amount in EUR
- `bkat` (string): BKAT catalog reference
- `penalty` (string): Additional penalties
- `required_refinements` (string): Refinement options
- `max_fine` (decimal): Maximum fine amount
- `points` (integer): Points in Flensburg

**Classification**:
- `classification` (integer): Violation category (0-9)
  - 5 = Parking violations
  - Other categories for different violation types

**Validity**:
- `valid_from` (date): When charge became active
- `valid_to` (date): When charge was deprecated

#### Constants
```ruby
CLASSIFICATIONS = %w(StVO Vorfahrt Geschwindigkeit Abstand Überholen Parken Halten Sonstige Autobahn)
FAVS = %w(112454 142100 141312 142200 142230) # Most common parking violations
```

#### Scopes
- `active`: Charges currently valid
- `parking`: Classification = 5
- `favorites`: Most commonly used charges

#### Methods

**Display**:
- `label`: Format for dropdowns (code + description)
- `fine_text`: Formatted fine amount with EUR symbol

**Validation**:
- `currently_valid?`: Check if charge is active today

---

### Brand

**File**: `app/models/brand.rb` (83 lines)

Vehicle manufacturer database for brand recognition.

#### Associations
- `has_many :notices`

#### Key Attributes

**Identification**:
- `name` (string, required): Brand name (e.g., "Mercedes-Benz")
- `token` (string, unique): Normalized identifier (e.g., "mercedes")
- `aliases` (text): Alternative names (JSON array)

**Classification**:
- `kind` (integer): Vehicle type
- `models` (text): Known models (JSON array)
- `falsepositives` (text): Common misidentifications (JSON array)

#### Kind Enum
```ruby
enum kind: {
  car: 0,
  truck: 1,
  bike: 2,
  scooter: 3,
  van: 4
}
```

#### Methods

**Matching**:
- `self.fuzzy_search(query)`: Find brands by partial name match
- `self.by_token(token)`: Find by normalized token
- `matches?(text)`: Check if text matches brand name or aliases

**Normalization**:
- `self.normalize(name)`: Convert name to token format

---

### BulkUpload

**File**: `app/models/bulk_upload.rb` (38 lines)

Container for batch photo uploads.

#### Associations
- `belongs_to :user`
- `has_many :notices`
- `has_many_attached :photos`

#### Key Attributes
- `status` (integer): Processing state
- `shared_album_url` (string): Google Photos album URL

#### Status Enum
```ruby
enum status: {
  initial: 0,     # Created, awaiting photos
  processing: 1,  # Downloading from Google Photos
  open: 2,        # Photos uploaded, ready to create notices
  done: 3,        # Notices created
  importing: 4,   # Currently importing
  error: 5        # Import failed
}
```

#### Methods
- `import!`: Trigger PhotosDownloadJob
- `create_notices(mode)`: Generate notices from photos (one per photo or grouped)

---

### DataSet

**File**: `app/models/data_set.rb` (105 lines)

Polymorphic storage for analysis results from various sources.

#### Associations
- `belongs_to :setable` (polymorphic): Usually Notice
- `belongs_to :keyable` (polymorphic): Usually Photo

#### Key Attributes
- `kind` (integer): Data source type
- `data` (jsonb): Flexible JSON storage

#### Kind Enum
```ruby
enum kind: {
  google_vision: 0,  # Google Cloud Vision API results
  exif: 1,           # EXIF metadata from photos
  car_ml: 2,         # car-ml service results
  geocoder: 3,       # Geocoding results
  proximity: 4       # Nearby violation suggestions
}
```

#### Data Structure Examples

**Google Vision**:
```json
{
  "text_annotations": [...],
  "image_properties": {
    "dominant_colors": [
      {"color": {"red": 100, "green": 50, "blue": 30}, "score": 0.5}
    ]
  }
}
```

**EXIF**:
```json
{
  "date_time": "2024-11-21 14:30:22",
  "gps": {
    "latitude": 52.520008,
    "longitude": 13.404954,
    "altitude": 34
  }
}
```

**Proximity**:
```json
{
  "tbnrs": ["112454", "141312"],
  "distances": [15.5, 23.1],
  "addresses": ["Hauptstrasse 123", "Nebenstrasse 45"]
}
```

#### Methods
- `extract_registrations`: Parse license plates from Vision API data
- `extract_brands`: Parse vehicle brands
- `extract_colors`: Parse dominant colors
- `extract_location`: Get GPS coordinates from EXIF
- `extract_timestamp`: Get photo timestamp

---

### Reply

**File**: `app/models/reply.rb` (19 lines)

Email responses from authorities via Action Mailbox.

#### Associations
- `belongs_to :notice`
- `belongs_to :inbound_email` (Action Mailbox)

#### Key Attributes
- `notice_id` (integer): Associated notice
- `inbound_email_id` (integer): Raw email record

#### Methods
- `subject`: Email subject line
- `body`: Email body text
- `from`: Sender email address
- `received_at`: When email was received

---

### Authorization

**File**: `app/models/authorization.rb`

OAuth provider connections for users.

#### Associations
- `belongs_to :user`

#### Key Attributes
- `provider` (string): OAuth provider name (github, google, twitter, etc.)
- `uid` (string): User ID from provider
- `token` (string): Access token (encrypted)
- `secret` (string): OAuth secret (for OAuth 1.0)

#### Validations
- Unique combination of provider + uid
- Provider must be in allowed list

---

## Database Schema Highlights

### Indexes

**Performance Indexes**:
- `notices(user_id)`: Fast user notice lookup
- `notices(token)`: Public notice URL resolution
- `notices(status)`: Filter by processing state
- `notices(zip)`: District association
- `notices(registration)`: License plate search
- `districts(zip)`: Primary key
- `users(email)`: Login lookup
- `users(token)`: Session validation
- `charges(tbnr)`: Violation code lookup

**Spatial Indexes**:
- `notices(lonlat)` using GIST: PostGIS spatial queries

### Foreign Keys
- Strong referential integrity
- `CASCADE` delete for dependent records
- `RESTRICT` for critical associations

### Materialized Views

**homepages**:
```sql
SELECT
  COUNT(DISTINCT districts.id) as districts_count,
  COUNT(DISTINCT users.id) as users_count,
  COUNT(DISTINCT notices.id) as shared_count,
  COUNT(photos.id) as photos_count
FROM ...
WHERE notices.status = 3 AND notices.created_at > NOW() - INTERVAL '1 year'
```

**leaders**:
```sql
SELECT
  users.id,
  users.nickname,
  COUNT(notices.id) as notice_count,
  DATE_TRUNC('week', notices.created_at) as week
FROM users
JOIN notices ON ...
WHERE notices.status = 3
GROUP BY users.id, week
ORDER BY notice_count DESC
```

---

## Concerns

### Incompletable

Allows saving records with validation errors (for draft notices).

```ruby
module Incompletable
  def save_incomplete
    @incomplete = true
    save(validate: false)
  end

  def incomplete?
    @incomplete || !valid?
  end
end
```

### Blockable

Email validation and blocklist checking.

```ruby
module Blockable
  DISPOSABLE_DOMAINS = %w(tempmail.com guerrillamail.com ...)
  BLOCKED_DOMAINS = %w(gmail.com yahoo.com ...)

  def email_valid?
    # Format check
    # MX record validation
    # Disposable domain check
  end
end
```

### Bitfield

Efficient boolean flag storage.

```ruby
module Bitfield
  def bitfield(name, **flags)
    flags.each do |flag_name, bit_position|
      define_method("#{flag_name}?") do
        (send(name) & (1 << bit_position)) != 0
      end

      define_method("#{flag_name}=") do |value|
        # Set or clear bit
      end
    end
  end
end
```

---

## Migrations Overview

Key migration features:
- PostGIS extension setup
- Geography column for spatial data
- JSONB columns for flexible data
- Enum types for status fields
- Composite indexes for query optimization
- Materialized view creation and refresh

---

This comprehensive model structure enables weg-li to efficiently manage violation reports while maintaining data integrity and performance.
