# Features and Workflows Documentation

## Core Features

### 1. Notice Creation and Management

The central feature of weg-li is creating and managing parking violation reports (notices).

#### Single Notice Creation

**Workflow**:

1. **Navigate to New Notice**:
   - Click "Neue Meldung" from dashboard
   - Or: POST to `/notices` endpoint

2. **Upload Photos**:
   - Drag and drop or click to select
   - Multiple photos supported (up to 10)
   - Accepted formats: JPG, PNG
   - Maximum size: 10MB per photo
   - Photos stored in Google Cloud Storage

3. **Automatic Analysis Triggered**:
   - Notice status set to "analyzing"
   - `AnalyzerJob` queued in Sidekiq
   - Processing happens in background

4. **EXIF Analysis**:
   - GPS coordinates extracted from photos
   - Timestamp extracted (date/time of violation)
   - If GPS found:
     - Reverse geocoding to get address
     - Proximity analysis for common violations nearby
   - Auto-fills: location, date/time

5. **Vision API Analysis**:
   - Photos sent to Google Cloud Vision
   - License plate OCR
   - Vehicle brand recognition
   - Vehicle color detection
   - Auto-fills: registration, brand, color

6. **User Review and Edit**:
   - Status changed to "open"
   - User can modify any auto-filled data
   - Required fields:
     - Registration (license plate)
     - Street address
     - City
     - ZIP code
     - Violation type (charge/tbnr)
     - Start date/time
   - Optional fields:
     - End date/time (for duration calculation)
     - Brand, color
     - Additional notes
     - Flags: vehicle empty, hazard lights, expired TÜV, etc.

7. **Save as Draft**:
   - Can save incomplete notices
   - Uses `Incompletable` concern
   - Validation bypassed for drafts

8. **Preview and Share**:
   - Click "Teilen" to preview
   - Select recipient email (district or aliases)
   - Choose export format:
     - Email with PDF
     - Email with photos attached
     - WiNoWiG XML (for compatible districts)
     - OWI21 XML (for Friedberg, Hessen)

9. **Submit to Authority**:
   - Notice status changed to "shared"
   - `sent_at` timestamp recorded
   - Email sent to district with:
     - Violation details
     - User information
     - Photos attached
     - PDF report (optional)
     - Reply-to address: `#{token}@anzeige.weg.li`

10. **Track Replies**:
    - Authorities can reply to notice-specific email
    - Action Mailbox receives replies
    - Reply records created and linked to notice
    - User notified of reply

#### Bulk Upload

**Purpose**: Efficiently create multiple notices from a set of photos.

**Workflow**:

1. **Create Bulk Upload**:
   - Navigate to "Bulk Upload"
   - Upload multiple photos (dozens or hundreds)
   - Or: Import from Google Photos shared album

2. **Google Photos Import**:
   - Paste shared album URL
   - Click "Import"
   - `PhotosDownloadJob` queued
   - Photos downloaded from album
   - ZIP files extracted automatically

3. **Review Uploaded Photos**:
   - All photos displayed in grid
   - Can delete unwanted photos
   - Photos not yet attached to notices

4. **Create Notices**:
   - Choose mode:
     - **One notice per photo**: Each photo becomes separate notice
     - **Group by time**: Photos close in time grouped together
   - Bulk analysis triggered for all photos
   - Notices created in "analyzing" status

5. **Batch Processing**:
   - Each notice analyzed independently
   - EXIF + Vision API for each
   - User reviews each notice individually
   - Or: Bulk operations:
     - Bulk share (send all at once)
     - Bulk delete
     - Bulk status change

---

### 2. Intelligent Auto-Fill System

weg-li uses multiple data sources to automatically fill notice details, reducing manual data entry.

#### User Preferences (Autosuggest Bitfields)

Users can enable/disable each auto-fill source:

