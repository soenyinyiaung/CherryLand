# NestJS Application Architecture

## Overview

The NestJS application serves as the central real-time communication and order management backend for the CherryLand delivery platform. It handles chat messaging, rider tracking, order processing, and push notifications through WebSocket connections and REST APIs.

## Technology Stack

- **Framework**: NestJS (Node.js framework)
- **WebSocket**: Socket.IO for real-time communication
- **Database**: Document database (for chat/orders), Relational database (for operational data)
- **Push Notifications**: Firebase Cloud Messaging (FCM)
- **HTTP Client**: Axios for external API calls
- **Language**: TypeScript

## Directory Structure

```
src/
├── Main application entry point
├── Root application module
├── Controllers
│   ├── Admin FCM controller
│   ├── Admin controller
│   ├── Order controller
│   ├── Order chat controller
│   ├── Pickup way controller
│   └── Rider status controller
├── Database layer
│   ├── Database module
│   └── Database service
├── Firebase integration
│   ├── Firebase module
│   └── Firebase service
├── Orders module
│   ├── Counter schema
│   ├── Order schema
│   ├── Orders module
│   └── Orders service
├── Services
│   ├── Date service
│   ├── Realtime service
│   ├── Rider notification service
│   ├── Rider service
│   └── User service
└── Websockets
    ├── Chat gateway
    ├── Riders gateway
    └── Websockets module
```

## Core Components

### 1. Application Bootstrap

The application entry point handles:

- **CORS Configuration**: Allows requests from specific domains (cherrylandtaunggyi.com, soenyinyiaung.site, localhost)
- **Port Configuration**: Uses environment PORT or defaults to 80
- **Network Binding**: Listens on 0.0.0.0 for containerized deployment
- **HTTP Methods**: Supports GET, POST, OPTIONS, PUT, PATCH, DELETE with credentials

### 2. Root Application Module

The AppModule orchestrates all application modules:

- **Database Module**: Relational database connection pool for operational data
- **Firebase Module**: FCM integration for push notifications
- **WebSockets Module**: Socket.IO gateways for real-time communication
- **HTTP Module**: Axios for external HTTP requests
- **Document Database Module**: NoSQL document database connection for chat/order documents
- **Orders Module**: Order management and document database schemas
- **Controllers**: All HTTP REST endpoints

### 3. Chat Gateway

The ChatGateway handles real-time chat communication between admin users through Socket.IO WebSocket connections. This gateway is responsible for message persistence, real-time broadcasting, user presence tracking, and integration with external services for order processing and notifications. It serves as the central hub for all admin-to-admin and admin-to-rider communication within the platform.

#### Connection Management
- **Token Validation**: Validates admin session tokens from handshake auth against admin_sessions table joined with users table, checking token validity and expiration
- **Online User Tracking**: Maintains in-memory Map tracking connected users by user ID with Set of socket IDs per user for multi-device support
- **Personal Rooms**: Each user automatically joins their personal room (`user-{userId}`) for targeted messaging across all their devices
- **Chat Rooms**: Users explicitly join specific chat rooms (`chat-{chatType}`) for team-based communication channels
- **Disconnect Handling**: Removes socket from user's socket set, emits user-offline event globally when no sockets remain for that user, cleans up online tracking
- **Auto-Rejoin Support**: Automatically rejoins chat rooms on message send to prevent dropped messages during reconnection

#### Message Events

**send-message**
- Validates sender ID against authenticated user from client.data.userId to prevent message spoofing
- Auto-joins chat room to prevent dropped messages during temporary disconnections
- Saves message to document database via OrdersService which handles transformation, mention parsing, and external service integration
- Sends acknowledgment to sender with server-generated ID for optimistic UI updates
- Broadcasts message to all room participants with server ID and attachments field
- Sends FCM notifications to mentioned users and reply targets for push notification delivery
- Supports both legacy images array format and new imageUrls array format for backward compatibility
- Handles messageType 'order' specially by triggering FastAPI parsing and rider notifications

**edit-message**
- Parses mentions from new text using regex pattern @(\w+) to extract all @username references
- Updates document database document with edit history tracking who edited and when
- Sends to FastAPI for order re-parsing if messageType is 'order' to update structured delivery data
- Broadcasts edit to chat room and sender's personal room for multi-device synchronization
- Maintains edit history array with previousText, newText, editedBy, editedAt for audit trail
- Updates attachments array to handle image additions/removals during edit
- Extracts imageUrls from attachments for broadcast compatibility

