# AI and Machine Learning Functions Documentation

## Overview

weg-li leverages multiple AI/ML technologies to automate the analysis of parking violation photos. This document provides an in-depth technical analysis of each AI function, including implementation details, data flows, algorithms, and integration patterns.

## Architecture Summary

```
┌──────────────────────────────────────────────────────────────────┐
│                        Photo Upload                               │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                       AnalyzerJob                                 │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    EXIF Analysis Pipeline                  │  │
│  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐   │  │
│  │  │    EXIF     │──▶│  Geocoder   │──▶│   Proximity     │   │  │
│  │  │  Extraction │   │  (Address)  │   │   Suggestions   │   │  │
│  │  └─────────────┘   └─────────────┘   └─────────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                   Vision Analysis Pipeline                 │  │
│  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐   │  │
│  │  │   Google    │──▶│   License   │──▶│     Brand       │   │  │
│  │  │   Vision    │   │   Plate OCR │   │   Recognition   │   │  │
│  │  │     API     │   └─────────────┘   └─────────────────┘   │  │
│  │  │             │──▶┌─────────────────────────────────────┐ │  │
│  │  └─────────────┘   │        Color Detection              │ │  │
│  │                    └─────────────────────────────────────┘ │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                   Alternative: car-ml Service              │  │
│  │  ┌─────────────────────────────────────────────────────┐   │  │
│  │  │  YOLO-based License Plate + Brand + Color Detection │   │  │
│  │  └─────────────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    User History Analysis                   │  │
│  │  ┌─────────────────────────────────────────────────────┐   │  │
│  │  │  Auto-fill from previously reported same vehicle    │   │  │
│  │  └─────────────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                    DataSet Storage (Polymorphic)                  │
│   Types: google_vision, exif, car_ml, geocoder, proximity        │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. Google Cloud Vision API Integration

### Purpose
Primary AI service for extracting text (license plates), identifying vehicle brands, and detecting dominant colors from uploaded photos.

### Implementation

**File**: `app/lib/annotator.rb`

```ruby
class Annotator
  def annotate_object(key = "ydmE3qL1CT32rH6hunWtxCzx")
    uri = self.class.bucket_uri(key)  # gs://weg-li-production/key
    image = { source: { gcs_image_uri: uri } }
    annotate(image)
  end

  def annotate(image)
    request = {
      requests: [{
        image: image,
        image_context: { language_hints: ["de"] },
        features: [
          { type: "DOCUMENT_TEXT_DETECTION" },
          { type: "IMAGE_PROPERTIES" }
        ]
      }]
    }
    response = image_annotator.batch_annotate_images(request)
    response.responses.first.to_h
  end
end
```

### API Features Used

#### DOCUMENT_TEXT_DETECTION
**Purpose**: Optical Character Recognition (OCR) optimized for documents

**Why this feature**:
- Better than `TEXT_DETECTION` for structured text like license plates
- Handles skewed, partially occluded, or low-contrast text
- Returns text blocks with bounding polygons

**Response Structure**:
```json
{
  "text_annotations": [
    {
      "description": "B-AB 1234\nMercedes-Benz\n...",
      "bounding_poly": {
        "vertices": [
          {"x": 10, "y": 20},
          {"x": 100, "y": 20},
          {"x": 100, "y": 50},
          {"x": 10, "y": 50}
        ]
      }
    },
    {
      "description": "B-AB 1234",
      "bounding_poly": {...}
    }
  ]
}
```

**Post-Processing**:
```ruby
# In DataSet#registrations
text_annotations = data.deep_symbolize_keys[:text_annotations]
with_likelyhood = Annotator.grep(text_annotations) { |it|
  Vehicle.plate?(it, prefixes: district&.prefixes)
}
Vehicle.by_likelyhood(with_likelyhood)
```

#### IMAGE_PROPERTIES
**Purpose**: Extract dominant colors from the image

**Response Structure**:
```json
{
  "image_properties_annotation": {
    "dominant_colors": {
      "colors": [
        {
          "color": {"red": 100, "green": 100, "blue": 100},
          "score": 0.45,
          "pixel_fraction": 0.32
        }
      ]
    }
  }
}
```

**Post-Processing**:
```ruby
# In Annotator.dominant_colors
colors = colors.select { |color| color[:score] >= threshold }
colors.map do |color|
  name = Colorizor.closest_match(color[:color])
  [name, (color[:score].to_f + color[:pixel_fraction].to_f).fdiv(2)]
