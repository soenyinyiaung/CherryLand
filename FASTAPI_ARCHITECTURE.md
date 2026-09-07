# FastAPI Backend Architecture

## Overview

The FastAPI Backend is a Python-based HTTP API service for parsing delivery order text into structured data. It serves as the order parsing engine for the CherryLand delivery platform, converting natural language order messages (in Myanmar and English) into structured delivery information including pickup locations, delivery details, fees, and complexity metrics. The service integrates with a relational database to persist parsed order data and supports multi-rider order assignment through mention tracking.

## Technology Stack

- **Language**: Python 3.x
- **Web Framework**: FastAPI (modern async web framework)
- **Database**: Relational database (via PyMySQL driver)
- **Data Validation**: Pydantic (data models and validation)
- **Text Processing**: Python regex (re module)
- **Environment**: python-dotenv (environment variable management)
- **ASGI Server**: Uvicorn (production server)

## Directory Structure

```
fastapi-backend/
├── Main application entry point
├── Requirements file
└── Dockerfile
```

The application follows a single-file architecture pattern with all components organized within the main.py file for simplicity and deployment ease.

## Core Components

### 1. Application Entry Point (main.py)

The main function initializes the FastAPI application and configures middleware:

- **Environment Loading**: Loads environment variables from .env file
- **Logging Configuration**: Sets up structured logging with timestamp, level, and message
- **FastAPI Instance**: Creates FastAPI app with title and version
- **CORS Middleware**: Configures cross-origin resource sharing for all origins
- **Database Configuration**: Sets up relational database connection parameters from environment variables
- **Route Registration**: Registers API endpoints for order parsing

### 2. Database Layer

The database layer manages relational database connectivity with context manager pattern:

#### Connection Configuration

- **Host**: Configured via DB_HOST environment variable
- **Port**: Configured via DB_PORT environment variable
- **User**: Configured via DB_USER environment variable
- **Password**: Configured via DB_PASS environment variable
- **Database Name**: Configured via DB_NAME environment variable
- **Charset**: UTF8MB4 for Myanmar language support
- **Cursor Class**: Dictionary cursor for row-as-dict results
- **Connection Timeout**: 10 seconds
- **Autocommit**: Disabled for transaction control

#### Database Context Manager

The get_db() context manager provides automatic transaction management:
- **Connection Opening**: Establishes relational database connection on entry
- **Commit**: Commits transaction on successful completion
- **Rollback**: Rolls back transaction on exception
- **Connection Closing**: Closes connection on exit
- **Error Handling**: Catches and re-raises exceptions after rollback

### 3. Data Models (Pydantic)

Pydantic models provide data validation and serialization:

#### OrderInput Model

Input model for order parsing requests:
- **orderId**: Integer order identifier
- **submitted_by**: String username of order submitter
- **text**: String order text content (required, non-empty)
- **status**: Optional string status (default: "pending")
- **Validation**: Text field validated to prevent empty strings

#### PickupDetails Model

Model for pickup location information:
- **shop_name**: String shop or location name
- **shop_ph**: List of phone numbers (default: empty list)
- **details**: List of item details (default: empty list)
- **shop_notes**: String additional notes (default: empty string)
- **type**: String pickup type (pickup, food, shopping, gate, passenger)
- **complexity**: Integer complexity score (default: 2)

#### DeliveryDetails Model

Model for delivery information:
- **name**: String recipient name (default: empty string)
- **addr**: String delivery address (default: empty string)
- **ph**: List of phone numbers (default: empty list)
- **notes**: String delivery notes (default: empty string)

#### ParsedOrder Model

Model for parsed order result:
- **fee**: Integer delivery fee (default: 0)
- **ways**: Integer number of delivery ways (default: 1)
- **comp**: Integer total complexity score (default: 0)
- **pickups**: List of PickupDetails (default: empty list)
- **deli**: DeliveryDetails object
- **warnings**: List of parse warning messages (default: empty list)

### 4. Regex Patterns (RX Class)

The RX class contains compiled regex patterns for text parsing, optimized for performance through pre-compilation:

#### Digit Conversion

- **BURMESE_DIGITS**: Translation table for Myanmar digits to Arabic digits (၀-၉ → 0-9)

#### Phone Number Patterns

- **PHONE**: Matches Myanmar phone numbers in various formats (09..., +959..., 959..., ၀၉...)
- **PHONE_CLEAN**: Removes spaces, dashes, and dots from phone numbers
- Supports 8-9 digit Myanmar numbers with country/prefix variations
- Handles Burmese digit notation

#### Section Splitting