**delete-message**
- Soft deletes from document database by setting isDeleted flag instead of actual removal for audit trail
- Records deletion timestamp and deleter user ID for accountability
- Broadcasts deletion to chat room and personal room for immediate UI update across devices
- Updates realtime tracking table to remove order ID from all mentioned riders
- Notifies riders via socket and FCM that order has been removed
- Deletes from relational database orders table to remove structured delivery data
- Ensures data consistency across document database, relational database, and realtime systems

**cancel-order**
- Toggles isCancelled flag in document database to mark order as cancelled or restored
- Broadcasts cancellation to chat room and personal room for admin visibility
- For cancelled orders: removes from realtime tracking table, notifies riders via socket/FCM, deletes from relational database orders table
- For restored orders: adds to realtime tracking table, notifies riders via socket/FCM, sends to FastAPI to recreate structured data
- Handles rider assignment changes during cancel/restore operations
- Maintains order history while preventing rider actions on cancelled orders

**add-reaction**
- Toggles emoji reactions on messages
- Server-side reaction calculation to prevent conflicts
- Broadcasts reaction updates to chat room and personal room

**mark-as-read**
- Client manages seenBy array
- Server broadcasts read receipts to chat room
- No database storage for read receipts

**typing-start/typing-stop**
- Tracks typing users per chat room
- Broadcasts typing indicators to room participants

#### FCM Token Management
- **Token Registration**: Registers admin FCM tokens via socket
- **Database Loading**: Loads tokens from database on module init
- **Duplicate Cleanup**: Removes duplicate token entries on startup
- **Token Caching**: In-memory cache for quick notification sending

### 4. Riders Gateway

The RidersGateway manages rider real-time tracking, location updates, and notifications through Socket.IO WebSocket connections. This gateway is the primary interface for rider device communication, handling GPS tracking, online status management, and order-related notifications. It maintains rider presence state and coordinates with other services for order assignment and delivery tracking.

#### Connection Management
- **Room Joining**: Riders join their personal room using rider ID for targeted messaging and location broadcasts
- **Online Status**: Updates rider location tracking table online status field to 1 when connected, 0 when disconnected
- **Name Caching**: Maintains in-memory cache of rider names and username-to-ID mappings to reduce database queries
- **Disconnect Handling**: Updates offline status in database, emits rider-offline event globally, broadcasts updated online rider count
- **Multi-Device Support**: Allows same rider to connect from multiple devices simultaneously
- **Auto-Name Resolution**: Automatically queries users table for rider username if not in cache during connection

#### Rider Events

**join-rider-room**
- Trims and validates rider ID string to prevent injection attacks
- Adds rider ID to online riders Set for presence tracking
- Joins rider's personal room using rider ID as room name for targeted messaging
- Updates online status in rider location tracking table to 1 for latest record
- Loads rider name from users table if not already in cache
- Broadcasts online rider count to all connected clients for admin dashboard updates
- Falls back to "Rider-{id}" if username not found in database

**sendLocation**
- Receives GPS coordinates (latitude, longitude) from rider device along with optional device status
- Resolves username from in-memory cache or queries users table if not cached
- Broadcasts location updates globally via updateLocations event for all admin clients to display on map
- Saves location to rider location tracking table asynchronously via UserService to avoid blocking socket response
- Includes battery percentage, charging status, and online status in database record
- Used for real-time rider tracking on admin dashboard map view

**rider-done-order**
- Queries rider status from database joining users table, realtime tracking table, and latest rider location tracking table record
- Parses activity status JSON string into array format (activity, thermal, homeway, orders)
- Formats rider data with identity, activity status, and device state for admin display
- Emits rider-refresh-row event to update admin dashboard table without full page reload
- Provides comprehensive rider snapshot including GPS, battery, charging status, and current orders
- Called when rider completes an order to refresh admin view

**register-fcm-token**
- Registers rider FCM tokens
- Saves to database
- Maintains in-memory cache for quick access

**force-reload**
- Emits force-reload event to specific user or all clients
- Used for version updates

#### FCM Token Management
- **Token Loading**: Loads rider/developer tokens on module init
- **Token Caching**: In-memory cache by rider ID
- **Token Retrieval**: Methods to get single or multiple tokens per rider

### 5. Orders Service

The OrdersService manages order documents in a document database with comprehensive CRUD operations, mention parsing, and integration with external services. This service is the core data layer for chat and order management, handling message transformation, persistence, search, and coordination with FastAPI for order parsing and rider notification services for delivery assignments.

#### Message Transformation
- **Image Handling**: Converts legacy images array format with full image objects to standardized attachments format with id, url, filename, size fields
- **URL Handling**: Converts new imageUrls array format with relative URLs to attachments format by extracting ID from URL path
- **Default Values**: Sets messageType to 'message' if not provided, initializes empty arrays for reactions and seenBy, sets isCancelled to false
- **Image URL Transformation**: Prepends image server URL to relative URLs for full path resolution in client applications
- **Attachment Normalization**: Ensures all image references follow consistent attachment structure regardless of input format