```ruby
user.autosuggest_from_exif?        # Use GPS/timestamp from photos
user.autosuggest_from_recognition? # Use OCR/Vision API results
user.autosuggest_from_proximity?   # Suggest violations from nearby
user.autosuggest_from_history?     # Auto-fill from previous reports
```

#### Auto-Fill Sources

**1. EXIF Metadata**:
- **Enabled by**: `autosuggest_from_exif`
- **Data extracted**:
  - GPS coordinates (latitude, longitude)
  - Timestamp (date/time)
  - Camera make/model
- **Auto-fills**:
  - Location (via reverse geocoding)
  - Street, city, ZIP
  - Start date/time
  - District (via ZIP lookup)

**2. Google Cloud Vision**:
- **Enabled by**: `autosuggest_from_recognition`
- **Data extracted**:
  - License plates (OCR)
  - Vehicle brands (logo recognition)
  - Vehicle colors (dominant color analysis)
- **Auto-fills**:
  - Registration
  - Brand
  - Color

**3. Proximity Analysis**:
- **Enabled by**: `autosuggest_from_proximity`
- **How it works**:
  - When GPS coordinates available
  - Query: `Notice.nearest_tbnrs(coordinates)`
  - Finds: Violations within 50m in last 6 months
  - Returns: Most common violation types
- **Auto-fills**:
  - Suggested charge (tbnr)
  - Common violation at this location

**4. User History**:
- **Enabled by**: `autosuggest_from_history`
- **How it works**:
  - Searches user's previous notices
  - Matches by registration (license plate)
  - Finds most recent matching notice
- **Auto-fills**:
  - Brand, color (from previous report)
  - Location (if same vehicle seen before)
  - Charge (if same violation pattern)
  - Flags (vehicle empty, hazard lights, etc.)
  - Notes (similar context)

#### Processing Priority

When multiple sources provide conflicting data:

1. User history (most specific to this vehicle)
2. Vision API (most accurate for current photo)
3. EXIF (most reliable for location/time)
4. Proximity (general suggestion)

User can always override any auto-filled data.

---

### 3. Geographic Features

weg-li uses PostGIS for advanced geographic functionality.

#### Spatial Queries

**Find Notices Nearby**:
```sql
SELECT * FROM notices
WHERE ST_DWithin(
  lonlat,
  ST_SetSRID(ST_MakePoint(13.404954, 52.520008), 4326)::geography,
  50  -- 50 meters
)
```

**Distance Calculation**:
```ruby
notice1.lonlat.distance(notice2.lonlat)  # Returns meters
```

**Proximity Analysis**:
```ruby
Notice.nearest_tbnrs(coordinates)
# Returns: Hash of tbnr => count for violations within 50m
```

#### Map Visualization

**Notice Map** (`/notices/map`):
- Interactive map showing all user's notices
- Markers colored by status:
  - Green: Shared
  - Yellow: Open
  - Blue: Analyzing
  - Gray: Disabled
- Click marker to view notice details
- Cluster markers when zoomed out

**Public Map** (planned feature):
- Heatmap of violations by area
- Anonymized data
- Filter by violation type, time period

#### District Boundaries

Each district has a center point (latitude, longitude) and maximum distance:
- Validates notices are within district bounds
- Prevents accidental reports to wrong authority
- Distance validation: `Geo::MAX_DISTANCE` (configurable)

---

### 4. License Plate Recognition

Advanced German license plate recognition with multiple parsing strategies.

#### German Plate Format

**Standard Format**: `PREFIX-LETTERS DIGITS`
- PREFIX: 1-3 letters (district code, e.g., "B" for Berlin)
- LETTERS: 1-2 letters
- DIGITS: 1-4 digits
- Optional: "E" suffix for electric vehicles

**Examples**:
- `B-AB 1234` (Berlin)
- `HH-XY 567` (Hamburg)
- `M-EV 890E` (Munich, electric)

#### Recognition Pipeline