end
```

### Configuration

**Environment Variables**:
```bash
GOOGLE_CLOUD_PROJECT=weg-li
GOOGLE_CLOUD_KEYFILE=/path/to/service-account.json
# OR
GOOGLE_APPLICATION_CREDENTIALS='{"type":"service_account",...}'
```

**Required IAM Roles**:
- `roles/vision.user` - Cloud Vision API User
- `roles/storage.objectViewer` - Read access to GCS bucket

**Bucket URI Format**:
```ruby
def self.bucket_uri(key)
  "gs://#{bucket_name}/#{key}"  # gs://weg-li-production/abc123.jpg
end
```

### Language Hints

The API is configured with German language hints:
```ruby
image_context: { language_hints: ["de"] }
```

This improves OCR accuracy for:
- German umlauts in license plates (Ö, Ä, Ü)
- German text in photos (street signs, vehicle badges)

### Safe Search Detection (Commented Out)

```ruby
# {type: 'SAFE_SEARCH_DETECTION'}  # Currently disabled
```

**Implementation exists but disabled**:
```ruby
def self.unsafe?(result)
  (result[:safe_search_annotation] || {}).any? { |key, value|
    %i[adult medical violence racy].include?(key) &&
    %i[LIKELY VERY_LIKELY].include?(value)
  }
end
```

Could be enabled to filter inappropriate content before processing.

### Cost and Rate Limiting

**Pricing** (as of 2024):
| Volume | Price per 1,000 |
|--------|-----------------|
| 0-1,000 | Free |
| 1,001-5,000,000 | $1.50 |
| 5,000,001+ | $0.60 |

**Rate Limits**:
- 1,800 requests/minute (default)
- Configurable via quota management

**Optimization Strategy**:
- Results cached in `data_sets` table (kind: `google_vision`)
- Single analysis per photo
- Manual re-analysis available

---

## 2. License Plate Recognition (OCR + Parsing)

### Purpose
Extract and validate German license plates from OCR text.

### Implementation

**File**: `app/lib/vehicle.rb`

### German License Plate Format

**Standard Format**: `PREFIX-LETTERS DIGITS[E]`

| Component | Description | Example |
|-----------|-------------|---------|
| PREFIX | 1-3 letters (district code) | B (Berlin), HH (Hamburg), M (Munich) |
| Separator | Dash or space | - |
| LETTERS | 1-2 letters | AB |
| Space | Separator | (space) |
| DIGITS | 1-4 digits | 1234 |
| E (optional) | Electric vehicle | E |

**Valid Examples**:
- `B-AB 1234` (Berlin)
- `HH-XY 567` (Hamburg)
- `M-EV 890E` (Munich, electric)
- `STA-B 1` (Starnberg, short format)

### Regex Patterns

**Strict Pattern** (`plate_regex`):
```ruby
def self.plate_regex(prefixes = Vehicle.plates.keys)
  Regexp.new("^(#{prefixes.join('|')}) ([A-Z]{1,3})\s?(\\d{1,4})(\s?E)?$")
end
# Example: /^(B|HH|M|...) ([A-Z]{1,3})\s?(\d{1,4})(\s?E)?$/
```

**Relaxed Pattern** (`relaxed_plate_regex`):
```ruby
def self.relaxed_plate_regex(prefixes = Vehicle.plates.keys)
  Regexp.new("^(#{prefixes.join('|')})O?:? ?([A-Z]{1,3})\s?(\\d{1,4})(\s?E)?$")
