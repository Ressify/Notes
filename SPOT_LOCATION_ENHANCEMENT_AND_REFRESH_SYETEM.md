# Implement Spots Filter by Open Now - Location-Based Search Enhancement & Spot Refresh System

## 🎯 Overview

This PR enhances the spots search functionality by making location parameters required and adding spatial filtering to text search queries. It also includes improvements to opening hours serialization, code quality fixes, and implements a comprehensive spot data refresh system to keep restaurant information up-to-date.

## ✨ Key Changes

### 1. Location-Based Text Search

- **Made location parameters required** for `/spots/search` endpoint
- **Added spatial filtering** to text search (5km radius)
- Results are now filtered by both text match AND location proximity

### 2. Code Quality Improvements

- ✅ Fixed hardcoded table names → now uses `cls.__tablename__`
- ✅ Fixed validation bug → properly handles coordinates like (0.0, 0.0)
- ✅ Removed debug code (JSON file saving)

### 3. Google Places API Enhancements

- 📅 **Enhanced opening hours serialization**:
  - Added `current_opening_hours.periods` (special hours/exceptions for next 7 days)
  - Added `regular_opening_hours.periods` (standard weekly schedule)
  - Removed `open_now` and `weekday_descriptions` (frontend should calculate `open_now` from periods)
  - Periods include time in HH:mm format, minutes from midnight, and midnight-spanning detection
- 🔍 Improved location bias handling (500m for API, 5km for DB)
- ✅ Added proper validation and error handling

### 4. Spot Data Refresh System (NEW)

#### Phase 1: Manual Refresh Command

- ✅ **New Flask CLI command**: `flask refresh-spots`
  - Refresh spots older than 30 days (default)
  - Refresh specific spots by ID
  - Custom age threshold support
  - Batch processing with configurable batch size
  - Comprehensive statistics reporting

#### Phase 2: Automatic Lazy Refresh

- ✅ **30-day refresh cycle**: Spots automatically refresh when accessed if:
  - `last_refreshed_at` is `null` (never refreshed)
  - `last_refreshed_at` is older than 30 days
- ✅ **Non-blocking background refresh**: Uses threading to refresh spots without blocking API responses
- ✅ **Integrated into all spot endpoints**:
  - `GET /spots` (list spots)
  - `GET /spots/search` (search spots)
  - `GET /spots/nearby` (nearby spots)
  - `GET /spots/recommended` (recommended spots)

#### Database Schema Changes

- ✅ Added `regular_opening_hours` field (JSONB) to store structured opening hours
- ✅ Added `last_refreshed_at` field (DateTime, timezone-aware) to track refresh timestamps
- ✅ Migrations: `b3ce48f98e49` and `2460e4b9ba52`

## ⚠️ Breaking Changes

### 1. Location Parameters Required

The `/spots/search` endpoint now **requires** `latitude` and `longitude` parameters.

**Before:**

```
GET /spots/search?q=restaurant
```

**After:**

```
GET /spots/search?q=restaurant&latitude=43.6500418&longitude=-79.3916043
```

### 2. Opening Hours Data Structure Changed

Opening hours response structure has changed:

**Before:**

```json
{
  "current_opening_hours": {
    "open_now": true,
    "weekday_descriptions": ["Monday: 9:00 AM – 5:00 PM", ...]
  }
}
```

**After:**

```json
{
  "current_opening_hours": {
    "periods": [
      {
        "open": {
          "day": 0,
          "time": "0900"
        },
        "close": {
          "day": 0,
          "time": "2200"
        }
      }
    ]
  },
  "regular_opening_hours": {
    "periods": [
      {
        "open": {
          "day": 0,
          "time": "0900"
        },
        "close": {
          "day": 0,
          "time": "2200"
        }
      }
    ]
  }
}
```

**Note**: Frontend must now calculate `open_now` using periods data and local timezone.

## 📋 Files Changed

### Core Models & Services

- `ressy_backend/models/spot.py` - Added spatial filtering, fixed table names, added `regular_opening_hours` and `last_refreshed_at` fields
- `ressy_backend/services/spot_service.py` - Integrated lazy refresh triggers
- `ressy_backend/services/spot_refresh_service.py` - **NEW** - Core refresh logic (Phase 1 & 2)
- `ressy_backend/services/googleplaces_services.py` - Enhanced opening hours, improved location bias