- **SECTION_SPLIT**: Splits pickup and delivery sections using keywords (To, ပို့ရန်, ပေးပို့ရန်, လိပ်စာ, arrows)
- Case-insensitive matching for English and Myanmar keywords
- Supports various separator formats (colon, dash, arrow)

#### Pickup Detection

- **PICKUP_KW**: Identifies pickup keywords in Myanmar and English
- **PICKUP_KW_STRIP**: Removes pickup keywords from text for name extraction
- **SHOP_SUFFIX_STRIP**: Removes shop suffixes for clean name extraction

#### Noise Filtering

- **HEADER**: Matches order headers (new order, customer order, dividers)
- **MENTION**: Matches @username mentions at line start
- **DIVIDER**: Matches horizontal dividers (===, ---, ___)
- **EMOJI_ONLY**: Matches lines containing only emojis

#### Fee and Ways Extraction

- **FEE**: Extracts delivery fee with Myanmar and English keywords
- **WAYS**: Extracts number of ways (e.g., "2 ways")
- **PRICE**: Extracts price amounts with currency indicators

#### Item Detection

- **ITEM_UNIT**: Identifies items with quantity units (တုံး, စည်း, ဘူး, pcs, box, etc.)
- Supports Myanmar and English unit names
- Used for distinguishing items from notes

#### Type Classification

- **TYPE_PASSENGER**: Identifies passenger transport orders
- **TYPE_SHOPPING**: Identifies shopping delivery orders
- **TYPE_GATE**: Identifies bus gate pickup orders
- **TYPE_FOOD**: Identifies food delivery orders with extensive food vocabulary

#### Delivery Note Detection

- **DELI_NOTE_KW**: Identifies delivery note keywords (msg, ချထားပေး, ဖုန်းဆက်ပေး, etc.)
- **MANUAL_NOTES**: Identifies manual pickup notes to exclude from item detection
- **EMOJI**: Detects emoji characters for note classification

#### Address and Name Detection

- **ADDR_KW**: Identifies address keywords (လမ်း, ရပ်ကွက်, မြို့, တိုက်, etc.)
- Used for distinguishing addresses from names

#### Mention Extraction

- **MENTION_EXTRACT**: Extracts @username mentions from text
- Captures alphanumeric usernames with underscores

### 5. Helper Utilities

Utility functions provide common text processing operations:

#### Digit Conversion

- **to_en()**: Converts Myanmar digits to Arabic digits using translation table

#### Phone Number Processing

- **normalize_phone()**: Normalizes phone numbers to 09XXXXXXXXX format
- Handles +959 and 959 prefixes
- Removes spaces, dashes, and dots
- **extract_phones()**: Extracts unique phone numbers from text and returns cleaned text

#### Text Cleaning

- **clean_symbol()**: Removes leading/trailing symbols and whitespace
- **is_noise()**: Identifies noise lines (headers, mentions, dividers, emojis)
- **extract_mentions()**: Extracts unique @mentions from text in first-seen order

#### Type Classification

- **classify_pickup_type()**: Classifies pickup type based on text and next line
- Returns type string and complexity score
- Supports passenger, shopping, gate, food, and default pickup types

#### Number Conversion

- **smart_int()**: Safely converts string to integer with comma removal
- Returns 0 on conversion failure

### 6. Core Parser (DeliveryParser)

The DeliveryParser class implements the main parsing logic:

#### Parse Method

Main parsing method that orchestrates the entire parsing process:

1. **Pre-cleaning**: Strips whitespace from input text
2. **Global Extractions**: Extracts fee and ways from entire text
3. **Section Splitting**: Splits text into pickup and delivery sections
4. **Fallback Handling**: Handles missing delivery section with inline "To" detection
5. **Pickup Parsing**: Parses pickup section with _parse_pickups method
6. **Delivery Parsing**: Parses delivery section with _parse_delivery method
7. **Complexity Calculation**: Sums complexity scores from all pickups
8. **Result Assembly**: Returns ParsedOrder with all extracted data

#### Pickup Parser (_parse_pickups)

Parses pickup section to extract multiple pickup locations:

- **Line Processing**: Iterates through lines, skipping noise lines
- **Phone Extraction**: Extracts phone numbers from each line
- **New Pickup Detection**: Identifies new pickup locations using keywords or context
- **Shop Name Extraction**: Cleans and extracts shop name from pickup line
- **Type Classification**: Classifies pickup type based on keywords
- **Item Detection**: Identifies items using unit patterns, excluding manual notes
- **Note Accumulation**: Accumulates non-item text as shop notes
- **Fallback Handling**: Creates unknown pickup if no pickups detected
- **Multiple Pickups**: Supports multiple pickup locations in single order