end
# Allows: "B:AB 1234", "BO AB 1234" (OCR errors)
```

**Quirky Mode Pattern** (`quirky_mode_plate_regex`):
```ruby
def self.quirky_mode_plate_regex(prefixes = Vehicle.plates.keys)
  Regexp.new("^P?D?C?O?B?(#{prefixes.join('|')})O?:? ?0?([A-Z]{1,3})\s?(\\d{1,4})(\s?E)?$")
end
# Handles severe OCR errors: "PDB-AB 1234", "OCB-AB 1234"
```

### Normalization Algorithm

```ruby
def self.normalize(text, text_divider: " ")
  return "" if text.blank?

  text
    .gsub("ö", "Ö")           # Normalize lowercase umlauts
    .gsub("ä", "Ä")
    .gsub("ü", "Ü")
    .gsub(/^([^A-Z,ÖÄÜ])+/, "") # Remove leading non-letters
    .gsub(/([^E,0-9])+$/, "")    # Remove trailing non-E/digits
    .gsub(/([^A-Z,ÖÄÜ0-9])+/, " ") # Replace separators with space
    .gsub(/([A-Z,ÖÄÜ]+) ([A-Z,ÖÄÜ]+)\s?(\d+)(\s?E)?/,
          "\\1#{text_divider}\\2 \\3\\4")  # Format output
end
```

**Input → Output Examples**:
- `"b-ab1234"` → `"B-AB 1234"`
- `"HH: XY 567"` → `"HH-XY 567"`
- `"M.EV.890e"` → `"M-EV 890E"`

### Likelihood Scoring

```ruby
def self.plate?(text, prefixes: nil, text_divider: " ")
  text = normalize(text)

  if prefixes.present? && text =~ plate_regex(prefixes)
    ["#{$1}#{text_divider}#{$2} #{$3}#{$4.to_s.gsub(/[^E]+/, '')}", 1.2]
  elsif prefixes.present? && text =~ relaxed_plate_regex(prefixes)
    ["#{$1}#{text_divider}#{$2} #{$3}#{$4.to_s.gsub(/[^E]+/, '')}", 1.1]
  elsif text =~ plate_regex
    ["#{$1}#{text_divider}#{$2} #{$3}#{$4.to_s.gsub(/[^E]+/, '')}", 1.0]
  elsif text =~ relaxed_plate_regex
    ["#{$1}#{text_divider}#{$2} #{$3}#{$4.to_s.gsub(/[^E]+/, '')}", 0.8]
  elsif text =~ quirky_mode_plate_regex
    ["#{$1}#{text_divider}#{$2} #{$3}#{$4.to_s.gsub(/[^E]+/, '')}", 0.5]
  end
end
```

**Scoring Breakdown**:
| Score | Condition |
|-------|-----------|
| 1.2 | Strict match + known district prefix |
| 1.1 | Relaxed match + known district prefix |
| 1.0 | Strict match (any prefix) |
| 0.8 | Relaxed match (any prefix) |
| 0.5 | Quirky mode match |

### Ranking Multiple Candidates

```ruby
def self.by_likelyhood(matches)
  return [] if matches.blank?

  groups = matches.group_by { |key, _| key.gsub(/\W/, "") }
  groups = groups.sort_by { |_, group|
    group.sum { |_, probability| probability }.fdiv(matches.size)
  }
  groups.map { |match| match[1].flatten[0] }.reverse