#### Order Operations

**create**
- Transforms message DTO using transformMessageDto to normalize input format
- Parses mentions from text using regex @(\w+) pattern and resolves to user IDs via UserService
- Saves to document database with mentions array for rider assignment tracking
- Sends to FastAPI if messageType is 'order' to parse structured delivery data (fee, ways, locations)
- Notifies all mentioned riders via RiderNotificationService with socket events and FCM push notifications
- Returns saved message with server-generated ID for client reference

**findAll**
- Filters by chatType for room-specific message retrieval
- Supports updatedAt timestamp filter for incremental sync (only returns messages updated after given time)
- Implements pagination with limit and offset parameters for efficient large dataset handling
- Excludes soft-delete fields (isDeleted, deletedAt, deletedBy) from response for clean data
- Converts UTC dates to Yangon timezone for local display consistency
- Transforms image URLs to full paths with image server URL prefix
- Sorts by updatedAt descending to show newest messages first

**findByChatType**
- Fetches all messages for a specific chat room by chatType field
- Sorted by timestamp descending for chronological order (newest first)
- Excludes soft-delete fields from response
- Applies date conversion and URL transformation for client compatibility
- Used for initial chat room load and full refresh scenarios

**findOne**
- Fetches single message by numeric ID field
- Excludes soft-delete fields from response
- Applies date conversion and URL transformation
- Returns null if message not found for graceful error handling
- Used for message detail views and edit operations

**update**
- Handles rider assignment changes by comparing old and new mentions arrays
- Removes riders no longer mentioned: calls RealtimeService.rtAction to remove order ID from rider's realtime tracking table, sends remove notification via RiderNotificationService
- Adds newly mentioned riders: calls RealtimeService.rtAction to add order ID to rider's realtime tracking table, sends assign notification via RiderNotificationService
- Notifies existing riders of text edits with edit action notification
- Sends to FastAPI if text changed and messageType is 'order' to update structured delivery data
- Maintains data consistency across document database, realtime tracking table, and rider notifications
- Returns updated message with all transformations applied

**updateByExternalId**
- Updates order by external system message ID field for external system integration
- Sends to FastAPI if text changed to keep structured data synchronized
- Used when external systems update order content
- Maintains compatibility with external message platforms

**addReaction/removeReaction**
- Toggles emoji reactions on messages by adding or removing reaction objects
- Each reaction contains userId, emoji, and createdAt timestamp
- Prevents duplicate reactions by checking existing reactions before adding
- Returns updated message with reactions array
- Used for message engagement tracking and quick feedback

**softDelete**
- Marks message as deleted by setting isDeleted flag to true
- Records deletion timestamp and deleter user ID for audit trail
- Does not remove from database, allowing recovery if needed
- Used for user-initiated deletion while maintaining history

**getMessageContext**
- Fetches target message and surrounding messages for context view
- Returns older messages (with ID less than target) and newer messages (with ID greater than target) separately
- Indicates if more messages exist in each direction with hasMoreOlder and hasMoreNewer flags
- Supports pagination with limit parameter for context window size
- Used for "jump to message" functionality and infinite scroll context

**searchMessages**
- Full-text search in message content using regex with case-insensitive matching
- Filter by sender ID to show messages from specific user
- Filter by mention to show messages where specific user was mentioned
- Filter by time range using startTime and endTime timestamps
- Supports pagination with limit and offset for result sets
- Returns messages sorted by updatedAt descending (most recent first)
- Used for message search and filtering in admin interface

#### Mention Parsing
- Regex pattern: @(\w+)
- Resolves usernames to user IDs via UserService
- Returns array of {userId, userName} objects

### 6. Firebase Service

The FirebaseService handles push notifications via a cloud messaging service, providing multi-platform notification delivery for Android, iOS (APNs), and Web Push. This service manages notification formatting, platform-specific configurations, and error handling for invalid tokens. It supports both data-only notifications for wake-up functionality and full notifications with title/body for user-facing alerts.

#### Notification Types

**Rider Notifications**
- **new-order**: New order assignment (action: create/edit/remove) with localized Myanmar text for different actions
- **order-photo-update**: Photo added to order with count display
- **order-remove**: Order removed with cancellation notification
- **new-pickup-way**: New pickup way assigned for delivery route
- **pickup-way-updated**: Pickup way modified with edit notification
- **pickup-way-deleted**: Pickup way removed with deletion notification

