# Image Service Architecture

## Overview

The Image Service is a Go-based HTTP API server for handling image uploads, storage, and retrieval with automatic WebP optimization. It serves as a centralized image management system for the CherryLand platform, providing image storage for multiple services including profiles, vouchers, orders, settlements, payments, and marketplace content. The service converts all uploaded images to WebP format for optimal file size and performance.

## Technology Stack

- **Language**: Go (Golang)
- **Web Framework**: Gin (HTTP router and middleware)
- **Image Processing**: chai2010/webp library for WebP conversion
- **Database**: MySQL (for session validation)
- **Storage**: Local filesystem with volume mounting support
- **Image Formats**: JPEG, PNG, GIF, WebP (all converted to WebP)

## Directory Structure

```
Image Server/
├── Main application entry point
├── Converter module
│   └── Image conversion logic
├── Database module
│   └── Database connection and session validation
├── Handlers module
│   └── HTTP request handlers
├── Middleware module
│   └── Authentication and CORS middleware
└── Storage directory
    ├── Profile images
    ├── Voucher attachments
    ├── Order attachments
    ├── Settlement brochures
    ├── Payment receipts
    └── Marketplace images
        ├── Business logos
        ├── Business covers
        ├── Category images
        ├── Product images
        └── Overall post images
```

## Core Components

### 1. Application Entry Point (main.go)

The main function initializes the application and configures HTTP routes:

- **Database Initialization**: Establishes MySQL connection for session validation
- **Router Setup**: Configures Gin router with middleware
- **Middleware Registration**: CORS and logging middleware
- **Route Configuration**: Service-specific route groups for different image types
- **Server Startup**: Starts HTTP server on port 80

#### Route Structure

The service organizes routes by service type with consistent patterns:

- **Profile Service**: Avatar images for user profiles
- **Voucher Service**: Attachment images for vouchers
- **Order Service**: Attachment images for orders
- **Settlement Service**: Brochure images for settlements
- **Payment Service**: Receipt images for payments
- **Marketplace Service**: Multiple image types for marketplace content
  - Business logos
  - Business cover images
  - Category images
  - Product images
  - Overall post images

Each service has three operations:
- GET endpoint for retrieving images (public)
- POST endpoint for uploading images (protected with authentication)
- DELETE endpoint for deleting images (protected with authentication)

### 2. Handlers Module (handlers.go)

The handlers module contains HTTP request handlers for image operations:

#### Image Structure

The Image struct represents image metadata:
- **ID**: Unique identifier (UUID)
- **Filename**: Storage filename (always .webp)
- **URL**: Public URL for image access
- **Size**: File size in bytes
- **MimeType**: MIME type (always image/webp)
- **Format**: Image format (always webp)
- **Service**: Service identifier (profile, vouchers, orders, etc.)

#### Service Path Mapping

Service-specific storage paths organize images by type:
- Profile images stored in profiles directory
- Voucher attachments stored in vouchers directory
- Order attachments stored in orders directory
- Settlement brochures stored in settlements directory
- Payment receipts stored in payments directory
- Marketplace images organized in marketplace subdirectories

#### Handler Functions

**UploadImage**
- Receives multipart form data with image file
- Validates file type (JPEG, PNG, GIF, WebP)
- Reads and validates image data
- Converts image to WebP format with quality setting (75)
- Generates unique ID using UUID
- Saves WebP file to service-specific storage path
- Returns image metadata with public URL
- Supports both simple and nested marketplace route structures

**GetImage**
- Retrieves image by ID from service-specific storage path
- Serves WebP file with appropriate content type
- Sets caching headers for 1-year cache duration
- Returns 404 if image not found
- Optimized for CDN and browser caching

**DeleteImage**
- Deletes image by ID from service-specific storage path
- Returns success response with image ID and service
- Returns 404 if image not found
- Used for image cleanup and removal

**ListImages**
- Lists all images for a specific service
- Reads storage directory for WebP files
- Returns array of image metadata with URLs
- Includes total count of images
- Useful for inventory and management operations

### 3. Converter Module (converter.go)

The converter module handles image format conversion and validation:

#### Conversion Functions

**ConvertToWebP**
- Decodes image from original format (JPEG, PNG, GIF, WebP)
- Encodes to WebP format with specified quality (default 75)
- Uses chai2010/webp library for true WebP encoding
- Returns compressed WebP byte array
- Quality range validation (1-100)
- Recommended quality: 75-80 for balance between size and quality

