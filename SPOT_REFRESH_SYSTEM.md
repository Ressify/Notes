# Spot Refresh System

## Overview

The Spot Refresh System ensures that restaurant/spot data in the database stays up-to-date with the latest information from Google Places API. This system addresses the need for fresh data including:

- **Opening Hours**: Regular weekly schedules and special hours
- **Ratings**: Current user ratings and reviews
- **Photos**: Updated photo collections
- **Business Information**: Name, address, types, and location changes
- **Metadata**: Provider information and additional details

The system operates in two phases:
1. **Phase 1**: Manual refresh via Flask CLI command
2. **Phase 2**: Automatic lazy refresh on spot access (30-day cycle)

## Database Schema Changes

### New Fields

#### `last_refreshed_at`
- **Type**: `DateTime(timezone=True)`
- **Nullable**: `True`
- **Purpose**: Tracks when a spot's data was last refreshed from Google Places API
- **Migration**: `2460e4b9ba52_add_last_refreshed_at_to_spots.py`

#### `regular_opening_hours`
- **Type**: `JSONB`
- **Nullable**: `True`
- **Purpose**: Stores structured opening hours data in periods format
- **Structure**: 
  ```json
  {
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
  ```
- **Migration**: `b3ce48f98e49_add_migration_message.py`

## Phase 1: Manual Refresh Command

### Flask CLI Command

The manual refresh command allows administrators to refresh spot data on-demand or in bulk.

**Command**: `flask refresh-spots`

### Command Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--limit` | `int` | `100` | Maximum number of spots to refresh |
| `--batch-size` | `int` | `10` | Number of spots to process before committing to database |
| `--spot-ids` | `string` | `None` | Comma-separated list of specific spot IDs to refresh (e.g., "1,2,3") |
| `--older-than-days` | `int` | `30` | Only refresh spots that haven't been updated in X days (or never refreshed) |

### Usage Examples

#### Refresh spots older than 30 days (default)
```bash
docker-compose exec ressy-backend flask refresh-spots
```

#### Refresh specific spots by ID
```bash
docker-compose exec ressy-backend flask refresh-spots --spot-ids "1,5,10,25"
```

#### Refresh spots older than 60 days
```bash
docker-compose exec ressy-backend flask refresh-spots --older-than-days 60
```

#### Refresh with custom batch size and limit
```bash
docker-compose exec ressy-backend flask refresh-spots --limit 500 --batch-size 20
```

### Output Format

The command provides a summary of the refresh operation:

```
Found 50 spots to refresh.
Processing in batches of 10...

==================================================
Refresh Summary:
  Total processed: 50
  Successfully updated: 48
  Skipped (no data): 2
  Errors: 0
==================================================
```

**Statistics:**
- **Total processed**: Total number of spots attempted
- **Successfully updated**: Spots that were refreshed successfully
- **Skipped (no data)**: Spots where Google Places API returned no data
- **Errors**: Spots that encountered errors during refresh

## Phase 2: Lazy Refresh (Automatic)

### Overview

Phase 2 implements automatic background refresh when spots are accessed through API endpoints. This ensures data freshness without manual intervention.

### Refresh Cycle

- **Cycle Duration**: 30 days (`REFRESH_CYCLE_DAYS = 30`)
- **Configuration**: Defined in `ressy_backend/services/spot_refresh_service.py`

### When Spots Are Refreshed

A spot is automatically refreshed if:
1. `last_refreshed_at` is `null` (never been refreshed)
2. `last_refreshed_at` is older than 30 days

### API Endpoints That Trigger Lazy Refresh