### API Routes & Schemas

- `ressy_backend/routes/spot.py` - Updated validation, made location required, integrated lazy refresh
- `ressy_backend/schemas/spot.py` - Made latitude/longitude required

### CLI Commands

- `ressy_backend/commands/refresh_spots.py` - **NEW** - Manual refresh command (Phase 1)
- `ressy_backend/app.py` - Registered new refresh command

### Database Migrations

- `migrations/versions/b3ce48f98e49_add_migration_message.py` - Added `regular_opening_hours` field
- `migrations/versions/2460e4b9ba52_add_last_refreshed_at_to_spots.py` - Added `last_refreshed_at` field

### Documentation

- `README.md` - Added refresh command examples
- `SPOT_REFRESH_SYSTEM.md` - **NEW** - Comprehensive refresh system documentation

## 🚀 New Features

### Manual Spot Refresh

Refresh spot data manually using the Flask CLI:

```bash
# Refresh spots older than 30 days (default)
docker compose exec ressy-backend flask refresh-spots

# Refresh specific spots by ID
docker compose exec ressy-backend flask refresh-spots --spot-ids "1,5,10"

# Refresh spots older than 60 days
docker compose exec ressy-backend flask refresh-spots --older-than-days 60

# Custom limit and batch size
docker compose exec ressy-backend flask refresh-spots --limit 500 --batch-size 20
```

### Automatic Background Refresh

Spots are automatically refreshed in the background when accessed through API endpoints if they are stale (null `last_refreshed_at` or older than 30 days). This happens non-blocking, so API responses return immediately with current data while refresh happens asynchronously.

## 🔄 Migration Instructions

Run the following migrations to add the new fields:

```bash
docker compose exec ressy-backend flask db upgrade
```

This will apply:

1. `b3ce48f98e49` - Adds `regular_opening_hours` field
2. `2460e4b9ba52` - Adds `last_refreshed_at` field

## 📊 What Gets Refreshed

When a spot is refreshed (manually or automatically), the following fields are updated from Google Places API:

- `name` - Restaurant/spot name
- `address` - Full address
- `types` - Business type categories
- `rating` - Average user rating
- `photos` - Photo metadata array
- `regular_opening_hours` - Weekly opening hours schedule
- `provider` - Provider metadata
- `info` - Additional information
- `location` - PostGIS point (latitude/longitude)
- `search_vector` - Full-text search vector (if name/address/types change)
- `last_refreshed_at` - Timestamp of refresh

## 🧪 Testing

### Manual Refresh Testing

```bash
# Test basic refresh
docker compose exec ressy-backend flask refresh-spots --limit 10

# Test specific spot refresh
docker compose exec ressy-backend flask refresh-spots --spot-ids "1"

# Test age-based filtering
docker compose exec ressy-backend flask refresh-spots --older-than-days 7
```

### API Testing

1. **Test location requirement**:

   ```bash
   # Should fail without location
   curl -X GET "http://localhost:5000/spots/search?q=restaurant" \
     -H "Authorization: Bearer <token>"

   # Should succeed with location
   curl -X GET "http://localhost:5000/spots/search?q=restaurant&latitude=43.65&longitude=-79.39" \
     -H "Authorization: Bearer <token>"
   ```

2. **Test lazy refresh**:
   - Access a spot with `last_refreshed_at = null` or older than 30 days
   - Check logs for background refresh activity
   - Verify `last_refreshed_at` is updated after refresh

## 📝 Additional Notes

- The refresh system uses background threading to avoid blocking API responses
- Background refresh errors are logged but don't affect API responses
- The 30-day refresh cycle can be modified by changing `REFRESH_CYCLE_DAYS` in `spot_refresh_service.py`
- All refresh operations respect Google Places API rate limits
- See `SPOT_REFRESH_SYSTEM.md` for detailed documentation on the refresh system

## 🔮 Future Enhancements (Phase 3)

Planned but not yet implemented:

- Scheduled background tasks for proactive refresh
- Task queue integration (Celery)
- Priority-based refresh for popular spots
- Enhanced monitoring and metrics