end
```

**Algorithm**:
1. Group matches by normalized plate (ignoring separators)
2. Sum likelihood scores for each group
3. Divide by total matches for relative score
4. Sort descending (highest confidence first)

### District Prefix Database

**File**: `config/data/plates.json`

```json
{
  "A": "Augsburg",
  "AA": "Ostalbkreis (Aalen)",
  "AB": "Aschaffenburg",
  "B": "Berlin",
  "HH": "Hamburg",
  "M": "München",
  ...
}
```

**Total Prefixes**: 700+ German district codes

**Usage**:
```ruby
Vehicle.plates["B"]  # => "Berlin"
Vehicle.district_for_plate_prefix("B-AB 1234")  # => "Berlin"
```

---

## 3. Vehicle Brand Recognition

### Purpose
Identify vehicle manufacturer from text detected in photos (logos, badges).

### Implementation

**File**: `app/models/brand.rb`

### Brand Database

**Source**: `config/data/cars.json` + Database records

**Structure**:
```json
{
  "name": "Mercedes-Benz",
  "token": "mercedes-benz",
  "kind": "car",
  "aliases": ["Mercedes", "MB", "Daimler"],
  "models": ["A-Klasse", "C-Klasse", "E-Klasse", "S-Klasse"],
  "falsepositives": ["AMG"]
}
```

### Recognition Algorithm

```ruby
def self.brand?(text)
  text = text.strip.downcase

  kinds.each_key do |kind|  # car, truck, bike, scooter, van
    res = in_brands?(text, send(kind))
    return res if res
  end
  nil
end

def self.in_brands?(text, brands)
  # Check false positives first
  brands.find { |entry|
    return false if entry.falsepositives.find { |ali| text == ali.strip.downcase }
  }

  # Exact name match (score: 1.0)
  res = brands.find { |entry| text == entry.name.strip.downcase }
  return res.name, 1.0 if res.present?

  # Alias match (score: 1.0)
  res = brands.find { |entry|
    entry.aliases.find { |ali| text == ali.strip.downcase }
  }
  return [res.name, 1.0] if res.present?

  # Partial name match (score: 0.8)
  res = brands.find { |entry| text.match?(entry.name.strip.downcase) }
  return res.name, 0.8 if res.present?

  # Model match (score: 0.5)
  res = brands.find { |entry|
    entry.models.find { |model|
      model =~ /\D{3,}/ && text == model.strip.downcase
    }
  }
  [res.name, 0.5] if res.present?
end
```

### Scoring Logic

| Match Type | Score | Example |
|------------|-------|---------|
| Exact name | 1.0 | "mercedes-benz" → Mercedes-Benz |
| Alias | 1.0 | "mb" → Mercedes-Benz |
| Partial name | 0.8 | "mercedes" → Mercedes-Benz |
| Model name | 0.5 | "a-klasse" → Mercedes-Benz |

### False Positive Filtering

```ruby
# AMG is often detected but is not the brand (it's a Mercedes sub-brand)
falsepositives: ["AMG"]

# Detection flow:
# 1. Check if text is in falsepositives → reject
# 2. Otherwise proceed with matching
```

---

## 4. Color Detection and Matching

### Purpose
Identify vehicle color from dominant colors in photos.

### Implementation

**File**: `app/lib/colorizor.rb`

### Color Mapping Database

Maps 150+ CSS/X11 color names to 14 basic vehicle colors:

```ruby
COLORS = {
  Color::RGB::AliceBlue => Color::RGB::White,
  Color::RGB::AntiqueWhite => Color::RGB::Beige,
  Color::RGB::Aqua => Color::RGB::Blue,
  Color::RGB::DarkGoldenRod => Color::RGB::Gold,
  Color::RGB::DarkGreen => Color::RGB::Green,
  # ... 150+ mappings
}
```

**Target Colors** (Official German vehicle registration colors):
- Black, White, Gray, Silver
- Red, Blue, Green, Yellow
- Brown, Beige, Orange, Gold
- Pink, Purple, Violet

### Algorithm

**Input**: RGB values from Google Vision API
```json
{"red": 100, "green": 100, "blue": 100}
```

**Processing**:
```ruby
def self.closest_match(color)
  # Create RGB color object
  rgb = Color::RGB.new(color[:red], color[:green], color[:blue])

  # Find closest match using color distance algorithm
  # First try with threshold of 20 (close matches only)
  closest = rgb.closest_match(COLORS.keys, 20)

  # Fallback to extended color set (includes metallic colors)
  closest ||= rgb.closest_match(TOTALS.keys)

  # Return simplified color name
  TOTALS[closest].name