**1. Google Cloud Vision OCR**:
```ruby
# In Annotator
response = vision_client.annotate_image(image_uri)
text_annotations = response.text_annotations

# Extract text blocks
texts = text_annotations.map(&:description)
# ["B-AB 1234", "Mercedes", "Hauptstrasse", ...]
```

**2. Plate Validation**:
```ruby
# In Vehicle module
def self.plate?(text)
  text.match?(plate_regex) || text.match?(relaxed_plate_regex)
end

plate_regex = /\A[A-ZÄÖÜ]{1,3}[\s\-]*[A-Z]{1,2}[\s\-]*\d{1,4}E?\z/
```

**3. Normalization**:
```ruby
def self.normalize(plate)
  plate.upcase
       .gsub(/[ÄÖÜØ]/, 'Ä' => 'AE', 'Ö' => 'OE', 'Ü' => 'UE', 'Ø' => 'OE')
       .gsub(/[\s\-]+/, ' ')
       .strip
end
# "b-ab1234" => "B-AB 1234"
```

**4. District Validation**:
```ruby
def self.district_for_plate(plate)
  prefix = plate.split(/[\s\-]/).first
  District.where("prefixes LIKE ?", "%#{prefix}%").first
end
```

**5. Likelihood Ranking**:
```ruby
def self.by_likelyhood(plates)
  plates.sort_by do |plate|
    score = 0
    score += 10 if plate.match?(perfect_format)
    score += 5 if district_prefix_exists?(plate)
    score += 3 if reasonable_length?(plate)
    score
  end.reverse
end
```

#### Quirky Mode

For difficult-to-read plates, relaxed regex:
```ruby
quirky_mode_plate_regex = /[A-ZÄÖÜ]{1,3}.*?[A-Z]{1,2}.*?\d{1,4}E?/
```

Allows:
- Extra spaces or dashes
- Some punctuation
- Partial matches

#### Manual Override

Users can always:
- Edit recognized plate
- Enter plate manually if OCR fails
- Flag for review

---

### 5. Violation Catalog (TBNR System)

weg-li uses Germany's official TBNR (Tatbestandsnummer) catalog.

#### TBNR Structure

**Format**: 6-digit code
- Digit 1: Violation category
- Digits 2-6: Specific violation

**Example**:
- `112454`: "Parken im eingeschränkten Halteverbot" (Parking in restricted no-stopping zone)
- First digit `1`: Traffic violations
- Classification `5`: Parking violations

#### Common Parking Violations (Classification 5)

**Top 5 Most Reported** (from `Charge::FAVS`):
1. `112454`: Restricted no-stopping zone (€25)
2. `142100`: Blocking bike lane (€55)
3. `141312`: Parking on sidewalk (€55)
4. `142200`: Blocking bike lane with obstruction (€70-100)
5. `142230`: Blocking bike lane causing danger (€80-100)

#### Charge Attributes

```ruby
charge = Charge.find_by(tbnr: '142100')
charge.description  # "Sie parkten auf einem Radweg."
charge.fine         # 55.00
charge.points       # 1
charge.classification  # 5 (parking)
```

#### Search and Selection

**In UI**:
- Dropdown with favorites
- Search by description
- Filter by classification
- Show fine amounts

**Charge Variants**:
- Some violations have refinements
- Example: "with obstruction", "causing danger"
- Different fine amounts

---

### 6. District Management

Districts represent German municipalities that receive violation reports.

#### District Database

**Coverage**: 3377+ districts covering all of Germany

**Key Fields**:
- `name`: Official municipality name
- `zip`: ZIP code (unique identifier)
- `email`: Authority email address
- `aliases`: Alternative email addresses (comma-separated)
- `prefixes`: License plate prefixes (e.g., "B,BP" for Berlin)
- `state`: German federal state (Bundesland)
- `config`: Integration type (standard, owi21, winowig, etc.)

#### Finding Your District

**By ZIP Code**:
```ruby
district = District.find_by(zip: '10115')
# => Berlin Mitte
```

**By License Plate**:
```ruby
district = District.for_prefix('B')
# => [Berlin districts]
```