**Admin Notifications**
- **mention**: User mentioned in message with localized Myanmar text for mention alerts
- **reply**: User's message replied to with reply indicator icon
- Custom admin notifications for system alerts and updates

#### Notification Features
- **Multi-platform Support**: Android with channel_id and click_action, iOS (APNs) with sound and badge, Web Push with urgency header and require interaction
- **High Priority**: Urgency set to high for immediate delivery across all platforms
- **Custom Data**: Additional data payload for app handling (orderId, chatId, action, etc.)
- **Sound**: Default notification sound for user attention
- **Click Actions**: Deep links to specific screens (rider app, admin chat, admin dashboard)
- **Icons/Badges**: App icons and badge increments for unread count
- **Wake-up Only**: Silent data-only notifications to wake up app without showing UI
- **Invalid Token Handling**: Automatically deletes invalid tokens from database via UserService when FCM returns registration errors

#### Message Structure
- **Android**: Channel ID, sound, click action, priority
- **APNs**: Sound, badge, content-available flag
- **Web Push**: Urgency header, require interaction, icons
- **Data Payload**: Custom key-value pairs for app logic

### 7. Rider Notification Service

The RiderNotificationService coordinates rider notifications across multiple channels including WebSocket socket events and Firebase Cloud Messaging push notifications. This service acts as the notification orchestration layer, ensuring riders receive order assignments, updates, and removals through both real-time socket connections (when online) and push notifications (when offline or in background). It maintains data consistency between socket broadcasts and FCM deliveries.

#### Notification Actions

**add_photo**
- Emits order-photo-update event to rider's personal socket room for real-time update
- Sends FCM notification with photo count and localized Myanmar text
- Used when admin adds photos to existing orders for rider reference
- Includes orderId, photoCount, and message in notification data
- Rider app displays photo count and prompts to view in app

**all_completed**
- Emits admin-row-update event globally to all connected admin clients
- Broadcasts rider completion status with username and timestamp
- Used when rider completes all assigned orders to refresh admin dashboard
- Includes rider ID, username, and completion timestamp
- Updates admin table row without full page reload

**remove**
- Emits new-order event with remove action to rider's socket room
- Emits admin-row-update event globally to all admin clients
- Sends FCM notification with order removal message
- Used when order is cancelled or deleted to remove from rider's active list
- Includes orderId in notification data for client-side removal
- Updates both rider app and admin dashboard simultaneously

**create/edit/assign**
- Emits new-order event to rider's personal socket room with full order data
- Emits admin-row-update event globally to all admin clients for dashboard sync
- Sends FCM notification with action type (create/edit) and order details
- Includes comprehensive order data: orderId, rawText, chatId, submitted_by, shop_name
- Used for initial order assignment and text edits to existing orders
- Rider app displays order details with action-specific UI (new vs edit)
- Admin dashboard updates rider row with latest order status

#### Notification Flow
1. Resolves rider ID to socket room ID via RiderService.resolveRiderId for targeted delivery
2. Emits socket event to rider's personal room for real-time delivery when rider is online
3. Emits admin update event globally to all connected admin clients for dashboard synchronization
4. Sends FCM notification if token available via RidersGateway.getRiderFcmToken for offline delivery
5. Returns success/failure status for error handling and logging
6. Handles missing rider identifiers gracefully with error messages

### 8. Realtime Service

The Realtime Service manages the realtime tracking table for rider order assignments, maintaining the mapping between riders and their currently assigned orders. This service handles the addition and removal of order IDs from rider assignments using a comma-separated list format, ensuring atomic updates and data consistency. It supports both single rider updates and batch operations for multiple riders simultaneously.

#### Operations

**rtAction**
- **add**: Adds order ID to rider's order_id comma-separated list if not already present
- **remove**: Removes order ID from rider's order_id comma-separated list
- Escapes username using DatabaseService.escape for SQL injection prevention
- Handles empty order_id lists gracefully (empty string after removal)
- Updates realtime tracking table atomically with single UPDATE statement
- Fetches current order_ids, modifies in memory, writes back to database
- Used by OrdersService.update for rider assignment changes

**batchRtAction**
- Processes multiple rider updates in sequence by iterating through array of {username, orderId, mode} objects
- Used for bulk order assignment changes when multiple riders are affected simultaneously
- Calls rtAction for each rider in the array
- Ensures all rider assignments are updated consistently
- Used during order deletion and mass rider reassignment operations

#### Realtime Tracking Table Structure
- **username**: Rider username (primary key)
- **order_id**: Comma-separated list of assigned order IDs
- **status**: JSON string with activity status
- **timestamp**: Last update timestamp

### 9. User Service