end
```

### Color Distance Calculation

The `color` gem uses Delta E (CIE76) color difference:

```
ΔE = √[(L₁-L₂)² + (a₁-a₂)² + (b₁-b₂)²]
```

Where L*a*b* is the CIELAB color space representation.

### Special Handling: Metallic Colors

```ruby
TOTALS = {
  Color::RGB::Gray10 => Color::RGB::Gray,
  Color::RGB::Gray20 => Color::RGB::Gray,
  # ...
  Color::RGB::Metallic::Aluminum => Color::RGB::Silver,
  Color::RGB::Metallic::Iron => Color::RGB::Silver,
  Color::RGB::Metallic::Steel => Color::RGB::Silver,
  Color::RGB::Metallic::CoolCopper => Color::RGB::Gold,
  # ...
}.merge(COLORS)
```

### Score Calculation

```ruby
# In Annotator.dominant_colors
colors.map do |color|
  name = Colorizor.closest_match(color[:color])
  # Score = average of Google Vision score and pixel fraction
  [name, (color[:score].to_f + color[:pixel_fraction].to_f).fdiv(2)]
end
```

**Google Vision scores**:
- `score`: Confidence in color detection (0.0-1.0)
- `pixel_fraction`: Percentage of image pixels with this color

---

## 5. EXIF Metadata Extraction

### Purpose
Extract GPS coordinates and timestamps from photo metadata.

### Implementation

**File**: `app/lib/exif_analyzer.rb`

```ruby
require "exifr/jpeg"

class ExifAnalyzer
  def metadata(image, debug: false)
    meta = {}

    exif = EXIFR::JPEG.new(image).exif
    if exif.present?
      # Extract timestamp
      deepexif = exif.fields[:exif]
      if deepexif
        meta[:date_time] = deepexif.fields[:date_time_original] ||
                           deepexif.fields[:date_time_digitized]
      elsif exif.fields[:date_time]
        meta[:date_time] = exif.fields[:date_time]
      end

      # Extract GPS coordinates
      gps = exif.fields[:gps]
      if gps.present?
        meta[:latitude] = gps.fields[:gps_latitude]&.to_f || Float::NAN
        meta[:longitude] = gps.fields[:gps_longitude]&.to_f || Float::NAN
        meta[:altitude] = gps.fields[:gps_altitude]&.to_f || Float::NAN
      end
    end

    meta
  rescue EXIFR::MalformedJPEG => e
    Rails.logger.warn("could not process image: #{e}")
    {}
  end
end
```

### Extracted Fields

| Field | EXIF Tag | Purpose |
|-------|----------|---------|
| `date_time` | DateTimeOriginal, DateTimeDigitized | Violation timestamp |
| `latitude` | GPSLatitude | Location Y coordinate |
| `longitude` | GPSLongitude | Location X coordinate |
| `altitude` | GPSAltitude | Elevation (informational) |

### Timestamp Priority

1. `DateTimeOriginal` - When photo was taken (preferred)
2. `DateTimeDigitized` - When photo was digitized
3. `DateTime` - File modification time (fallback)

### Filename Timestamp Parsing

**File**: `app/jobs/analyzer_job.rb`

```ruby
def self.time_from_filename(filename)
  # Pattern: 20241121_143022
  token = filename[/.*(20\d{6}_\d{6})/, 1]
  # Pattern: 2024-11-21_14-30-22
  token ||= filename[/.*(20\d{2}-\d{2}-\d{2}_\d{2}-\d{2}-\d{2})/, 1]
  # Pattern: 2024-11-21-14-30-22-xxx
  token ||= filename[/.*(20\d{2}-\d{2}-\d{2}-\d{2}-\d{2}-\d{2})-.*/, 1]

  Time.zone.parse(token.gsub("-", "")) if token
