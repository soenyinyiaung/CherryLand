# Cronjob Architecture

## Overview

The Cronjob system provides automated task scheduling for the CherryLand delivery platform, handling duty management, penalty enforcement, daily reporting, and database backups. The system consists of PHP and Python scripts that run on scheduled intervals to monitor rider performance, enforce duty requirements, generate analytical reports, and maintain data integrity through regular backups.

## Technology Stack

- **PHP**: Server-side scripting for duty checking and penalty enforcement
- **Python**: Data analysis and reporting with matplotlib for chart generation
- **Bash**: Shell scripting for database backup operations
- **MySQL**: Relational database for data persistence
- **Telegram API**: External messaging service for notifications
- **Environment Variables**: Configuration management via .env files
- **Matplotlib**: Python library for data visualization
- **MySQL Connector**: Python database connector for MySQL

## Directory Structure

```
Cronjob
├── Environment configuration file
├── PHP duty checking scripts
│   ├── Morning duty checker
│   ├── Non-morning duty checker
│   └── Environment loader
├── Python reporting scripts
│   ├── Morning duty reporter
│   └── Daily summary reporter
└── Shell scripts
    └── Database backup script
```

## Core Components

### 1. Environment Loader (env_loader.php)

Utility function for loading environment variables from .env files:

**Functionality**
- Reads .env file line by line
- Skips comments (lines starting with #)
- Parses KEY=VALUE pairs
- Sets environment variables using putenv()
- Populates $_ENV superglobal array
- Handles empty lines and whitespace

**Usage**
- Required by PHP scripts for configuration
- Loads database credentials
- Loads API tokens and chat IDs
- Centralized configuration management

### 2. Morning Duty Checker (check_morning_duty.php)

PHP script that monitors morning duty compliance and enforces penalties:

**Workflow**

1. **Environment Loading**
   - Loads environment variables from .env file
   - Establishes database connection
   - Sets timezone to Asia/Yangon
   - Configures database charset to UTF8MB4

2. **Duty Retrieval**
   - Queries duty schedule table for today's assigned riders
   - Validates JSON format of username list
   - Extracts rider usernames from duty assignment

3. **Compliance Checking**
   - For each assigned rider:
     - Retrieves rider information (chat ID, Telegram username)
     - Checks realtime status for "ready" indication
     - Checks order table for completed orders
     - Determines penalty eligibility

4. **Penalty Enforcement**
   - Applies penalty if no orders completed
   - Checks for existing penalty to avoid duplicates
   - Records penalty with reason and amount
   - Accumulates penalty list for notification

5. **Duty Rotation**
   - Initiates database transaction
   - Calculates next duty date based on schedule count
   - Updates current duty entry to future date
   - Identifies next duty assignment
   - Handles schedule wrap-around (circular rotation)
   - Commits transaction or rolls back on error

6. **Notification**
   - Sends Telegram message with penalty summary
   - Formats message with HTML parsing
   - Includes penalty amounts and reasons
   - Notes night summary report will deduct penalties

**Penalty Logic**
- Penalty applied only if no orders completed
- Reason: "အဆင်သင့်မဖြစ်" (not ready) if not marked ready
- Fixed penalty amount per violation
- Duplicate prevention by checking existing penalties

### 3. Non-Morning Duty Checker (check_non_morning_duty.php)

PHP script that monitors non-morning duty riders for compliance:

**Workflow**

1. **Environment Loading**
   - Loads environment variables from .env file
   - Establishes database connection
   - Sets timezone to Asia/Yangon
   - Configures database charset to UTF8MB4

2. **Active Rider Retrieval**
   - Queries active riders with rider role
   - Excludes specific system accounts
   - Filters by status (active)
   - Retrieves rider information (chat ID, Telegram username)

3. **Compliance Checking**
   - For each active rider:
     - Checks leave days table for approved leave
     - Skips penalty if on approved leave
     - Checks realtime status for "ready" indication
     - Checks order table for completed orders
     - Determines penalty eligibility

4. **Penalty Enforcement**
   - Applies penalty if not ready and no orders
   - Checks for existing penalty to avoid duplicates
   - Records penalty with reason and amount
   - Accumulates penalty list for notification

5. **Notification**
   - Sends Telegram message with penalty summary
   - Formats message with HTML parsing
   - Includes penalty amounts and reasons
   - Notes work zone requirement for ready status

**Penalty Logic**
- Penalty applied if not ready AND no orders completed
- Reason: "အဆင်သင့်မဖြစ်" (not ready) if not marked ready
- Fixed penalty amount per violation
- Leave days exempt riders from penalties
- Duplicate prevention by checking existing penalties

### 4. Morning Duty Reporter (morning_duty.py)

Python script that reports tomorrow's morning duty assignment:

**Workflow**

1. **Environment Loading**
   - Loads environment variables from .env file
   - Configures database connection parameters
   - Retrieves Telegram bot token and chat ID

2. **Duty Retrieval**
   - Queries duty schedule table for tomorrow's assignment
   - Retrieves username list in JSON format
   - Parses JSON array of usernames
   - Cleans and formats usernames with @ prefix

3. **Notification**
   - Constructs Telegram message with duty list
   - Formats message with HTML parsing
   - Sends message via Telegram API
   - Handles API errors gracefully

**Message Format**
- Header: "🌅 Tomorrow Morning Duty"
- Body: List of @usernames on separate lines
- Parse mode: HTML

### 5. Daily Summary Reporter (summary.py)

Python script that generates daily analytical reports with visualizations:

**Workflow**

1. **Environment Loading**
   - Loads environment variables from .env file
   - Configures database connection parameters
   - Retrieves Telegram bot token and user IDs

2. **Data Retrieval**
   - Queries order table for 3-day revenue summary
   - Groups by date with revenue and way count aggregation
   - Queries top 3 riders by income for today
   - Groups by username with income and方式 aggregation

3. **Data Processing**
   - Calculates 3-day date labels
   - Maps data to date buckets
   - Extracts income and way arrays
   - Formats top rider data

4. **Chart Generation**

**Total Income Chart**
- 3-day bar chart comparison
- Target line at 900K
- Minimum limit line at 550K
- Color-coded bars (gray for past, blue for today)
- Value labels on bars
- Legend with target indicators

**Delivery Ways Chart**
- 3-day bar chart comparison
- Way target line at 250
- Way red line at 170
- Color-coded bars (gray for past, green for today)
- Value labels on bars
- Legend with target indicators

**Top Riders Chart**
- Horizontal bar chart for top 3 riders
- Income values with way counts
- Yellow color for bars
- Inverted y-axis for ranking
- Value labels with income and ways

5. **Notification**
   - Sends charts via Telegram API
   - Uploads image for first recipient
   - Reuses file_id for subsequent recipients
   - Includes captions with Markdown parsing
   - Handles multiple user IDs

6. **Cleanup**
   - Removes temporary chart files
   - Handles cleanup on errors
   - Ensures no file leakage

**Report Types**
- 3-Day Total Income Trend
- 3-Day Delivery Ways Comparison
- Top 3 Riders Today

### 6. Database Backup Script (db_backup.sh)

Bash script for automated MySQL database backups:

**Workflow**

1. **Environment Loading**
   - Loads .env file from specified path
   - Exports environment variables
   - Handles missing .env file error

2. **Backup Configuration**
   - Sets backup path for compressed backup file
   - Configures MySQL dump parameters
   - Handles password parameter conditionally

3. **Backup Execution**
   - Runs mysqldump with optimized options
   - Single transaction for consistency
   - Quick mode for large tables
   - No table locking for minimal disruption
   - Compresses output with gzip

4. **Security**
   - Sets restrictive file permissions (600)
   - Protects backup file from unauthorized access

**Backup Options**
- Single transaction for data consistency
- Quick mode for performance
- No table locking for availability
- Gzip compression for storage efficiency

## Data Flow

### Morning Duty Check Flow

1. **Cron triggers** PHP script at scheduled time
2. **Environment loader** loads configuration from .env
3. **Database connection** established with credentials
4. **Duty schedule** queried for today's assignments
5. **User information** retrieved for each assigned rider
6. **Compliance checked** via realtime and order tables
7. **Penalties applied** if criteria met (no duplicate check)
8. **Duty rotated** to next assignment in schedule
9. **Transaction committed** or rolled back on error
10. **Telegram notification** sent with penalty summary
11. **Database connection** closed

### Non-Morning Duty Check Flow

1. **Cron triggers** PHP script at scheduled time
2. **Environment loader** loads configuration from .env
3. **Database connection** established with credentials
4. **Active riders** queried from users table
5. **Leave status** checked for each rider
6. **Compliance checked** via realtime and order tables
7. **Penalties applied** if criteria met (no duplicate check)
8. **Telegram notification** sent with penalty summary
9. **Database connection** closed

### Daily Summary Report Flow

1. **Cron triggers** Python script at scheduled time
2. **Environment loader** loads configuration from .env
3. **Database connection** established with credentials
4. **Revenue data** queried for 3-day period
5. **Top riders** queried for today's performance
6. **Charts generated** using matplotlib
7. **Images saved** as temporary files
8. **Telegram API** called with image uploads
9. **File IDs reused** for multiple recipients
10. **Temporary files** cleaned up
11. **Database connection** closed

### Database Backup Flow

1. **Cron triggers** bash script at scheduled time
2. **Environment loader** loads .env file
3. **Variables exported** for shell environment
4. **mysqldump executed** with optimized parameters
5. **Output compressed** with gzip
6. **File saved** to backup location
7. **Permissions set** to restrictive mode
8. **Process completes** with backup file

## Database Interactions

### Tables Accessed

**Duty Schedule Table**
- Stores daily duty assignments
- Contains date and username list (JSON)
- Used for duty rotation and reporting

**Users Table**
- Stores rider information
- Contains username, Telegram username, chat ID
- Used for rider identification and notification

**Realtime Table**
- Stores rider status updates
- Contains username, status, timestamp
- Used for readiness checking

**Orders Table**
- Stores order records
- Contains username, delivery fee, way count, timestamp
- Used for order completion checking and reporting

**Penalties Table**
- Stores penalty records
- Contains username, date, reason, amount
- Used for penalty tracking and duplicate prevention

**Leave Days Table**
- Stores approved leave records
- Contains username, date
- Used for leave exemption checking

### Query Patterns

**Duty Retrieval**
- Single record query by date
- JSON parsing for username arrays
- Transaction-based updates

**Compliance Checking**
- Existence checks for realtime status
- Existence checks for order records
- Date-based filtering for today's records

**Penalty Management**
- Duplicate check before insertion
- Single record insertion
- Reason-based grouping

**Reporting Queries**
- Aggregation by date for revenue
- Aggregation by username for performance
- Ordering by income/ways for rankings
- Date range filtering for 3-day periods

## Security Considerations

### Authentication

- **Environment Variables**: Sensitive credentials stored in .env files
- **Database Credentials**: Separate credentials for cronjob access
- **API Tokens**: Telegram bot tokens for notification access
- **File Permissions**: Restrictive permissions on backup files (600)

### Data Protection

- **SQL Injection Prevention**: Prepared statements for all database queries
- **Transaction Safety**: Database transactions for atomic operations
- **Error Handling**: Graceful error handling with logging
- **Backup Security**: Encrypted or restricted access to backup files

### Access Control

- **Database Access**: Limited to required tables and operations
- **API Access**: Telegram API with bot tokens
- **File System**: Restricted backup file locations
- **Network**: Outbound HTTPS requests only

## Performance Optimizations

### Database Operations

- **Prepared Statements**: Reusable prepared statements for queries
- **Index Usage**: Queries utilize database indexes
- **Transaction Batching**: Multiple operations in single transaction
- **Connection Management**: Proper connection closing

### Script Optimization

- **Duplicate Prevention**: Check before insert to avoid redundant operations
- **Conditional Execution**: Skip processing when not needed (leave days)
- **Efficient Queries**: Optimized SQL with proper filtering
- **Memory Management**: Cleanup of temporary files

### Backup Optimization

- **Single Transaction**: Consistent backup without locking
- **Quick Mode**: Fast dump for large tables
- **No Locking**: Minimal disruption to operations
- **Compression**: Reduced storage requirements

## Error Handling

### Database Errors

- **Connection Errors**: Logged and script exits gracefully
- **Query Errors**: Logged with error details
- **Transaction Errors**: Rollback on exception
- **Connection Cleanup**: Ensure connection closure

### API Errors

- **Telegram API**: Logged on failure, script continues
- **Timeout Handling**: 10-second timeout for API calls
- **Response Validation**: Check API response success

### File System Errors

- **Missing .env**: Error message and exit
- **Backup Path**: Error handling for file operations
- **Temporary Files**: Cleanup on error or success

## Scheduling

### Recommended Cron Schedule

**Morning Duty Checker**
- Schedule: Daily at specified morning time
- Purpose: Check morning duty compliance
- Frequency: Once per day

**Non-Morning Duty Checker**
- Schedule: Daily at specified time
- Purpose: Check general rider compliance
- Frequency: Once per day

**Morning Duty Reporter**
- Schedule: Daily evening before duty
- Purpose: Announce tomorrow's duty
- Frequency: Once per day

**Daily Summary Reporter**
- Schedule: Daily evening after operations
- Purpose: Generate performance reports
- Frequency: Once per day

**Database Backup**
- Schedule: Daily at low-traffic time
- Purpose: Create database backup
- Frequency: Once per day

## External Integrations

### Telegram API

- **Bot Token**: Authentication for bot API access
- **Chat ID**: Target chat for notifications
- **Message Types**: Text messages and photo uploads
- **Parse Modes**: HTML and Markdown support
- **File Reuse**: File ID optimization for multiple recipients

### MySQL Database

- **Connection**: MySQL connector for Python
- **Connection**: MySQLi extension for PHP
- **Charset**: UTF8MB4 for Myanmar language support
- **Timezone**: Asia/Yangon for local time operations

## Implementation Details

### Configuration Management

**Environment Variables**
- Database host, user, password, name
- Telegram bot token and chat IDs
- Summary user IDs (comma-separated)
- Backup paths and file locations

### Dependency Management

**PHP Dependencies**
- MySQLi extension for database connectivity
- Standard PHP libraries for HTTP requests

**Python Dependencies**
- mysql-connector-python for database
- matplotlib for chart generation
- requests for HTTP requests
- python-dotenv for environment loading

**Bash Dependencies**
- mysqldump for database export
- gzip for compression

### Logging

- **Error Logging**: PHP error_log function
- **Print Logging**: Python print statements
- **Exit Codes**: Non-zero exit on errors
- **Error Messages**: Descriptive error messages

This cronjob system provides automated monitoring, enforcement, and reporting for the delivery platform, ensuring duty compliance, tracking performance, and maintaining data integrity through regular backups.