#### Delivery Parser (_parse_delivery)

Parses delivery section to extract delivery information:

- **Line Processing**: Iterates through lines, skipping noise lines
- **Phone Extraction**: Extracts all phone numbers from delivery section
- **Fee Line Skipping**: Skips pure fee lines to avoid confusion
- **Emoji Detection**: Treats lines with emojis as notes
- **Name Detection**: Identifies short lines without address keywords as person names
- **Address Accumulation**: Accumulates address parts from remaining lines
- **Note Accumulation**: Accumulates delivery notes with keywords or emojis
- **Warning Generation**: Warns if no phone numbers found
- **Result Assembly**: Returns DeliveryDetails with name, address, phones, and notes

### 7. API Endpoints

The service exposes HTTP endpoints for order parsing:

#### Parse Only Endpoint

**POST endpoint for parse-only operation**

Quick parsing without database save, useful for frontend preview:

- **Input**: OrderInput model with orderId, submitted_by, and text
- **Processing**: Calls DeliveryParser.parse() to extract structured data
- **Output**: ParseOnlyResponse with parsed ParsedOrder
- **Use Case**: Real-time preview before order submission
- **Database**: No database operations performed

#### Parse and Save Endpoint

**POST endpoint for parse-and-save operation**

Parses order and saves to relational database with multi-mention support:

- **Input**: OrderInput model with orderId, submitted_by, text, and status
- **Processing**:
  1. Parses order text using DeliveryParser.parse()
  2. Extracts mentions from text (@username references)
  3. Falls back to submitted_by if no mentions found
  4. Serializes pickup details, delivery phones, and warnings to JSON
  5. Queries existing order rows for this orderId
  6. Calculates mention set differences (to_add, to_keep, to_delete)
  7. Deletes rows for users no longer mentioned
  8. Updates rows for users still mentioned with new data
  9. Inserts rows for newly mentioned users
- **Output**: Success response with mention changes and parsed data
- **Database Operations**: DELETE, UPDATE, INSERT on orders table
- **Transaction**: All operations in single transaction with rollback on error
- **Error Handling**: Returns 409 for duplicate errors, 500 for database errors
- **Logging**: Logs mention changes (added, updated, removed)

#### Health Check Endpoint

**GET endpoint for health check**

Service health and database connectivity check:

- **Processing**: Attempts database query with connection
- **Output**: Status with database connection state
- **Use Case**: Health monitoring and load balancer checks

## Data Flow

### Parse Only Flow

1. **Client sends POST request** with OrderInput containing orderId, submitted_by, and text
2. **FastAPI validates** input using Pydantic model (text non-empty check)
3. **DeliveryParser.parse()** processes text through regex patterns
4. **Global extraction** extracts fee and ways from entire text
5. **Section splitting** separates pickup and delivery sections
6. **Pickup parsing** extracts shop names, phones, items, and notes
7. **Delivery parsing** extracts name, address, phones, and notes
8. **Complexity calculation** sums pickup complexity scores
9. **Response** returns ParsedOrder with all extracted data
10. **No database operations** performed

### Parse and Save Flow

1. **Client sends POST request** with OrderInput containing orderId, submitted_by, text, and status
2. **FastAPI validates** input using Pydantic model
3. **DeliveryParser.parse()** processes text and returns ParsedOrder
4. **Mention extraction** extracts @username references from text
5. **Fallback handling** uses submitted_by if no mentions found
6. **JSON serialization** converts pickup details, phones, and warnings to JSON
7. **Database connection** established via context manager
8. **Existing rows query** fetches current order rows for orderId
9. **Set calculation** determines to_add, to_keep, to_delete sets
10. **DELETE operations** remove rows for users no longer mentioned
11. **UPDATE operations** refresh data for users still mentioned
12. **INSERT operations** add rows for newly mentioned users
13. **Transaction commit** saves all changes atomically
14. **Response** returns success with mention changes and parsed data
15. **Error handling** rolls back transaction on exception

## Multi-Mention Order Management

The service supports multi-rider order assignment through mention tracking:

### Mention Set Operations

- **New Mentions**: @username references in current text
- **Existing Mentions**: Usernames from existing order rows
- **To Delete**: Existing mentions not in new mentions (removed riders)
- **To Keep**: Existing mentions still in new mentions (retained riders)
- **To Add**: New mentions not in existing mentions (new riders)

### Database Synchronization

- **Delete Rows**: Removes order rows for riders removed from mentions
- **Update Rows**: Refreshes order data for riders retained in mentions
- **Insert Rows**: Creates order rows for newly mentioned riders
- **Atomic Transaction**: All operations in single transaction for consistency
- **Conflict Detection**: Returns 409 for duplicate orderId/username pairs