**By Location**:
```ruby
# Reverse geocoding provides ZIP
# ZIP lookup provides district
```

#### Email Configuration

**Primary Email**:
- Main authority contact
- Where most reports are sent

**Alias Emails**:
- Alternative departments
- Specific teams (e.g., bicycle unit)
- User can choose recipient

**Personal Email Flag**:
- Some districts use personal emails
- Flag allows anonymization in public listings
- Privacy protection

#### Integration Types (Config)

**Standard** (config: 0):
- Regular email with PDF
- Most common

**Signature** (config: 1):
- Requires user signature attachment
- More formal process

**Munich** (config: 2):
- Special PIS (Police Information System) integration
- Custom XML format

**OWI21** (config: 3):
- Friedberg (Hessen) format
- Requires GMK (municipality key)
- XML export

**Hamburg** (config: 4):
- Hamburg-specific format
- Custom processing

**WiNoWiG** (config: 5):
- Standardized XML format
- Widerspruchs- und Informa­tions­system für Ordnungs­widrigkeiten
- Growing adoption

---

### 7. PDF and XML Export

Multiple export formats for compatibility with different authority systems.

#### PDF Generation

**File**: `app/lib/pdf_generator.rb`

**Purpose**: Human-readable violation report

**Includes**:
- Header with weg.li branding
- Violation details table:
  - Registration, brand, color
  - Location (street, city, ZIP)
  - Date/time, duration
  - Violation code and description
  - Fine amount
- User information:
  - Name, address
  - Email, phone (optional)
  - Signature image (if attached)
- Photos:
  - All attached photos embedded
  - Scaled to fit page
  - Captions with date/time
- Footer:
  - QR code linking to public charge page
  - weg.li URL
  - Page numbers

**Technical Details**:
- Gem: `prawn` for PDF generation
- Gem: `rqrcode` for QR codes
- Encoding: Windows-1252 (compatibility with German systems)
- Font: Helvetica
- Page size: A4

**Generation**:
```ruby
pdf_data = PdfGenerator.generate(notice, user)
send_data pdf_data, filename: "meldung_#{notice.token}.pdf"
```

#### WiNoWiG XML Export

**File**: `app/lib/xml_generator.rb`

**Purpose**: Standardized format for digital processing

**Schema**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<winowig>
  <meldung>
    <kennzeichen>B-AB 1234</kennzeichen>
    <marke>Mercedes-Benz</marke>
    <farbe>Silber</farbe>
    <ort>
      <strasse>Hauptstrasse 1</strasse>
      <plz>10115</plz>
      <stadt>Berlin</stadt>
    </ort>
    <tatzeit>
      <von>2024-11-21T14:30:00</von>
      <bis>2024-11-21T15:30:00</bis>
    </tatzeit>
    <tbnr>112454</tbnr>
    <beschreibung>Parken im eingeschränkten Halteverbot</beschreibung>
    <melder>
      <name>Max Mustermann</name>
      <email>max@example.com</email>
      <adresse>Beispielstrasse 1, 10115 Berlin</adresse>
    </melder>
    <fotos>
      <foto>
        <url>https://storage.googleapis.com/...</url>
        <timestamp>2024-11-21T14:30:22</timestamp>
      </foto>
    </fotos>
  </meldung>