The following endpoints automatically trigger lazy refresh for returned spots:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/spots` | `GET` | List nearby spots with optional text search |
| `/spots/search` | `GET` | Full-text search across all spots |
| `/spots/nearby` | `GET` | Get spots nearby a given location |
| `/spots/recommended` | `GET` | Get recommended spots for a location |

### How It Works

1. **Non-Blocking**: Refresh happens in a background thread, so API responses return immediately
2. **Background Threading**: Uses Python's `threading` module with daemon threads
3. **App Context**: Each background thread creates its own Flask application context
4. **Re-query Safety**: The spot is re-queried in the background thread to ensure data consistency

**Flow Diagram:**

```
API Request → Get Spots from DB → Check last_refreshed_at
                                    ↓
                          [Needs Refresh?]
                         /              \
                       Yes              No
                        ↓                ↓
            Start Background Thread    Return Spots
                        ↓
            Fetch from Google Places API
                        ↓
            Update Spot in Database
                        ↓
            Commit Transaction
```

### Important Notes

- **Immediate Response**: API requests return immediately with current data (may be stale)
- **Background Update**: Refresh happens asynchronously after the response is sent
- **Next Request**: Subsequent requests will see the updated data (if refresh completed)
- **No Blocking**: The refresh process does not slow down API responses
- **Error Handling**: Background refresh errors are logged but don't affect the API response

## What Gets Refreshed

When a spot is refreshed, the following fields are updated from Google Places API:

| Field | Type | Description |
|-------|------|-------------|
| `name` | `string` | Restaurant/spot name |
| `address` | `string` | Full address |
| `types` | `array[string]` | Business type categories |
| `rating` | `float` | Average user rating |
| `photos` | `array[object]` | Photo metadata array |
| `regular_opening_hours` | `object` | Weekly opening hours schedule |
| `provider` | `object` | Provider metadata (Google Places) |
| `info` | `object` | Additional information |
| `location` | `Geometry` | PostGIS point (latitude/longitude) |
| `search_vector` | `TSVECTOR` | Full-text search vector (updated if name/address/types change) |
| `last_refreshed_at` | `DateTime` | Timestamp of refresh |

## Technical Implementation

### Key Files

#### `ressy_backend/services/spot_refresh_service.py`
Core refresh service containing:
- `should_refresh_spot(spot)`: Determines if a spot needs refreshing
- `lazy_refresh_spot(spot)`: Triggers non-blocking background refresh
- `refresh_spot_data(spot)`: Performs the actual refresh operation
- `refresh_spots_batch(spots, batch_size)`: Batch refresh with commit management
- `REFRESH_CYCLE_DAYS`: Configuration constant (30 days)

#### `ressy_backend/commands/refresh_spots.py`
Flask CLI command implementation:
- Command registration and option parsing
- Query building with filters
- Statistics collection and reporting

#### `ressy_backend/services/spot_service.py`
Service layer integration:
- `get_or_create_spots_from_google()`: Triggers lazy refresh for returned spots

#### `ressy_backend/routes/spot.py`
API endpoint integration:
- All spot listing/search endpoints call `lazy_refresh_spot()` for each returned spot

### Core Functions

#### `should_refresh_spot(spot: Spot) -> bool`
Checks if a spot needs refreshing based on `last_refreshed_at` timestamp.

**Logic:**
- Returns `True` if `last_refreshed_at` is `None`
- Returns `True` if `last_refreshed_at < (now - 30 days)`
- Returns `False` otherwise

#### `lazy_refresh_spot(spot: Spot) -> None`
Triggers a non-blocking background refresh for a spot.

**Process:**
1. Checks if refresh is needed via `should_refresh_spot()`
2. Validates spot has `provider_id`
3. Starts background thread with Flask app context
4. Re-queries spot in background thread
5. Calls `refresh_spot_data()` if still needed
6. Commits transaction

#### `refresh_spot_data(spot: Spot) -> bool`
Performs the actual refresh operation.

**Process:**
1. Fetches latest data from Google Places API using `provider_id`
2. Updates all relevant fields from API response
3. Updates `location` using PostGIS functions
4. Updates `search_vector` if name/address/types changed
5. Sets `last_refreshed_at` to current UTC timestamp
6. Returns `True` on success, `False` on failure

#### `refresh_spots_batch(spots: list[Spot], batch_size: int) -> dict`
Refreshes multiple spots in batches with error handling.

**Returns:**
```python
{
    "total": int,      # Total spots processed
    "updated": int,    # Successfully refreshed
    "errors": int,     # Errors encountered
    "skipped": int     # Skipped (no data from API)
}
```

## Usage Examples

### Manual Refresh Scenarios

#### Scenario 1: Initial Backfill
Refresh all spots that have never been refreshed:
```bash
docker-compose exec ressy-backend flask refresh-spots --older-than-days 0 --limit 1000
```

#### Scenario 2: Weekly Maintenance
Refresh spots older than 7 days (more aggressive refresh):
```bash
docker-compose exec ressy-backend flask refresh-spots --older-than-days 7 --limit 500
```

#### Scenario 3: Specific Spot Update
Refresh a specific spot that was reported as having outdated information:
```bash
docker-compose exec ressy-backend flask refresh-spots --spot-ids "123"
```

### API Request Examples

#### Example 1: List Spots (Triggers Lazy Refresh)
```http
GET /spots?latitude=43.6500418&longitude=-79.3916043&limit=5&offset=0
Authorization: Bearer <token>
```

**Response:** Returns immediately with current spot data. If any spots are stale (null `last_refreshed_at` or >30 days old), they are queued for background refresh.

#### Example 2: Search Spots (Triggers Lazy Refresh)
```http
GET /spots/search?q=sushi&latitude=43.6500418&longitude=-79.3916043&limit=10
Authorization: Bearer <token>
```

**Response:** Returns search results immediately. Stale spots are refreshed in the background.

### Expected Behavior

1. **First Request**: 
   - Spots with `last_refreshed_at = null` trigger background refresh
   - API returns immediately with existing data
   - Background thread fetches fresh data and updates database

2. **Subsequent Requests** (within 30 days):
   - No refresh triggered (data is fresh)
   - API returns current data

3. **After 30 Days**:
   - Spots older than 30 days trigger background refresh
   - API returns current data immediately
   - Background thread updates with fresh data

## Future Enhancements (Phase 3)

### Scheduled Background Tasks

Phase 3 would implement scheduled background tasks to proactively refresh spots without requiring user access. This could include:

- **Cron-based Scheduling**: Periodic refresh of all spots or high-priority spots
- **Task Queue Integration**: Using Celery or similar for distributed task processing
- **Priority-based Refresh**: Refresh popular/frequently accessed spots more often
- **Rate Limiting**: Respect Google Places API rate limits
- **Monitoring**: Track refresh success rates and API usage

**Note:** Phase 3 is planned but not yet implemented.

## Error Handling

### Background Refresh Errors

Errors during background refresh are:
- Logged with full stack traces
- Do not affect API responses
- Do not block other spot refreshes
- Database transactions are rolled back on error

### Manual Refresh Errors

Errors during manual refresh:
- Are logged with details
- Increment the `errors` counter in statistics
- Do not stop batch processing
- Each spot's transaction is independent

### Common Error Scenarios

1. **Missing `provider_id`**: Spot cannot be refreshed (skipped)
2. **Google Places API Error**: Network or API errors (logged, skipped)
3. **No Data Returned**: API returns empty response (skipped)
4. **Database Error**: Transaction rollback (logged, error counter incremented)

## Configuration

### Refresh Cycle Duration

The refresh cycle can be modified by changing `REFRESH_CYCLE_DAYS` in:
```
ressy_backend/services/spot_refresh_service.py
```

**Current Value**: `30` days

**Note**: Changing this value affects when spots are considered stale for lazy refresh. Manual refresh commands can override this with `--older-than-days`.

## Migration History

- **`2460e4b9ba52`**: Added `last_refreshed_at` field to `spots` table
- **`b3ce48f98e49`**: Added `regular_opening_hours` field to `spots` table

To apply migrations:
```bash
docker-compose exec ressy-backend flask db upgrade
```