The UserService provides comprehensive user-related database operations including user identity resolution, location tracking, and push notification token management. This service acts as the primary interface for user data access across the application, ensuring consistent data handling and error recovery.

#### User Operations

**getUserIdByUsername**
- Queries users table by username using parameterized query for security
- Returns user ID as integer if found, null if not found
- Used for resolving @username mentions in chat messages to actual user IDs
- Implements error handling with null fallback for missing users
- Critical for mention parsing and rider assignment logic

**getUsernameById**
- Queries users table by numeric ID using parameterized query
- Returns username string if found, falls back to ID string if user not found
- Used for displaying user-friendly names instead of numeric IDs
- Provides graceful degradation when user data is unavailable
- Essential for rider notifications and admin display

**getUserRole**
- Queries users table for role field (admin, rider, office, ceo, developer)
- Returns 'rider' as default role if query fails or role not found
- Determines FCM token storage location and notification routing
- Used for permission-based access control in various services
- Supports role-based token management in FCM operations

**saveRiderLocation**
- Inserts new record into rider location tracking table with full GPS and device state
- Includes latitude (decimal degrees), longitude (decimal degrees), battery percentage (string), charging status (boolean), online status (integer)
- Creates new record each time instead of updating existing record for historical tracking
- Enables location history analysis and rider movement patterns
- Supports offline location tracking when rider reconnects
- Used by RidersGateway sendLocation event handler

**saveFcmToken**
- Inserts or updates FCM token tracking table with user ID, token string, role, and timestamp
- Uses ON DUPLICATE KEY UPDATE for idempotency - safe to call multiple times
- Automatically updates role and timestamp if token already exists
- Supports multiple tokens per user (different devices)
- Called during rider registration and token refresh operations
- Critical for push notification delivery

**deleteFcmToken**
- Deletes specific token for a given user ID from FCM token tracking table
- Used when user logs out or removes device
- Ensures notifications don't go to inactive devices
- Part of token lifecycle management

**deleteInvalidFcmToken**
- Deletes token by token value alone (without user ID)
- Called automatically when FCM service returns invalid token error
- Prevents repeated failed notification attempts
- Maintains token hygiene by removing expired/unregistered tokens
- Triggered by FirebaseService error handling

**getFcmTokens**
- Retrieves all active FCM tokens for a specific user ID
- Returns array of token strings for multi-device support
- Used by notification services to send to all user devices
- Supports fallback when primary token fails

### 10. Controllers

Controllers provide REST API endpoints for external system integration and administrative operations. These controllers expose the application's functionality through HTTP requests, enabling external services, mobile apps, and admin dashboards to interact with the NestJS backend. Each controller delegates business logic to corresponding services while handling request validation, routing, and response formatting.

#### Order Chat Controller

REST endpoints for order/chat management, providing full CRUD operations for messages and order-related actions:

- **POST endpoint for creating messages**: Create new message with automatic mention parsing and rider notification
- **GET endpoint for listing messages**: List messages with filters (chatType for room filtering, updatedAt for incremental sync, limit/offset for pagination)
- **GET endpoint for messages by chat type**: Get messages by chat type for initial room load
- **GET endpoint for single message**: Get single message by ID for detail view
- **PUT endpoint for updating messages**: Update message with rider assignment changes and text edits
- **PUT endpoint for updating by external ID**: Update by external system ID for external system integration
- **PUT endpoint for updating message status**: Update message status field
- **PUT endpoint for marking as read**: Mark message as read by specific user (updates seenBy array)
- **PUT endpoint for adding reactions**: Add emoji reaction to message with user ID and emoji
- **DELETE endpoint for removing reactions**: Remove emoji reaction from message
- **PUT endpoint for soft delete**: Soft delete message (marks isDeleted flag, updates realtime, notifies riders)
- **DELETE endpoint for hard delete**: Hard delete message (permanent removal from document database)
- **GET endpoint for message context**: Get message context with surrounding messages for jump-to-message functionality
- **GET endpoint for searching messages**: Search messages with filters (text, sender, mention, time range, pagination)

#### Rider Status Controller

Endpoints for rider status management, providing triggers for rider-related operations and status updates:

- **POST endpoint for order notification trigger**: Trigger new order notification to riders via RiderNotificationService
- **POST endpoint for rider status update**: Update rider status in realtime tracking table for activity tracking
- **POST endpoint for rider location update**: Update rider GPS location via UserService.saveRiderLocation
- **POST endpoint for FCM token save**: Save rider FCM token via UserService.saveFcmToken for push notifications

#### Pickup Way Controller

Endpoints for pickup way management, handling delivery route and pickup point notifications:

- **POST endpoint for pickup way trigger**: Trigger pickup way update notification to riders for route changes

#### Admin Controllers

Admin-specific controllers for administrative operations and FCM token management:

- **Admin Controller**: Admin-specific operations including user management and system configuration
- **Admin FCM Controller**: Admin FCM token management for push notification delivery to admin devices

### 11. Database Service

The DatabaseService provides relational database connection pool and query execution, serving as the primary interface for all relational database operations. This service manages connection pooling, query execution, and SQL injection prevention through parameterized queries and escape functions. It abstracts the underlying database driver while providing a consistent API for all services that need to interact with the relational database.

#### Features
- **Connection Pool**: Promise-based relational database connection pool for efficient resource management and connection reuse
- **Query Execution**: Parameterized queries using prepared statements for security against SQL injection
- **Escape Function**: SQL injection prevention through string escaping for dynamic SQL construction
- **Error Handling**: Centralized error logging with try-catch blocks for consistent error reporting
- **Transaction Support**: Implicit transaction handling through connection pool
- **Query Methods**: Separate methods for execute (with parameters) and query (for SELECT operations)

### 12. Date Service

The DateService handles date/time conversions and formatting, providing consistent timezone handling across the application. This service ensures all timestamps are stored in UTC for database consistency while being displayed in local timezone (Asia/Yangon) for user-facing applications. It handles various date formats including Unix timestamps, ISO strings, and display-ready formatted strings.

#### Features
- **UTC Date Generation**: Current UTC timestamp for consistent database storage
- **Yangon Time Conversion**: UTC to Asia/Yangon timezone conversion for local display
- **Timestamp Conversion**: Unix timestamp to UTC date object for date manipulation
- **Order Date Formatting**: Converts order document dates to human-readable display format with time
- **Timezone Consistency**: Ensures all date operations use consistent timezone handling
- **Display Formatting**: Formats dates for user interface display with appropriate locale settings

### 13. Schemas

Schemas define the document database structure for data persistence, ensuring consistent data models across the application. These schemas are used by Mongoose for document database operations, providing validation, default values, and type checking for all stored documents.

#### Order Schema

Document database structure for orders, representing chat messages and delivery orders with comprehensive metadata:

- **id**: Numeric message ID for API references and URL routing
- **chatType**: Chat room identifier for message grouping and room-based retrieval
- **senderId**: Sender user ID for message authorship tracking and permission checks
- **text**: Message text content with support for Unicode and Myanmar language
- **attachments**: Array of image attachments with id, url, filename, size fields for media handling
- **time**: Display time string formatted for user interface presentation
- **timestamp**: Unix timestamp for chronological ordering and time-based queries
- **status**: Message status field for workflow tracking (pending, delivered, cancelled)
- **messageType**: Type indicator ('message' for chat, 'order' for delivery orders)
- **isCancelled**: Cancellation flag for order lifecycle management
- **reactions**: Array of emoji reactions with userId, emoji, createdAt for engagement tracking
- **seenBy**: Array of user IDs who have read the message for read receipt tracking
- **replyTo**: Reference to parent message for threaded conversations
- **mentions**: Array of mentioned users with userId and userName for rider assignment
- **edited**: Boolean flag indicating if message has been edited
- **editedBy**: User ID of the editor for edit history tracking
- **editedAt**: Timestamp of last edit for audit trail
- **editedHistory**: Array of edit history entries with previousText, newText, editedBy, editedAt
- **isDeleted**: Soft delete flag for message removal without data loss
- **deletedAt**: Timestamp of deletion for audit trail and recovery
- **deletedBy**: User ID of deleter for accountability
- **externalSystemId**: External system message ID for external system integration and synchronization

#### Counter Schema

Document database counter for auto-incrementing order IDs, ensuring unique sequential message IDs across the system. This schema maintains a single counter document that is atomically incremented for each new message, preventing ID collisions and providing predictable ID sequences for API references and URL routing.

## Data Flow

This section describes the end-to-end data flows for key operations in the NestJS application, showing how data moves through various components, services, and external systems.

### Chat Message Flow

The chat message flow handles the creation, persistence, and distribution of messages from admin users to other admins and riders:

1. **Client sends message** via Socket.IO (send-message event) with message DTO containing text, attachments, and metadata
2. **ChatGateway validates** sender ID against authenticated user from client.data.userId to prevent message spoofing
3. **OrdersService transforms** message DTO using transformMessageDto to normalize input format (images → attachments, URL handling, default values)
4. **OrdersService parses** mentions from text using regex @(\w+) pattern and resolves to user IDs via UserService.getUserIdByUsername
5. **Document database save** creates order document with mentions array, attachments, and metadata via Mongoose
6. **FastAPI call** for order parsing if messageType is 'order' to extract structured delivery data (fee, ways, locations) for rider assignment
7. **Rider notifications** sent to all mentioned riders via RiderNotificationService (socket events to rider rooms + FCM push notifications)
8. **Acknowledgment** sent to sender with server-generated ID for optimistic UI updates
9. **Broadcast** to all chat room participants via Socket.IO for real-time message display
10. **FCM notifications** sent to mentioned users and reply targets via FirebaseService for offline delivery