**GetImageInfo**
- Extracts image metadata (width, height, format)
- Uses standard image decoding library
- Returns dimensions and format information
- Used for validation and processing decisions

**ValidateImage**
- Validates image data integrity
- Attempts to decode image to verify format
- Returns boolean indicating validity
- Prevents corrupted or invalid file uploads

**GetFileExtension**
- Always returns .webp extension
- Maintains consistent file naming convention

**GetMimeTypeFromFilename**
- Always returns image/webp MIME type
- Ensures consistent content-type headers

### 4. Database Module (database.go)

The database module manages MySQL connection for session validation:

#### Connection Configuration

- **Host**: Configured via DB_HOST environment variable (default: mariadb)
- **User**: Configured via DB_USER environment variable (default: root)
- **Password**: Configured via DB_PASSWORD environment variable
- **Database Name**: Configured via DB_NAME environment variable
- **Connection Pool**: Max 25 open connections, 5 idle connections
- **Parse Time**: Enabled for proper time handling

#### Functions

**InitDB**
- Initializes database connection with environment variables
- Validates connection with ping operation
- Configures connection pool settings
- Returns error if connection fails
- Used by middleware for session validation

**CloseDB**
- Closes database connection gracefully
- Called on application shutdown
- Ensures proper resource cleanup

### 5. Middleware Module (middleware.go)

The middleware module provides HTTP middleware for authentication and CORS:

#### CORS Middleware

- **Origin**: Allows all origins (*) for cross-origin requests
- **Credentials**: Supports credentials for authenticated requests
- **Headers**: Allows custom headers including X-Internal-Auth, X-Session-Token, X-User-Id
- **Methods**: Supports POST, OPTIONS, GET, PUT, DELETE
- **Preflight**: Handles OPTIONS requests with 204 status

#### Logger Middleware

- **Request Timing**: Measures request duration
- **Logging**: Logs method, path, status, and duration
- **Performance Monitoring**: Enables performance analysis
- **Debugging**: Aids in troubleshooting request issues

#### Internal Auth Middleware

The authentication middleware supports multiple authentication methods:

**Internal Service Authentication**
- Validates X-Internal-Auth header against INTERNAL_AUTH_KEY environment variable
- Used for service-to-service communication
- Bypasses session validation for internal services
- Primary authentication method for protected endpoints

**Admin Session Validation**
- Validates X-Session-Token header against admin session table
- Joins with users table to retrieve user ID
- Checks token expiration
- Sets user_id and session_type context variables
- Used for admin user authentication

**Rider Session Validation**
- Validates X-Session-Token header against rider session table
- Joins with users table to retrieve user ID
- Checks token expiration
- Sets rider_id and session_type context variables
- Used for rider authentication

**Development Mode**
- If INTERNAL_AUTH_KEY is not set, allows all requests
- Intended for development and testing
- Should not be used in production

**Security Features**
- Logs invalid authentication attempts with client IP
- Returns 403 Forbidden for invalid authentication
- Prevents unauthorized access to protected endpoints
- Supports multiple authentication methods for flexibility

## Data Flow

### Image Upload Flow

1. **Client sends POST request** with multipart form data containing image file
2. **CORS middleware** validates cross-origin request headers
3. **Internal Auth middleware** validates authentication (internal key or session token)
4. **UploadImage handler** receives file from form data
5. **File type validation** checks extension against allowed types (JPEG, PNG, GIF, WebP)
6. **File reading** loads image data into memory
7. **Image validation** verifies image integrity using converter.ValidateImage
8. **WebP conversion** converts image to WebP format with quality setting (75)
9. **UUID generation** creates unique identifier for image
10. **File saving** writes WebP file to service-specific storage path
11. **URL generation** creates public URL based on service type and image ID
12. **Response** returns image metadata with ID, filename, URL, size, and MIME type

### Image Retrieval Flow

1. **Client sends GET request** with image ID in URL path
2. **CORS middleware** validates cross-origin request headers
3. **GetImage handler** extracts image ID from URL parameter
4. **Path resolution** determines service-specific storage path
5. **File lookup** searches for WebP file with matching ID
6. **File reading** loads image data from filesystem
7. **Cache headers** sets Cache-Control for 1-year caching
8. **Content type** sets image/webp MIME type
9. **Response** returns image data with appropriate headers
10. **Browser caching** stores image locally for 1 year