end
```

Used when EXIF data is missing or stripped.

---

## 6. Geocoding and Reverse Geocoding

### Purpose
Convert GPS coordinates to addresses and vice versa.

### Implementation

**Gem**: `geocoder`

**Providers**:
- **Nominatim** (OpenStreetMap) - Free, default
- **OpenCage** - Alternative with API key

### Reverse Geocoding Flow

```ruby
# In AnalyzerJob#handle_exif
coords = exif_data_set.coords  # [latitude, longitude]
if coords.present?
  result = Geocoder.search(coords)
  if result.present?
    geocoder_data_set = notice.data_sets.create!(
      data: result,
      kind: :geocoder,
      keyable: photo
    )

    if notice.user.from_exif?
      address = geocoder_data_set.address
      notice.latitude = address[:latitude]
      notice.longitude = address[:longitude]
      notice.zip = address[:zip]
      notice.city = address[:city]
      notice.street = address[:street]
    end
  end
end
```

### Address Parsing

**File**: `app/models/data_set.rb`

```ruby
def address
  case kind
  when "geocoder"
    if data.present?
      result_klass = (
        if Geocoder.config.lookup == :nominatim
          Geocoder::Result::Nominatim
        else
          Geocoder::Result::Opencagedata
        end
      )
      result = result_klass.new(data.first["data"])
      Notice.geocode_data(result)
    end
  end
end
```

### Extracted Address Fields

| Field | Source | Example |
|-------|--------|---------|
| `street` | road + house_number | "Hauptstrasse 123" |
| `city` | city/town/village | "Berlin" |
| `zip` | postcode | "10115" |
| `latitude` | lat | 52.520008 |
| `longitude` | lon | 13.404954 |

---

## 7. Proximity-Based Suggestions

### Purpose
Suggest common violation types based on violations previously reported at the same location.

### Implementation

**File**: `app/models/notice.rb`

```ruby
def self.nearest_tbnrs(latitude, longitude, distance = 50, count = 10)
  sql = "
    SELECT
      tbnr,
      COUNT(tbnr) as count,
      SUM(ST_DistanceSphere(lonlat::geometry, ST_MakePoint($1, $2))) as distance,
      SUM(ST_DistanceSphere(lonlat::geometry, ST_MakePoint($1, $2))) / COUNT(tbnr) as diff
    FROM notices
    WHERE
      status = 3
      AND created_at > (CURRENT_DATE - INTERVAL '6 months')
      AND ST_DWithin(lonlat::geography, ST_MakePoint($1, $2), $3)
    GROUP BY tbnr
    ORDER BY diff
    LIMIT $4
  "
  binds = [longitude, latitude, distance, count]
  Notice.connection.exec_query(sql, "distance-query", binds).to_a
end
```

### Algorithm Details

**Spatial Query**: Uses PostGIS `ST_DWithin` for efficient radius search

**Parameters**:
- `distance`: Search radius in meters (default: 50m)
- `count`: Maximum results (default: 10)

**Filters**:
- Only shared notices (`status = 3`)
- Last 6 months only
- Within specified radius

**Ranking**:
- Ordered by `diff` (average distance from point)
- Closer + more frequent = higher priority

### Integration

```ruby
# In AnalyzerJob#handle_exif
if notice.data_sets.proximity.blank?
  result = Notice.nearest_tbnrs(*coords)
  if result.present?
    proximity_data_set = notice.data_sets.create!(
      data: result,
      kind: :proximity,
      keyable: photo
    )
    if notice.user.from_proximity?
      notice.tbnr ||= proximity_data_set.tbnrs.first
    end
  end
end
```

### DataSet Access

```ruby
# In DataSet#tbnrs
def tbnrs
  case kind
  when "proximity"
    data.map { |it| it["tbnr"] }.compact
  end