This flow ensures messages are persisted, distributed to relevant parties, and synchronized across all connected clients while maintaining data consistency between document database, relational database, and external services.

### Rider Location Flow

The rider location flow handles GPS tracking from rider devices to admin dashboard displays:

1. **Rider sends location** via Socket.IO (sendLocation event) with latitude, longitude, battery percentage, charging status, and online status
2. **RidersGateway broadcasts** location globally via updateLocations event to all connected admin clients for map display
3. **UserService saves** location to rider location tracking table asynchronously via saveRiderLocation to avoid blocking socket response
4. **Admin clients receive** real-time location updates and display rider positions on interactive map with battery and charging indicators

This flow enables real-time rider tracking on the admin dashboard map, allowing admins to view rider positions, battery levels, and online status for efficient dispatch and monitoring. The asynchronous database save ensures socket responsiveness while maintaining location history for analysis.

### Order Assignment Flow

The order assignment flow handles the process of assigning orders to riders through @username mentions in chat messages:

1. **Admin creates order** with @rider mentions in chat message, specifying which riders should receive the order
2. **OrdersService parses** mentions from text using regex @(\w+) pattern and resolves to user IDs via UserService.getUserIdByUsername
3. **Document database save** creates order document with mentions array containing userId and userName for each mentioned rider
4. **RealtimeService adds** order ID to each rider's realtime tracking table via rtAction with mode 'add' for order assignment tracking
5. **RiderNotificationService emits** new-order event to each rider's personal socket room via RidersGateway for real-time delivery when rider is online
6. **RiderNotificationService sends** FCM notification to each rider via FirebaseService for offline delivery with order details and action type
7. **Admin clients receive** admin-row-update event globally to refresh dashboard with updated rider status and assigned orders

This flow ensures orders are assigned to the correct riders, tracked in the realtime system for operational visibility, and delivered through both real-time sockets (for online riders) and push notifications (for offline riders). The admin dashboard stays synchronized with rider assignments through global events.

## Security Considerations

The NestJS application implements multiple layers of security to protect against common vulnerabilities and ensure data integrity:

- **Token Validation**: Admin session tokens validated on WebSocket connection against admin_sessions table joined with users table, checking token validity and expiration before allowing connection
- **SQL Injection Prevention**: Parameterized queries using prepared statements for all database operations, with escape function for dynamic SQL construction in RealtimeService
- **CORS Configuration**: Restricted to specific domains (cherrylandtaunggyi.com) with credentials support for secure cross-origin requests
- **User Authorization**: Sender ID validation for message operations to prevent message spoofing and ensure users can only act as themselves
- **Soft Delete**: Messages marked deleted instead of actual removal for audit trail and data recovery capabilities
- **Input Validation**: Message DTO transformation and mention parsing with regex validation to prevent malformed input
- **Error Handling**: Centralized error logging without exposing sensitive database details to clients
- **Credential Protection**: Database credentials and Firebase service account stored in environment variables and JSON files outside version control

## Performance Optimizations

The NestJS application implements several performance optimizations to ensure efficient resource utilization and responsive user experience:

- **Connection Pooling**: Relational database connection pool for efficient resource management and connection reuse, reducing connection overhead for high-volume operations
- **In-Memory Caching**: FCM tokens and rider names cached in memory to reduce database queries for frequently accessed data, improving notification delivery speed
- **Duplicate Token Cleanup**: Automatic cleanup of duplicate FCM token entries on module initialization to prevent redundant notification attempts and reduce database load
- **Batch Operations**: Support for bulk rider updates via batchRtAction to process multiple rider assignments in a single operation, reducing database round trips
- **Pagination**: All list operations support pagination with limit and offset parameters to prevent large dataset transfers and enable efficient incremental loading
- **Selective Fields**: Excludes soft-delete fields (isDeleted, deletedAt, deletedBy) from queries to reduce data transfer size and improve query performance
- **Asynchronous Operations**: Location saves and non-critical database operations performed asynchronously to avoid blocking socket responses
- **Auto-Rejoin**: Automatic chat room rejoin on message send prevents dropped messages during temporary disconnections without additional database queries

## Implementation Details