### Image Deletion Flow

1. **Client sends DELETE request** with image ID in URL path
2. **CORS middleware** validates cross-origin request headers
3. **Internal Auth middleware** validates authentication
4. **DeleteImage handler** extracts image ID from URL parameter
5. **Path resolution** determines service-specific storage path
6. **File deletion** removes WebP file from filesystem
7. **Response** returns success with image ID and service
8. **Error handling** returns 404 if file not found

## Security Considerations

- **Authentication**: Protected endpoints require valid authentication (internal key or session token)
- **Session Validation**: Database-backed session validation with expiration checks
- **CORS Configuration**: Configurable CORS headers for cross-origin requests
- **File Type Validation**: Strict validation of uploaded file types
- **Image Validation**: Integrity check to prevent corrupted files
- **Internal Auth Key**: Shared secret for service-to-service communication
- **Development Mode**: Allows bypassing authentication when INTERNAL_AUTH_KEY not set
- **Request Logging**: Logs all requests including failed authentication attempts
- **Path Traversal Prevention**: Uses filepath.Join for safe path construction
- **UUID Generation**: Uses UUID for unique, unpredictable image IDs

## Performance Optimizations

- **WebP Conversion**: Reduces file size by 25-35% compared to JPEG/PNG
- **HTTP Caching**: 1-year cache headers for images (Cache-Control: public, max-age=31536000, immutable)
- **Connection Pooling**: Database connection pool with configurable limits
- **Local Storage**: Filesystem-based storage for fast access
- **Format Standardization**: All images stored as WebP for consistency
- **Quality Optimization**: 75 quality setting balances size and visual quality
- **Browser Caching**: Immutable cache directive prevents revalidation
- **CDN Compatibility**: Cache headers optimized for CDN integration

## Storage Organization

The service organizes images by service type in a hierarchical directory structure:

### Service-Specific Directories

- **Profiles**: User avatar images
- **Vouchers**: Voucher attachment images
- **Orders**: Order attachment images
- **Settlements**: Settlement brochure images
- **Payments**: Payment receipt images
- **Marketplace/Businesses/Logo**: Business logo images
- **Marketplace/Businesses/Covers**: Business cover images
- **Marketplace/Categories**: Category images
- **Marketplace/Products**: Product images
- **Marketplace/Overall**: Overall post images

### File Naming Convention

- All files stored with UUID-based names
- File extension always .webp
- No original filename stored (only UUID.webp)
- Prevents filename conflicts and ensures uniqueness

### Volume Mounting

Storage directory designed for Docker volume mounting:
- Persistent storage across container restarts
- Easy backup and migration
- Scalable storage capacity
- Separation of application code and data

## External Integrations

- **MySQL Database**: Session validation for admin and rider authentication
- **Internal Services**: Service-to-service communication via X-Internal-Auth header
- **CDN Integration**: Cache headers optimized for CDN deployment
- **Container Orchestration**: Designed for Docker and Kubernetes deployment

## Implementation Details

### Dependency Management

- **Go Modules**: Uses go.mod and go.sum for dependency management
- **Gin Framework**: HTTP routing and middleware
- **WebP Library**: chai2010/webp for image conversion
- **MySQL Driver**: go-sql-driver/mysql for database connectivity
- **UUID Generator**: google/uuid for unique identifiers

### Error Handling

- **File Validation**: Returns 400 for invalid file types
- **Image Validation**: Returns 400 for corrupted images
- **Storage Errors**: Returns 500 for filesystem failures
- **Not Found**: Returns 404 for missing images
- **Authentication**: Returns 403 for invalid authentication
- **Database Errors**: Logs and handles connection failures

### Configuration

Environment variables control application behavior:
- **DB_HOST**: Database host (default: mariadb)
- **DB_USER**: Database user (default: root)
- **DB_PASSWORD**: Database password
- **DB_NAME**: Database name
- **INTERNAL_AUTH_KEY**: Internal authentication key for service-to-service communication

### Service Resource Mapping

The service maps internal service names to public resource names:
- Profile → avatar
- Voucher → attachment
- Order → attachment
- Settlement → brochure
- Payment → receipt
- Marketplace businesses logo → logo
- Marketplace businesses cover → cover
- Marketplace categories → categorie
- Marketplace products → product
- Marketplace overall → post

This mapping allows consistent public URLs while maintaining internal service organization.