end
```

---

## 8. User History Analysis (Auto-Fill)

### Purpose
Pre-fill notice details based on the same vehicle being reported previously.

### Implementation

**File**: `app/models/notice.rb`

```ruby
def apply_favorites(registrations)
  other = user
    .notices
    .shared
    .where(
      "REPLACE(registration, ' ', '') IN(?)",
      registrations.map { |registration| registration.gsub(/\s/, "") }
    )
    .order(created_at: :desc)
    .first

  if other
    self.registration = other.registration
    self.brand = other.brand if !brand? && other.brand?
    self.color = other.color if !color? && other.color?
    self.location = other.location if !location? && other.location?
    self.tbnr = other.tbnr if !tbnr? && other.tbnr?
    self.flags = other.flags if !flags? && other.flags?
    self.note = other.note if !note? && other.note?
  end
end
```

### Algorithm

1. **Search**: Find user's previous shared notices with same registration
2. **Match**: Normalize plates (remove spaces) for comparison
3. **Select**: Most recent match
4. **Copy**: Transfer known fields (only if not already set)

### Copied Fields

| Field | Purpose |
|-------|---------|
| `registration` | License plate (normalized) |
| `brand` | Vehicle manufacturer |
| `color` | Vehicle color |
| `location` | Street address |
| `tbnr` | Violation type |
| `flags` | Vehicle empty, hazard lights, etc. |
| `note` | Additional notes |

### User Control

Enabled via autosuggest bitfield:
```ruby
user.autosuggest_from_history?  # true/false
```

---

## 9. Alternative: car-ml Service

### Purpose
Backup ML service using YOLO for license plate and vehicle recognition.

### Implementation

**File**: `app/lib/annotator.rb`

```ruby
def annotate_yolo(key = "ydmE3qL1CT32rH6hunWtxCzx")
  client = HTTP.use(logging: { logger: Rails.logger }).timeout(10)
  headers = { "Content-Type" => "application/json" }
  url = ENV.fetch("CAR_ML_URL", "https://weg-li-car-ml.onrender.com")

  response = client.post(url, headers: headers, json: {
    google_cloud_urls: [key]
  })

  if response.status.success?
    JSON.parse(response.body)
  else
    raise HTTP::ResponseError.new,
          "Request failed with status #{response.status}: #{response.body}"
  end
end
```

### Response Format

```json
{
  "suggestions": {
    "license_plate_number": ["B-AB 1234"],
    "make": ["Mercedes-Benz"],
    "color": ["silver"]
  }
}
```

### DataSet Integration

**File**: `app/models/data_set.rb`

```ruby
def registrations
  case kind
  when "car_ml"
    data["suggestions"]["license_plate_number"]
  end
end

def brands
  case kind
  when "car_ml"
    data["suggestions"]["make"]
  end
end

def colors
  case kind
  when "car_ml"
    data["suggestions"]["color"]
  end
end
```

### Configuration

```bash
CAR_ML_URL=https://weg-li-car-ml.onrender.com  # Default
# Or custom deployment
CAR_ML_URL=https://your-car-ml-instance.com
```

### When to Use

- Google Cloud Vision API unavailable
- Cost optimization (car-ml may be cheaper for high volume)
- Specific accuracy requirements
- Self-hosted deployment

---

## 10. Analysis Pipeline Orchestration

### Purpose
Coordinate all AI functions in sequence, store results, and update notices.

### Implementation

**File**: `app/jobs/analyzer_job.rb`

```ruby
class AnalyzerJob < ApplicationJob
  retry_on EXIFR::MalformedJPEG, attempts: 5, wait: :polynomially_longer
  retry_on ActiveStorage::FileNotFoundError, attempts: 5, wait: :polynomially_longer
  discard_on Encoding::UndefinedConversionError
  discard_on ActiveRecord::RecordInvalid

  def perform(notice)
    analyze(notice)
  end

  def analyze(notice)
    handle_exif(notice)    # Phase 1: Metadata
    handle_vision(notice)  # Phase 2: Vision AI

    notice.status = :open
    notice.save_incomplete!
  end