This section provides implementation-specific details about how the NestJS application is structured and how components interact at the code level.

### Dependency Injection Pattern

The application uses NestJS's built-in dependency injection system for service management:

- **Constructor Injection**: All services receive dependencies through constructor parameters with TypeScript decorators (@Inject for circular dependencies)
- **Circular Dependencies**: ForwardRef() used for circular dependencies between ChatGateway and FirebaseService, RidersGateway and FirebaseService
- **Module-level Providers**: Services registered as providers in their respective modules (DatabaseModule, FirebaseModule, OrdersModule, etc.)
- **Global Availability**: Root-level services (AppService) available across the application, module-scoped services available within their modules
- **Lazy Loading**: Services instantiated only when needed by the NestJS DI container

### Module Organization

The application follows NestJS's modular architecture with clear separation of concerns:

- **AppModule**: Root module importing all feature modules and configuring global providers
- **DatabaseModule**: Encapsulates relational database connection and query execution
- **FirebaseModule**: Manages Firebase Admin SDK initialization and notification sending
- **WebsocketsModule**: Contains both ChatGateway and RidersGateway for real-time communication
- **OrdersModule**: Handles order documents, schemas, and business logic
- **Controllers**: REST endpoints organized by functional area (orders, riders, pickup ways, admin)

### Service Interdependencies

Services interact through well-defined interfaces with clear dependencies:

- **ChatGateway → OrdersService**: Message creation, updates, and search operations
- **ChatGateway → UserService**: User ID resolution and username lookups
- **ChatGateway → RealtimeService**: Realtime tracking table updates for order assignments
- **ChatGateway → RiderNotificationService**: Rider notification coordination
- **ChatGateway → FirebaseService**: Admin FCM notifications
- **RidersGateway → UserService**: Rider location saving and username resolution
- **RidersGateway → FirebaseService**: Rider FCM notifications
- **OrdersService → UserService**: Mention resolution via username-to-ID mapping
- **OrdersService → RealtimeService**: Rider assignment updates via realtime tracking table
- **OrdersService → RiderNotificationService**: Rider notifications for order changes
- **RiderNotificationService → RidersGateway**: Socket room targeting for rider notifications
- **RiderNotificationService → FirebaseService**: FCM push notifications for riders
- **RealtimeService → DatabaseService**: Realtime tracking table operations
- **UserService → DatabaseService**: All user-related database operations

### WebSocket Event Handling Pattern

Both gateways follow a consistent pattern for WebSocket event handling:

1. **Event Reception**: Socket.IO event listener receives data from client
2. **Validation**: Input validation and authentication checks (token validation, user ID verification)
3. **Business Logic**: Service method call for data processing and persistence
4. **Database Operations**: Service performs necessary database operations via DatabaseService
5. **External Service Calls**: Integration with FastAPI, Firebase, or other external services
6. **Response Emission**: Socket.IO event emission back to client(s) with result
7. **Error Handling**: Try-catch blocks with error logging and graceful error responses

### Error Handling Strategy

The application implements a layered error handling approach:

- **Service Level**: Try-catch blocks in service methods with console.error logging for debugging
- **Gateway Level**: Error handling in event handlers with error responses to clients
- **Controller Level**: NestJS exception filters for HTTP endpoint error handling
- **Database Level**: Connection pool error handling with automatic reconnection
- **FCM Level**: Invalid token detection with automatic token cleanup via UserService
- **Graceful Degradation**: Fallback values (null, empty arrays, default strings) when operations fail

### Data Consistency Patterns

The application ensures data consistency across multiple data stores:

- **Atomic Updates**: Single UPDATE statements for realtime tracking table modifications
- **Soft Delete**: Messages marked deleted instead of removal for audit trail
- **Synchronized Operations**: Updates to document database, relational database, and realtime tracking table coordinated in single operations
- **Transaction-like Behavior**: Multiple related operations performed sequentially with error rollback logic
- **Event-driven Sync**: Socket.IO events broadcast to ensure all clients receive updates simultaneously

## External Integrations

The NestJS application integrates with several external services to provide comprehensive functionality:

- **FastAPI**: Order parsing service for structured delivery data extraction from natural language messages, parsing delivery fee, pickup/dropoff locations, and route information
- **Firebase Cloud Messaging**: Push notifications for mobile and web platforms with multi-platform support (Android, iOS, Web Push) for offline delivery
- **Image Service**: External image server for attachment URLs, providing CDN-based image storage and delivery with URL transformation
- **Relational Database**: Operational database for users, sessions, realtime tracking data, FCM tokens, and rider location history
- **Document Database**: NoSQL database for chat and order history with schema validation and flexible document structure