</winowig>
```

#### OWI21 Export

**Purpose**: Friedberg (Hessen) specific format

**Additional Fields**:
- `gmk`: Municipality key (Gemeindekennziffer)
- Structured offense data
- Specific XML schema version

**Generation**:
```ruby
XmlGenerator.owi21_format(notice, district)
```

---

### 8. User Management

#### Registration and Authentication

**OAuth Providers**:
- GitHub, Google, Twitter, Telegram, Amazon, Apple
- One-click signup
- No password needed

**Email Authentication** (Passwordless):
1. Enter email address
2. Receive magic link
3. Click link to authenticate
4. JWT token validation
5. Session created

**Benefits**:
- No password to remember
- More secure (no password leaks)
- Faster signup process

#### User Profile

**Required Fields**:
- Email (validated)
- Nickname (public display name)

**Optional Fields**:
- Full name
- Address (for formal reports)
- Phone number
- Signature image

**Settings**:
- Autosuggest preferences (4 bitfield options)
- Email notification preferences
- Public profile visibility
- Reminder email opt-out

#### Access Levels

**User** (0): Regular users
- Create and manage own notices
- Upload photos
- View own statistics

**Community** (1): Community moderators
- All user permissions
- View aggregated statistics
- Access to community features (planned)

**Admin** (42): Administrators
- Full system access
- User management
- District management
- Sidekiq dashboard
- System configuration

#### User Statistics

**Dashboard**:
- Total notices created
- Notices shared (sent to authorities)
- Open notices (drafts)
- Total photos uploaded
- Success rate
- Weekly/monthly trends

**Leaderboard**:
- Weekly top contributors
- All-time rankings
- Opt-out available (hide_public_profile flag)

---

### 9. Reply Tracking

Automated tracking of responses from authorities.

#### Inbound Email Setup

**Email Format**: `#{notice.token}@anzeige.weg.li`

**Example**: `abc123xyz@anzeige.weg.li`

**Each notice has unique email address**:
- Token generated on creation
- Automatically included in sent reports (Reply-To field)
- Action Mailbox routes to correct notice

#### Reply Processing

**Flow**:
1. Authority sends email to notice-specific address
2. Sendgrid receives email
3. Webhook to Action Mailbox: `/rails/action_mailbox/sendgrid/inbound_emails`
4. `AutoreplyMailbox` processes email
5. Reply record created:
   ```ruby
   Reply.create(
     notice: notice,
     inbound_email: inbound_email,
     created_at: Time.current
   )
   ```
6. User notified (unless disabled):
   ```ruby
   UserMailer.reply_notification(user, notice).deliver_later
   ```

#### Reply Management

**User Interface**:
- Replies shown on notice detail page
- Display: sender, subject, date, body
- Full email content accessible
- Can download original email

**Notifications**:
- Email notification to user
- Optional: disable via `disable_autoreply_notifications` flag
- Badge count on dashboard

---

### 10. Scheduled Jobs and Maintenance

Background jobs that keep the system running smoothly.

#### Daily Jobs

**AnalyticsJob** (6:00 AM):
- Calculate usage statistics
- Update user metrics
- Generate reports
- Archive old data

**GeocodingCleanupJob** (7:00 AM):
- Fix failed geocoding attempts
- Re-geocode addresses with errors
- Update coordinates

**DataCleanupJob** (5:00 AM):
- Remove orphaned records
- Clean up expired tokens
- Purge old analysis results

**NoticeArchiverJob** (4:00 AM):
- Archive notices older than 2 years
- Move to cold storage (optional)
- Maintain database performance

#### Weekly Jobs

**ExportJob** (Monday 3:00 AM):
- Generate weekly backups
- Export statistics
- Send summary reports to admins

**Reminder Jobs** (Monday mornings):
- **ActivationReminderJob** (8:10 AM):
  - Remind users with open drafts
  - Encourage completion and sharing

- **UsageReminderJob** (8:20 AM):
  - Remind users about bulk upload feature
  - Promote efficient reporting

- **ExpiringReminderJob** (8:30 AM):
  - Remind about old open notices
  - Suggest sharing or deletion

#### Frequent Jobs

**MaterializedViewUpdaterJob** (Every 5 minutes):
- Refresh homepage statistics
- Update leaderboard
- Fast public page loads

**StuckJob** (Hourly):
- Monitor Sidekiq for stuck processes
- Kill zombie jobs
- Alert via Slack if issues found

---

This comprehensive feature set enables weg-li to be a powerful yet user-friendly tool for civic engagement in improving road safety and urban mobility.