end
```

### Processing Phases

**Phase 1: EXIF Analysis** (`handle_exif`)
1. Extract EXIF from each photo
2. Create `exif` DataSet
3. If GPS coordinates found:
   - Reverse geocode → `geocoder` DataSet
   - Find nearby violations → `proximity` DataSet
4. Auto-fill location, dates (if `from_exif` enabled)

**Phase 2: Vision Analysis** (`handle_vision`)
1. For each photo:
   - Call Google Vision API
   - Create `google_vision` DataSet
2. Extract registrations (license plates)
3. If found:
   - Look up user history (`apply_favorites`)
   - Auto-fill registration, brand, color (if enabled)
4. Break after first successful plate detection

### Error Handling

| Error | Strategy |
|-------|----------|
| `MalformedJPEG` | Retry 5 times with polynomial backoff |
| `FileNotFoundError` | Retry 5 times (file may not be uploaded yet) |
| `UndefinedConversionError` | Discard (encoding issue) |
| `RecordInvalid` | Discard (data integrity issue) |

### User Preference Checks

```ruby
notice.user.from_exif?        # Use GPS/timestamp from EXIF
notice.user.from_recognition? # Use Vision API results
notice.user.from_proximity?   # Use nearby violation suggestions
notice.user.from_history?     # Use previous reports for same vehicle
```

---

## 11. DataSet Storage Model

### Purpose
Polymorphic storage for all AI analysis results.

### Implementation

**File**: `app/models/data_set.rb`

```ruby
class DataSet < ApplicationRecord
  belongs_to :setable, polymorphic: true  # Usually Notice
  belongs_to :keyable, polymorphic: true  # Usually Photo

  enum :kind, {
    google_vision: 0,
    exif: 1,
    car_ml: 2,
    geocoder: 3,
    proximity: 4
  }
end
```

### Schema

```sql
CREATE TABLE data_sets (
  id bigint PRIMARY KEY,
  setable_type varchar,    -- "Notice"
  setable_id bigint,       -- notice.id
  keyable_type varchar,    -- "ActiveStorage::Attachment"
  keyable_id bigint,       -- photo.id
  kind integer,            -- 0=google_vision, 1=exif, etc.
  data jsonb,              -- Flexible JSON storage
  created_at timestamp,
  updated_at timestamp
);
```

### Data Extraction Methods

| Kind | Method | Returns |
|------|--------|---------|
| `google_vision` | `registrations` | Ranked license plates |
| `google_vision` | `brands` | Ranked brand names |
| `google_vision` | `colors` | Ranked color names |
| `exif` | `coords` | [latitude, longitude] |
| `exif` | `date_time` | Timestamp |
| `geocoder` | `address` | Address hash |
| `proximity` | `tbnrs` | Array of violation codes |
| `car_ml` | `registrations` | License plates |
| `car_ml` | `brands` | Brand names |
| `car_ml` | `colors` | Color names |

---

## Summary

### AI Function Matrix

| Function | Service | Purpose | Accuracy | Cost |
|----------|---------|---------|----------|------|
| License Plate OCR | Google Vision | Extract plate text | High | $1.50/1000 |
| Plate Parsing | Local regex | Validate German format | Very High | Free |
| Brand Recognition | Google Vision + DB | Identify manufacturer | Medium | Included |
| Color Detection | Google Vision + Color gem | Identify vehicle color | Medium | Included |
| EXIF Extraction | Local EXIFR | Get GPS/timestamp | Very High | Free |
| Geocoding | Nominatim/OpenCage | Convert coords to address | High | Free/Low |
| Proximity Analysis | PostGIS | Suggest violations | Very High | Free |
| User History | PostgreSQL | Auto-fill repeat vehicles | Very High | Free |
| Alternative ML | car-ml | Backup recognition | Medium-High | Variable |

### Key Design Principles

1. **Multi-source fusion**: Combine multiple AI sources for accuracy
2. **Graceful degradation**: Work with partial data if some sources fail
3. **User control**: Bitfields allow users to enable/disable each source
4. **Result caching**: Store AI results to avoid repeated API calls
5. **Cost optimization**: Single analysis per photo, cached results
6. **German-specific**: Optimized for German license plates and locations

---

**Last Updated**: November 2024

**Documentation Version**: 1.0