### Use Cases

- **Rider Reassignment**: Remove rider by removing @mention
- **Multi-Rider Orders**: Assign multiple riders by mentioning multiple @usernames
- **Order Updates**: Edit text to modify mention list and order data simultaneously
- **Fallback**: Single rider assignment via submitted_by when no mentions

## Text Processing Capabilities

### Myanmar Language Support

- **Digit Conversion**: Myanmar digits (၀-၉) converted to Arabic digits (0-9)
- **Keyword Matching**: Myanmar keywords for pickup, delivery, and type classification
- **Phone Numbers**: Myanmar phone number formats with Burmese digits
- **Text Normalization**: Handles Myanmar text encoding and whitespace

### English Language Support

- **Keyword Matching**: English keywords for pickup, delivery, and type classification
- **Phone Numbers**: International and local phone number formats
- **Address Detection**: English address keywords and patterns
- **Item Units**: English unit names (pcs, box, pack, etc.)

### Mixed Language Support

- **Bilingual Keywords**: Supports both Myanmar and English keywords simultaneously
- **Flexible Parsing**: Handles mixed Myanmar-English text
- **Unicode Handling**: Proper UTF-8 support for Myanmar characters

## Security Considerations

- **Input Validation**: Pydantic model validation prevents empty text
- **SQL Injection Prevention**: Parameterized queries for all database operations
- **Transaction Safety**: Automatic rollback on errors prevents partial updates
- **Error Handling**: Generic error messages prevent information leakage
- **Environment Variables**: Sensitive credentials stored in environment variables
- **CORS Configuration**: Allows all origins for internal service communication

## Performance Optimizations

- **Regex Pre-compilation**: All regex patterns compiled at startup for performance
- **LRU Cache**: Decorator for expensive function calls (if implemented)
- **Connection Pooling**: Database connection reuse via context manager
- **Batch Operations**: Single transaction for multiple database operations
- **JSON Serialization**: Efficient JSON encoding with ensure_ascii=False
- **Dictionary Deduplication**: Uses dict.fromkeys() for unique lists while preserving order

## Error Handling

- **Validation Errors**: Returns 400 for invalid input (empty text)
- **Database Errors**: Returns 500 for general database failures
- **Integrity Errors**: Returns 409 for duplicate orderId/username pairs
- **Parse Warnings**: Collected in warnings array for client feedback
- **Transaction Rollback**: Automatic rollback on any database error
- **Logging**: Structured logging for error tracking and debugging

## External Integrations

- **Relational Database**: Relational database for order persistence
- **Internal Services**: Called by NestJS application for order parsing
- **Frontend Applications**: Parse-only endpoint for preview functionality

## Implementation Details

### Dependency Management

- **FastAPI**: Modern async web framework with automatic OpenAPI documentation
- **PyMySQL**: Relational database driver with DictCursor support
- **Pydantic**: Data validation and serialization with type hints
- **python-dotenv**: Environment variable management from .env files
- **Uvicorn**: ASGI server for production deployment

### Configuration

Environment variables control application behavior:
- **DB_HOST**: Database host address
- **DB_PORT**: Database port number
- **DB_USER**: Database username
- **DB_PASS**: Database password
- **DB_NAME**: Database name

### Database Schema

The service interacts with the orders table with the following structure:
- **id**: Auto-increment primary key
- **orderId**: Order identifier (shared across mentions)
- **username**: Username of assigned rider
- **pickup**: Concatenated shop names
- **delivery**: Delivery address or name
- **way_count**: Number of delivery ways
- **deli_fee**: Delivery fee amount
- **submitted_by**: Username of order submitter
- **complexity**: Total complexity score
- **pickup_details**: JSON array of pickup details
- **delivery_ph**: JSON array of delivery phone numbers
- **delivery_name**: Delivery recipient name
- **status**: Order status
- **warnings**: JSON array of parse warnings

### Complexity Scoring

Pickup complexity scores based on type:
- **Passenger**: 3 (complex routing)
- **Shopping**: 7 (multiple items, potential waiting)
- **Gate**: 2 (simple pickup point)
- **Food**: 3 (standard food delivery)
- **Pickup**: 2 (default pickup)

Total complexity is sum of all pickup complexities, used for rider workload calculation.

### Warning System

Parser generates warnings for:
- Missing delivery section
- Empty delivery section
- No phone numbers in delivery
- No pickup locations detected
- Missing "To" keyword for section split

Warnings help identify parsing issues and improve order quality.
