# CherryLand Taunggyi — Code Execution Architecture

CherryLand is one connected delivery and marketplace platform. An office message can become a chat record, a structured delivery order, a rider notification, a rider task, a realtime dashboard update, and an operational settlement. This document follows those connections from function to function.

It is intentionally written from the source code: entry event → conditions → called function → data write → output event or HTTP response.

## 1. Runtime entry map

~~~mermaid
flowchart LR
  WEB[Public browser] --> LWEB[Laravel web routes]
  ADM[Admin Vue PWA] --> SIO[Socket.IO]
  ADM --> LAPI[Laravel marketplace API]
  RID[Rider PWA] --> RPHP[PHP rider API]
  SIO --> CHAT[ChatGateway]
  CHAT --> ORD[OrdersService]
  ORD --> MONGO[(MongoDB)]
  ORD --> PARSE[FastAPI parser]
  PARSE --> OPS[(Relational database operations)]
  RPHP --> OPS
  RPHP --> NREST[NestJS REST]
  LAPI --> LCTRL[Laravel controllers]
  LCTRL --> MARKET[(Marketplace relational database)]
  IMG[Image request] --> GIN[Go image service]
  GIN --> FILES[(WebP file storage)]
~~~

| Request source | Code entry | First application dispatch |
|---|---|---|
| Admin chat | Admin socket client | NestJS ChatGateway Socket.IO event |
| Admin marketplace | Admin marketplace API | Laravel API routes |
| Rider PWA | Rider app wrapper and scripts | Rider PHP endpoint |
| Public marketplace | Laravel web routes | Laravel web controller |
| Media client | Image service | Gin middleware then Go handler |

## 2. Application boot chain

### 2.1 Compose service connectivity

Docker Compose places the application containers on visnnara-network. The code uses the following internal host names.

| Caller | Host name used | Called workload |
|---|---|---|
| NestJS OrdersService | fastapi-app | FastAPI order parser |
| NestJS gateways/services | relational database | operational users, sessions, realtime, FCM data |
| NestJS Mongoose setup | mongodb | chat message documents |
| PHP rider API | NestJS app through environment variable | orders/reaction/realtime bridge |
| Laravel | relational database | marketplace DB and office DB connection |
| Go image service | relational database | session lookup at image mutation time |

### 2.2 NestJS starts its runtime graph

1. NestJS bootstrap creates AppModule.
2. bootstrap enables CORS, chooses PORT or 80, and listens on 0.0.0.0.
3. AppModule loads DatabaseModule. DatabaseService constructor creates one MySQL promise pool.
4. AppModule loads WebsocketsModule. ChatGateway and RidersGateway become Socket.IO handlers.
5. AppModule opens MongoDB using Mongoose, then OrdersModule registers Order and counter schemas.
6. AppModule loads FirebaseModule, used by chat and rider notification services.
7. ChatGateway.onModuleInit loads administrator FCM tokens and removes duplicate token records.
8. RidersGateway.onModuleInit loads rider/developer FCM tokens into memory.
9. Controller decorators register HTTP routes; SubscribeMessage decorators register Socket.IO event names.

### 2.3 FastAPI starts parser functions

1. FastAPI main loads environment values into DB_CONFIG.
2. get_db opens a PyMySQL connection only when a handler enters a database block.
3. The context manager commits after normal handler completion, rolls back on an exception, and closes in finally.
4. FastAPI registers parse_only, parse_and_save, and health functions.

### 2.4 Go image service starts handler closures

1. Image service main calls database.InitDB.
2. Handlers package init loops servicePaths and creates all storage directories.
3. Gin registers CORS and Logger globally.
4. main registers each public GET image route.
5. Each upload/delete group runs InternalAuth before UploadImage or DeleteImage.

## 3. Admin chat: one new message through every function

### 3.1 Browser-side message dispatch

Admin orders API createMessage receives messageData.

1. It gets the shared socket client.
2. When socketClient.isConnected is true, it sends Socket.IO event send-message and immediately resolves the optimistic local message.
3. When the socket is unavailable, it sends HTTP POST to orders endpoint as fallback.
4. The Vue chat data layer uses Dexie chatDB. Incoming messages are stored locally with serverId; incoming attachment URLs are fetched and their blobs are saved as local image records for offline display.

### 3.2 Socket event: ChatGateway.handleSendMessage

Source: NestJS chat gateway handleSendMessage.

| Step | Exact runtime action | Result |
|---:|---|---|
| 1 | Read client.data.userId, which handleConnection set from the socket handshake. | Current socket identity. |
| 2 | Compare String(message.senderId) against String(userId). | Different values emit message-acknowledged with Unauthorized and return. |
| 3 | Construct room name from chatType. | Target chat room. |
| 4 | Await client.join(roomName). | Sender joins even after reconnect. |
| 5 | Await ordersService.create(message). | Mongo message write plus order-specific side effects. |
| 6 | Emit message-acknowledged to sender with tempId and savedMessage.id. | Client replaces optimistic identity with server ID. |
| 7 | Build broadcast object without incoming images/imageUrls and with Mongo attachments. | One normalized wire payload. |
| 8 | Set messageType to message if absent. | Normal message type is explicit. |
| 9 | Emit new-message to the room. | All room listeners receive the saved message. |
| 10 | Build a Set from mentioned users and reply target, excluding sender. | De-duplicated push recipients. |
| 11 | For each recipient call getAdminFcmTokens and FirebaseService.sendAdminNotification. | Mention/reply push delivery. |

~~~mermaid
sequenceDiagram
  participant UI as Admin PWA
  participant G as ChatGateway
  participant O as OrdersService
  participant M as MongoDB
  participant P as FastAPI
  participant D as Relational database
  participant F as Firebase
  UI->>G: send message event
  G->>G: sender ID comparison
  G->>O: create message
  O->>M: save chat document
  O->>P: parse and save for order type
  P->>D: synchronize assignee order rows
  G-->>UI: message acknowledged with IDs
  G-->>UI: new message broadcast
  G->>F: mention or reply push notification
~~~

### 3.3 OrdersService.create: input normalization, Mongo write, parser call

Source: NestJS orders service transformMessageDto, create, parseMentions, sendToFastAPI.

1. transformMessageDto starts with an empty attachments array.
2. If input has images, it maps each object. A URL-based image becomes id, url, filename, size. A legacy inline object becomes type, base64/data URL, size.
3. If input instead has imageUrls, it splits each URL, uses the final path segment as image id, produces a WebP filename, and assigns size zero.
4. It returns exactly these persisted fields: chatType, senderId, text, attachments, time, timestamp, status, messageType, isCancelled false, reactions, seenBy, replyTo.
5. parseMentions runs regex @ followed by word characters. Every discovered username calls UserService.getUserIdByUsername against the operational users table.
6. create spreads the normalized message and resolved mentions into a new Mongoose Order model, then awaits save.
7. For messageType order with nonempty text, it resolves sender username with UserService.getUsernameById and invokes sendToFastAPI.
8. The same order branch loops resolved mentions and invokes RiderNotificationService.notifyRider with action create, order ID, raw text, chat ID, sender and rider identity.
9. The Mongo model returned by save becomes the ChatGateway acknowledgement and broadcast source.

### 3.4 FastAPI parse-and-save: raw text becomes per-rider rows

NestJS sends this shape to FastAPI:

~~~json
{
  "orderId": "Mongo message ID",
  "submitted_by": "sender username",
  "text": "raw order text",
  "status": "pending"
}
~~~

Source: FastAPI parse_and_save handler.

| Step | Function or expression | Effect |
|---:|---|---|
| 1 | OrderInput | Requires integer orderId, submitted_by, nonblank text; supplies pending default status. |
| 2 | DeliveryParser.parse(order.text) | Returns fee, ways, total complexity, pickup list, delivery object, warnings. |
| 3 | extract_mentions(order.text) | Returns unique @username values in first-seen order. |
| 4 | no-mention branch | Uses submitted_by as the single owner username. |
| 5 | JSON serialization | pickup_details is JSON pickup array; delivery_ph is JSON phone array; warnings is JSON warning array. |
| 6 | SELECT on orders table by order ID | Reads the usernames already owning rows for this order. |
| 7 | Set operations | Computes to_delete, to_keep, and to_add from old/new username sets. |
| 8 | delete loop | Deletes rows of removed assignees. |
| 9 | update loop | Refreshes all structured columns for retained assignees. |
| 10 | insert loop | Inserts all structured columns for new assignees. |
| 11 | get_db exit | Commits all loops as one connection context; exception path rolls back. |
| 12 | response object | Returns added, updated, removed usernames, inserted IDs, parser output and warnings. |

This means one order message can create several relational database orders records. They share orderId but differ by assigned username.

### 3.5 DeliveryParser: exact parsing stages

Source: FastAPI DeliveryParser class.

1. parse strips raw text.
2. It applies RX.FEE. The captured Burmese/Arabic number goes through to_en then smart_int, which removes commas and returns an integer.
3. It applies RX.WAYS. The parsed number is clamped with max(value, 1).
4. It splits the text with RX.SECTION_SPLIT into pickup_raw and delivery_raw. If no delivery section exists, it tries inline To. When both fail it adds a missing-delivery warning.
5. _parse_pickups splits pickup text by line. is_noise removes headers, mentions, dividers, emoji-only lines, and blank lines.
6. extract_phones returns unique normalized phone values and the phone-removed line.
7. A line begins a new pickup when it matches pickup keyword rules or when there is no current pickup and the line is not a mention/item line.
8. New pickup logic removes pickup markers/suffixes, cleans symbols, looks ahead one line, then classify_pickup_type assigns passenger, shopping, gate, food, or default pickup plus complexity.
9. Lines inside active pickup add phones, classify short unit-containing lines as details, and append other text into shop_notes.
10. If no pickup is found, parser creates one Unknown Pickup fallback and adds warning.
11. _parse_delivery again extracts phones, skips fee-only lines, marks note-keyword/emoji lines as notes, marks a short first non-address line as name, and joins remaining lines into address.
12. parse returns ParsedOrder with comp equal to the sum of pickup complexity values.

## 4. Chat reads, edits, deletion, cancellation and reactions

### 4.1 REST reader functions

OrderChatController maps each HTTP route into OrdersService.

| Route | Controller function | Service flow |
|---|---|---|
| GET orders endpoint | findAll | Optional chatType and updatedAt query; sort updatedAt descending; skip/limit; hides soft-delete fields; transforms Yangon dates and relative image URLs. |
| GET orders by chat type | findByChatType | Reads chat room documents sorted by timestamp; transforms dates and media URLs. |
| GET order by ID | findOne | Parses numeric ID; reads one Mongo document. |
| GET order context | getMessageContext | Reads target, older IDs, newer IDs; reverses old list and joins chronological context. |
| GET order search | searchMessages | Builds Mongo filter from text regex, sender, mention, time range, limit, offset. |

transformImageUrls checks every attachment. Any URL not starting with HTTP receives IMAGE_SERVER_URL as prefix before API response. DateService converts date fields to Yangon display values.

### 4.2 Socket edit event

Source: ChatGateway.handleEditMessage plus OrdersService.update.

1. Gateway loads current document using findOne.
2. It parses mentions from replacement text and resolves every username to operational user ID.
3. It builds updateData with text, mentions, edited true, editor ID, UTC edit time, and optional image IDs/attachment replacement.
4. If previousText was sent by UI, gateway appends an editedHistory object to prior history.
5. Gateway awaits OrdersService.update.
6. update reads the pre-change Mongo record, then findOneAndUpdate returns the new one.
7. It turns old/new mentions into two string ID sets.
8. Each old ID absent from new set: resolve username, RealtimeService.rtAction remove, then RiderNotificationService action remove.
9. Each new ID absent from old set: resolve username, RealtimeService.rtAction add, then RiderNotificationService action assign with updated raw text.
10. When text changed, each retained rider receives RiderNotificationService action edit.
11. When changed document is an order, update calls sendToFastAPI to refresh structured orders rows.
12. Gateway performs its own changed-order parse call as well, then emits message-edited to chat room and editor personal room.

### 4.3 Delete and cancel functions

| Event | Mongo mutation | Operational mutation | Socket output |
|---|---|---|---|
| delete-message | update sets isDeleted, deletedAt, deletedBy | Every mentioned rider gets realtime remove and rider remove notification; SQL deletes orders by orderId. | message-deleted to chat and personal room; delete-acknowledged to sender. |
| cancel-order true | update sets isCancelled true | Removes from every rider realtime list, notifies remove, deletes orders rows by orderId. | order-cancelled with cancellation flag. |
| cancel-order false | update clears cancellation state | Adds each rider back to realtime, sends create notification, reruns parser against stored text. | order-cancelled with restored flag. |

### 4.4 Reactions, read receipts and typing

1. add-reaction receives currentReactions, current socket user, and emoji.
2. It searches same user plus same emoji. Existing pair is removed; absent pair is appended with createdAt.
3. It stores calculated reaction array through OrdersService.update, then emits reaction-updated to chat room and personal room.
4. REST reaction PUT calls OrdersService.addReaction. It loads Mongoose model, only adds absent pair, saves model.
5. REST reaction DELETE calls removeReaction, filters matching pair, saves model.
6. mark-as-read Socket handler takes client supplied seenBy array and broadcasts message-read; it does not write through this Socket path.
7. typing-start places user ID into typingUsers for chatType then broadcasts user-typing true. typing-stop removes it and broadcasts false.

## 5. Rider path: login, assigned orders, accept, delivery, settlement

### 5.1 Rider login function

Source: Rider login API endpoint.

1. Includes database configuration and authentication helper.
2. Rejects non-POST call; decodes JSON body; requires username and password.
3. Prepared SQL finds users table by username or phone JSON member.
4. If stored password column empty, hashes submitted password and updates users table password column. Otherwise calls password_verify.
5. Generates session token with random_bytes and bin2hex; calculates 30-day expiresAt.
6. Creates rider sessions table if absent.
7. Inserts rider ID, token, device info, IP, user-agent, expiry.
8. Removes password column, adds token to response user object, and sends success JSON.

Every protected v3 endpoint runs verifyToken from authentication helper. It reads Authorization Bearer token, queries rider sessions table joined to users table, checks expiry, then returns the authenticated rider row.

### 5.2 Assigned order list

Source: Rider orders API endpoint.

~~~mermaid
sequenceDiagram
  participant R as Rider PWA
  participant PHP as orders endpoint
  participant N as NestJS GET orders
  participant M as MongoDB
  R->>PHP: Bearer token
  PHP->>PHP: verify token returns rider ID
  PHP->>N: GET orders endpoint
  N->>M: OrdersService find all
  M-->>N: chat/order documents
  N-->>PHP: message array
  PHP->>PHP: mention and state filter
  PHP-->>R: rider format JSON plus ETag
~~~

The PHP loop checks every message mention for userId matching current rider. It rejects messages where isDeleted, isCancelled, or status equals done. Kept messages are reshaped to chatId, orderId, rawText, is_edit, submitted_by, is_parsed, parsed_json and photos. The JSON MD5 becomes ETag; matching If-None-Match exits with 304.

### 5.3 Acceptance

Source: Rider order accept API endpoint.

1. Session is verified and JSON chatId/orderId is read.
2. PHP emits success to the rider immediately, flushes HTTP response, then continues background work.
3. It sends PUT to NestJS orders reaction endpoint with authenticated rider userId and heart emoji.
4. OrderChatController.addReaction calls OrdersService.addReaction.
5. OrdersService loads Mongo message, prevents duplicated matching heart, pushes new reaction when absent, then saves.

Current code uses this heart reaction as acceptance persistence.

### 5.4 Delivery action

Source: Rider order action API endpoint delivered branch.

1. Reads order ID, rider username, action, coordinate and timestamp.
2. Reads current realtime table row for that username.
3. Splits realtime order ID comma list, removes delivered ID, rejects when the ID was not in list.
4. Updates operational orders table status to delivered and sets username.
5. Updates order raw table status flag to 1 for that rider/order.
6. fastNotify posts action remove to NestJS new order trigger endpoint.
7. If list becomes empty, rewrites realtime table status into ready state, clears order ID, updates timestamp, posts all completed trigger.
8. If other IDs stay, writes the remaining comma list and timestamp.
9. Returns status success with all_done boolean.

### 5.5 Fee, way count and settlement actions

| Action field | Code branch | Storage operation |
|---|---|---|
| update_fee | First searches orders table for matching rider/order; then ways table if no order. | Updates orders table delivery fee where billing sent flag is 0, or ways table delivery fee. |
| update_way_count | Uses matching rider/order and billing sent flag 0. | Updates orders table way count. |
| settle | Starts database transaction after confirming submitting user. | Deletes existing settlements for order/rider, parses note lines into payer plus amount, inserts settlement rows, then completes transaction. |

## 6. Rider realtime socket and notification fabric

### 6.1 join-rider-room

Source: NestJS riders gateway handleJoinRiderRoom.

1. Trims incoming rider ID.
2. Stores it as client.data.userId, adds it to onlineRiders, and joins Socket.IO room named exactly rider ID.
3. Updates the latest rider locations table record to online status flag 1.
4. When no cached name exists, selects username from users table and fills rider names cache and username to ID cache.
5. Calls broadcastOnlineCount, which emits online-riders-count globally.

Disconnect does the inverse operational update: latest rider locations table becomes offline, rider offline emits, and online count refreshes.

### 6.2 sendLocation

1. Requires userId, lat and lng.
2. Resolves username from cache or UserService.getUsernameById.
3. Emits updateLocations globally with GPS, client status, rider identity and online flag.
4. Calls UserService.saveRiderLocation asynchronously with battery, charging and online values.

### 6.3 rider-done-order

1. One SQL query joins users table, realtime table, and the newest rider locations table row.
2. Gateway parses realtime table status JSON with ready fallback.
3. It converts realtime table order ID into an array.
4. It creates formattedRiderData with identity, activity/thermal/homeway/order status and GPS/device state.
5. It emits rider-refresh-row, allowing admin table refresh with no browser reload.

### 6.4 One notification function, several outputs

Source: NestJS rider notification service notifyRider.

1. identifier becomes username when supplied, otherwise rider ID column.
2. RiderService.resolveRiderId converts it to current rider socket room ID.
3. action add_photo emits order-photo-update to that room and sends Firebase order-photo-update.
4. action all_completed emits admin-row-update globally.
5. action remove emits new-order action remove to rider room, admin-row-update globally, and Firebase order-remove.
6. all other actions construct orderData, emit new-order to rider room, emit admin-row-update globally, then send Firebase new-order.

## 7. NestJS HTTP bridge used by PHP

| HTTP route | Controller function | Downstream behavior |
|---|---|---|
| POST new order trigger endpoint | OrderController.newOrderTrigger | Sends body into RiderNotificationService.notifyRider. |
| POST rider status trigger endpoint | RiderStatusController.updateRiderStatusTrigger | Routes legacy status changes into realtime system. |
| POST rider location update endpoint | RiderStatusController.updateRiderLocation | Uses user service/gateway location workflow. |
| POST rider FCM token endpoint | RiderStatusController.saveFcmToken | Persists rider push token. |
| POST pickup way trigger endpoint | PickupWayController.pickupWayTrigger | Emits pickup way update path. |
| POST admin config update endpoint | AdminController.configUpdate | Invokes administrative refresh behavior. |

## 9. Marketplace public page functions

### 9.1 Route model binding

Laravel routes bind slug values into Business, Category, and Product. All three models return slug from getRouteKeyName. The bound model enters controller method before controller conditions execute.

### 9.2 Public controller execution

| Method | Query/function path | Returned view |
|---|---|---|
| HomeController.index | Four independent queries: featured restaurants six, food categories six, featured food products eight, popular restaurants eight; then static vertical descriptors. | home |
| RestaurantController.index | active restaurant businesses, sort_order, paginate 12. | restaurants.index |
| RestaurantController.show | inactive means 404; active products grouped by category ID; related categories eager-loaded then unique/sorted. | restaurants.show |
| MenuController.index | active ordered food categories plus available featured food products. | menu.index |
| MenuController.category | inactive means 404; active available food products paginated 12. | menu.category |
| MenuController.show | unavailable means 404; fetches four random available same-type products excluding current. | menu.show |
| MenuController.overall | eager loads business/category/images; newest first; paginates 15. | menu.overall |
| MenuController.overallShop | first-or-fail business by slug; eager-loads only that business overall posts. | menu.overall-shop |
| SearchController.index | Empty q returns three empty collections; nonempty q searches business/product/category name and description with active/available scope. | search.index |
| CartController.products | Normalizes array/comma product IDs to ints; whereIn plus business eager load; keyBy id; returns JSON. | JSON product map |

## 10. Marketplace admin: client function to transactional data write

### 10.1 Vue API client

Admin marketplace API obtains admin_token from document.cookie, places it in Authorization Bearer header, then maps UI action to these endpoint families:

~~~text
/api/market/businesses
/api/market/categories
/api/market/products
/api/market/overall
~~~

### 10.2 Laravel function chain

1. API routes prefix market combines with Laravel API prefix to form /api/market paths.
2. AuthenticateMarketplaceAdmin.handle reads Bearer token, then cookie fallback.
3. It checks format, queries office admin sessions table joined to users table, verifies active/unexpired non-rider session, and places authenticated user on request attributes.
4. Mutating endpoints call CheckMarketplacePermission.handle after authentication. That function reads request attached role and compares it to permission mapping/hierarchy.
5. Controller executes Eloquent query or write on marketplace database.

### 10.3 Business, category and product CRUD

| Action | Controller logic |
|---|---|
| index | Starts model query; selectively adds query-parameter filters; orders sort_order/name; paginates; emits Resource collection plus meta. |
| show | Finds numeric ID; absent row produces 404 JSON; present row passes through Resource. |
| store | Form Request validates body; Model create receives validated data; Resource returns HTTP 201. |
| update | Numeric find, Form Request validation, model update, fresh Resource response. |
| destroy | Numeric find, model delete, no-content success response. |

### 10.4 Overall content has parent/child transaction flow

OverallController.store validates business/category/featured/images, starts transaction, creates one overall parent, creates one OverallImage child for each image URL, eager loads parent relations, commits, and returns populated parent.

OverallController.update updates parent first. When images is present it deletes all old child rows and creates replacement rows before commit. destroy removes image children then parent in one transaction.

## 11. Image code path

### 11.1 Route closure selection

Image service main binds a distinct service key to each route group. Examples:

| Request | Handler closure key |
|---|---|
| GET orders attachment endpoint | GetImage orders |
| POST orders attachment upload | InternalAuth then UploadImage orders |
| GET marketplace products endpoint | GetImage marketplace_products |
| POST marketplace products upload | InternalAuth then UploadImage marketplace_products |

Profile, voucher, settlement, payment, business logo, business cover, category and overall use the same closure design with their own service key.

### 11.2 InternalAuth chain

1. Reads X-Internal-Auth, INTERNAL_AUTH_KEY and X-Session-Token.
2. Matching internal key calls Gin next immediately.
3. Otherwise queries admin sessions table joined to users table with session token.
4. Then queries rider sessions table joined to users table with same token.
5. Valid admin attaches user ID column and admin session type to Gin context before next.
6. Valid rider attaches rider ID column and rider session type before next.
7. Remaining branch completes before an invalid mutation request reaches handler.

### 11.3 UploadImage byte flow

1. Reads multipart field image.
2. Reads original extension and rejects formats outside jpg/jpeg/png/gif/webp.
3. Opens file and reads file.Size bytes into memory.
4. converter.ValidateImage validates bytes.
5. converter.ConvertToWebP transforms it using configured quality 75.
6. getServicePath selects directory. marketplace products selects storage/marketplace/products.
7. uuid.New creates ID, .webp is appended, filepath.Join creates destination.
8. os.WriteFile writes bytes.
9. Service mapping creates relative URL such as /api/v1/marketplace/products/UUID.
10. Handler responds HTTP 201 with id, filename, size, MIME type, format, service and URL.

GetImage repeats path construction with requested id, reads the WebP bytes, writes one-year immutable cache header and sends WebP data. DeleteImage builds same path and removes the file.

## 12. Data writes by execution flow

~~~mermaid
flowchart TB
  A[Admin Socket order] --> MC[Mongo Order]
  A --> FP[FastAPI parser]
  FP --> OR[Relational database orders per mention]
  A --> RT[Relational database realtime]
  R[Rider delivered] --> OR
  R --> RAW[Relational database order raw]
  R --> RT
  R --> ST[Relational database settlements or ways]
  M[Marketplace admin] --> CAT[Marketplace catalogue tables]
  M --> OV[overall and overall_image]
  I[Image upload] --> FS[UUID WebP file]
~~~

| Storage | Writer functions | Reader functions |
|---|---|---|
| Mongo Order documents | OrdersService create/update; ChatGateway events | admin chat REST/socket, rider get_orders through NestJS |
| Relational database orders | FastAPI parse_and_save; rider delivered/fee/way | rider settlement/update, operational UI |
| Relational database order raw | Admin normal/edit/assignment flow; rider delivery | Admin photo/assignment/edit flow |
| Relational database realtime | NestJS RealtimeService.rtAction; rider delivery | rider/admin realtime views, gateway rider refresh |
| Relational database rider sessions | rider login | PHP verifyToken, Go image InternalAuth |
| Relational database FCM tokens | token registration functions | gateway module init, notifyRider, admin push send |
| Marketplace tables | Laravel market controllers/OverallController | Laravel public controllers/admin API |
| WebP storage | UploadImage | GetImage and client URL fetch |

## 13. Primary code navigation

| Capability | Main source functions |
|---|---|
| Admin chat | ChatGateway handleConnection, handleSendMessage, handleEditMessage, handleDeleteMessage, handleCancelOrder |
| Chat data | OrdersService transformMessageDto, create, parseMentions, sendToFastAPI, findAll, update, getMessageContext, searchMessages |
| Rider live layer | RidersGateway handleJoinRiderRoom, handleSendLocation, handleRiderDoneOrder; RiderNotificationService notifyRider |
| Parser | DeliveryParser parse, _parse_pickups, _parse_delivery; parse_only, parse_and_save |
| Rider PHP | login endpoint, orders endpoint, order accept endpoint, order action endpoint, update order endpoint |
| Marketplace | HomeController, RestaurantController, MenuController, SearchController, CartController, Market API controllers, OverallController |
| Media | InternalAuth, UploadImage, GetImage, DeleteImage |

The code path is the CherryLand story: natural team communication flows directly into structured delivery execution, rider action, realtime office visibility, marketplace content, and optimized media delivery.

**Note:** The system previously used Telegram for order management and operations. All Telegram-related functionality has been removed and replaced with the company's own platform for order management and operations.

## 14. Portal Architecture Overview

### 14.1 Admin Portal (Progressive Web Application)

**Technology Stack:**
- Modern JavaScript framework with Composition API
- Build tooling for development and production
- Utility-first CSS framework for styling
- WebSocket client for real-time communication
- Cloud messaging service for push notifications
- IndexedDB wrapper for offline data storage
- Client-side routing library
- Data fetching and state management
- Mapping library for geographic visualization
- Charting library for data visualization
- PWA plugin for offline capabilities

**Application Architecture:**
The admin portal follows a component-based architecture with clear separation of concerns:

- **Application Bootstrap** - Initializes services, registers push tokens, sets up service workers, manages version checking
- **Routing Layer** - Permission-based route guards protect access to different sections
- **View Components** - Page-level components for different functional areas (dashboard, chat, realtime, marketplace)
- **Reusable Components** - Shared UI components including chat-specific elements
- **Composables** - Reusable logic for authentication, chat data, socket connections, permissions
- **Utilities** - Helper functions for dates, image handling, message search
- **API Clients** - HTTP clients for communicating with backend services
- **Local Database** - IndexedDB schema for offline message storage
- **Socket Client** - WebSocket configuration and connection management

**Key Architectural Patterns:**
- Permission-based access control with route guards
- Real-time communication via WebSocket with HTTP fallback
- Offline-first architecture using browser storage
- Push notification system for alerts
- Version management with forced update capability
- Rich text messaging with emoji support
- Image upload with preview functionality
- Message search and filtering capabilities
- Reaction system for message feedback
- Read receipts and typing indicators
- Admin panel for content management
- Real-time tracking dashboard
- Financial settlement management

**Authentication Flow:**
1. User authenticates through login interface
2. Session token stored in browser storage
3. Authentication state managed by composable
4. Route guards verify authentication and permissions
5. API requests include authentication token
6. Session validated against backend database

### 14.2 Rider Portal (Hybrid Native + Web Application)

**Technology Stack:**
- Cross-platform mobile framework with WebView component
- Native SDK for device capabilities (Location, Battery, Network, Notifications, Task Manager)
- WebSocket client for real-time communication
- Cloud messaging SDK for push notifications
- Local storage for data persistence
- Lightweight JavaScript framework for reactive UI in WebView
- Utility-first CSS framework for styling
- Charting library for data visualization
- Chart plugin for data labels
- Image cropping library
- Icon library

**Application Architecture:**
The rider portal uses a hybrid architecture combining native mobile capabilities with web technologies:

**Native Layer:**
- **Application Bootstrap** - Initializes native services, manages push tokens, handles background tasks
- **Location Services** - Foreground and background location tracking with permission handling
- **Background Task Management** - Continuous location tracking with battery-aware throttling
- **Distance-based Updates** - Triggers updates when movement exceeds threshold
- **WebSocket Management** - Connection lifecycle and state management
- **Native-Web Bridge** - Communication channel between native and web layers
- **Version Management** - Cache clearing and update handling
- **Push Message Handling** - Background and foreground notification processing
- **Token Refresh** - Automatic push token updates

**WebView Layer:**
- **Service Worker Integration** - PWA capabilities with cloud messaging
- **Reactive Components** - State management for UI components
- **WebSocket Client** - Real-time communication initialization
- **Data Visualization** - Chart library setup and configuration
- **Pull-to-Refresh** - Content refresh functionality
- **Modern UI Effects** - Glass morphism, dark mode, animations
- **Navigation System** - Tab-based navigation with active states
- **Smooth Transitions** - Animation effects for UI changes

**Authentication Layer:**
- **State Management** - Authentication state handling
- **Form Validation** - Input validation and error handling
- **Visual Design** - Modern UI with animations and effects
- **Session Storage** - Token persistence and management
- **Auto-redirect** - Post-authentication navigation

**API Layer:**
- **Token Verification** - Session validation middleware
- **Authentication Endpoints** - Login, logout, registration
- **Order Management** - Order fetching, acceptance, actions, updates
- **Real-time Data** - Status and dashboard information
- **History Management** - Order history and pagination
- **Profile Management** - User profile updates and account info
- **Location Services** - Place lookup and geographic data
- **Version Checking** - App version validation

**Assets Layer:**
- **Stylesheets** - Compiled CSS framework and custom styles
- **JavaScript Modules** - Framework libraries and application logic
- **Third-party Libraries** - Charting, socket client, image processing

**Service Worker Layer:**
- **Cache Management** - Asset caching with version control
- **Offline Support** - Fallback pages for offline scenarios
- **Network Strategy** - Cache-first or network-first based on resource type
- **Push Integration** - Background message handling
- **Version Updates** - Cache clearing on app updates
- **Notification Handling** - Click and wake-up processing

**Configuration Files:**
- **PWA Manifest** - Application metadata and display settings
- **Version Configuration** - Version tracking and update URLs
- **App Configuration** - Tracking settings and intervals
- **Offline Fallback** - Offline page template

**Key Architectural Patterns:**

**Location Tracking:**
- Background task with battery-aware throttling
- Distance threshold triggers immediate updates
- Battery and charging status included in data
- Network connectivity monitoring
- Throttling prevents excessive API calls
- Last location cached locally

**Push Notification Integration:**
- Token registration on app startup
- Automatic token refresh handling
- Background message processing
- Wake-up notifications for location updates
- In-app notification display
- Version update triggers cache clear

**Real-time Communication:**
- WebSocket connection for live updates
- Location broadcasting
- Order status synchronization
- Admin notification delivery
- Connection state management
- Automatic reconnection

**Version Management:**
- Version checking on app load
- Cache busting with version parameters
- Push-triggered cache clearing
- Service worker lifecycle management
- Update download links
- Force reload for critical updates

**Order Management:**
- Optimized fetching with caching headers
- Acceptance mechanism
- Delivery confirmation with location
- Fee and route updates
- Settlement processing
- Real-time synchronization
- Historical data with pagination

**UI/UX Patterns:**
- Pull-to-refresh functionality
- Modern visual effects (glass morphism, blur)
- Dark mode support
- Smooth animations and transitions
- Bottom navigation with active states
- Loading indicators
- Mobile-responsive design
- Data visualization with charts
- Image processing capabilities

**Authentication Flow:**
1. User opens mobile application
2. WebView loads authentication interface
3. User enters credentials
4. Client sends authentication request
5. Server validates against user database
6. Session token generated with expiry
7. Device information logged
8. Token returned and stored locally
9. Subsequent requests include authentication token
10. Server validates token on each request
11. Session and device verification performed

### 14.3 Pickup Portal (Hybrid Native + Web Application)

**Technology Stack:**
- Cross-platform mobile framework with WebView component
- Location SDK for GPS functionality
- Local storage for data persistence
- Safe area handling for mobile UI
- Lightweight JavaScript framework for reactive UI in WebView
- Utility-first CSS framework loaded via CDN
- Internationalization library for multi-language support
- Browser language detection
- Charting library for data visualization
- Chart plugin for data labels
- Mapping library for geographic visualization

**Application Architecture:**
The pickup portal uses a hybrid architecture similar to the rider portal but optimized for pickup/delivery operations:

**Native Layer:**
- **WebView Wrapper** - Container for web interface
- **Version Management** - Cache busting and update handling
- **Location Permissions** - Platform-specific permission handling
- **GPS Retrieval** - Location requests with timeout protection
- **Native-Web Bridge** - Bidirectional communication via postMessage
- **Hardware Integration** - Back button handling, phone calls, external links
- **Loading States** - Visual feedback during operations
- **Version Persistence** - Local storage for version tracking

**WebView Layer:**
- **Server-side Token Verification** - Authentication at request level
- **Security Headers** - Bot blocking and security policies
- **Reactive Components** - State management for UI
- **Internationalization Setup** - Multi-language support
- **Data Visualization** - Chart library initialization
- **Mapping Integration** - Interactive map setup
- **Modern UI Effects** - Glass morphism, dark mode, animations
- **Navigation System** - Tab-based navigation
- **Animation Effects** - Slide-up, fade-in transitions
- **Visual Polish** - Scrollbar hiding, hover effects

**Authentication Layer:**
- **Login Interface** - Authentication state management
- **Registration Interface** - Multi-step signup process
- **Form Validation** - Input validation and error handling
- **Visual Design** - Modern UI with animations
- **Progress Tracking** - Step indicators for multi-step flows
- **Location Selection** - Map integration for address selection
- **Password Management** - Visibility toggle and validation

**API Layer:**
- **Configuration** - Environment setup and helpers
- **Authentication Endpoints** - Login, signup, logout, token verification
- **Order Management** - Pickup order operations
- **Route Management** - Delivery route and way management
- **Profile Management** - User profile operations
- **Password Operations** - Password change functionality
- **Feedback System** - User feedback collection

**Localization Layer:**
- **Translation Files** - JSON-based translation dictionaries
- **UI Strings** - Common interface text
- **Navigation Labels** - Menu and button text
- **Domain Terminology** - Industry-specific translations
- **Validation Messages** - Error and success messages
- **Multi-language Support** - Dynamic language switching

**Service Worker Layer:**
- **Cache Management** - Versioned asset caching
- **Cache Strategy** - Cache-first for static resources
- **API Handling** - Skip caching for dynamic requests
- **CDN Caching** - External resource optimization
- **Offline Support** - Fallback pages for offline scenarios
- **Cache Cleanup** - Old cache removal on updates

**Configuration Files:**
- **PWA Manifest** - Application metadata and shortcuts
- **Version Configuration** - Version tracking and update URLs
- **Environment Configuration** - Database and service credentials
- **Version Alert** - Update notification logic

**Key Architectural Patterns:**

**Location Integration:**
- Platform-specific permission handling
- Timeout-protected location requests
- Native-web communication bridge
- JavaScript injection for location responses
- Error handling for permission denial
- Timeout protection against hanging requests

**WebView Communication:**
- Bidirectional messaging between layers
- Action-based communication (call, browse, locate)
- Phone integration via native linking
- External link handling
- WebView detection and compatibility
- Error handling for communication failures

**Internationalization:**
- Translation management system
- Automatic language detection
- Multi-language support (English, Myanmar)
- Dynamic language switching
- Localized UI strings
- Domain-specific terminology

**Mapping Integration:**
- Interactive map library integration
- Location selection in registration
- Address visualization
- Interactive map controls
- CSS and JS integration

**Data Visualization:**
- Chart library setup
- Data label plugins
- Dashboard statistics
- Trend visualization
- Status distribution charts

**UI/UX Patterns:**
- Modern visual effects (glass morphism, blur)
- Dark mode support
- Animation effects (fade-in, slide-in)
- Card hover effects
- Bottom navigation
- Progress indicators
- Loading overlays
- Mobile-responsive design
- Scrollbar hiding

**Security Features:**
- Server-side token verification
- Security headers and policies
- Bot detection and blocking
- SQL injection prevention
- Token-based authentication
- Session management

**Messaging Integration:**
- Bot notifications for orders
- Cancellation notifications
- Formatted message support
- Contact linking
- Assignment notifications
- Error handling for API failures

**Authentication Flow:**
1. User opens mobile application
2. WebView loads authentication interface
3. User chooses login or registration
4. For registration: Multi-step form with progress tracking
5. User enters credentials and contact information
6. Client sends authentication request
7. Server validates against database
8. Session token generated and stored
9. Token returned and stored in cookie
10. Subsequent requests verified at server level
11. Invalid tokens redirect to authentication
12. Token verified on each API call

**Build Process:**
- Server-side rendering for PHP files
- CDN-based resource loading (CSS, JS libraries)
- Framework libraries loaded via CDN
- Mapping library loaded via CDN
- Internationalization library loaded via CDN
- Native framework uses Expo (no build step)
- Service worker registered automatically

### 14.4 Marketplace Backend (Web Framework)

**Technology Stack:**
- Modern PHP framework
- Latest PHP version
- Relational database
- API authentication library
- Server-side templating engine
- Object-relational mapper

**Application Architecture:**
The marketplace backend follows a Model-View-Controller (MVC) pattern with RESTful API design:

**Routing Layer:**
- **Public Web Routes** - Customer-facing routes for browsing and searching
- **API Routes** - Admin-facing routes for content management
- **Route Model Binding** - Automatic model resolution from route parameters
- **Route Groups** - Organized route prefixes with middleware
- **Named Routes** - Referenceable route identifiers

**Controller Layer:**
- **Public Controllers** - Handle customer-facing requests (homepage, restaurants, menu, search, cart)
- **API Controllers** - Handle admin CRUD operations (businesses, categories, products, overall posts)
- **Request Validation** - Form request classes for input validation
- **Resource Transformation** - API resources for consistent JSON responses
- **Middleware Integration** - Authentication and permission checks

**Model Layer:**
- **Business Model** - Restaurant/business entities with relationships
- **Category Model** - Food categorization with hierarchy
- **Product Model** - Menu items with pricing and availability
- **Overall Model** - Featured posts/content with image relationships
- **OverallImage Model** - Image attachments for overall posts

**Middleware Layer:**
- **Authentication Middleware** - Validates session tokens from cookie or Bearer header
- **Session Verification** - Checks session table for valid, unexpired sessions
- **Role Verification** - Ensures user is not a rider account
- **Permission Middleware** - Role-based access control for operations
- **Permission Hierarchy** - Superadmin > admin > editor > viewer

**Database Layer:**
- **Marketplace Database** - Content management (businesses, categories, products, overall posts)
- **Office Database** - User authentication and session management
- **Operational Database** - Separate database for operational data
- **Migrations** - Database schema version control
- **Seeders** - Initial data population

**View Layer:**
- **Blade Templates** - Server-side rendered HTML templates
- **Component Organization** - Reusable template components
- **Layout Inheritance** - Consistent page structure
- **Dynamic Content** - Data binding from controllers

**Key Architectural Patterns:**

**Public Web Routes:**
- Homepage with featured content
- Restaurant listing and detail pages
- Menu navigation and category browsing
- Product detail pages
- Overall posts and business-specific content
- Category listings
- Search functionality across businesses, products, and categories
- Shopping cart operations
- Static informational pages
- SEO-friendly sitemap generation

**Marketplace API Routes:**
- Business CRUD operations
- Category CRUD operations
- Product CRUD operations
- Overall post CRUD operations with image management

**Authentication Flow:**
1. Admin accesses protected route
2. Middleware checks for authentication token
3. Token validated against session table
4. User role and permissions verified
5. Request proceeds to controller
6. Controller executes business logic
7. Response returned with appropriate status

**Permission System:**
- Role-based access control
- Permission-specific middleware
- Hierarchical permission structure
- Granular operation control (create, read, update, delete)

**Database Connections:**
- Separate databases for different concerns
- Connection management per database type
- Transaction support for data integrity
- Query optimization through eager loading

**Deployment Architecture:**
- Container-based deployment
- Web server for HTTP handling
- PHP processor for server-side logic
- Process manager for background tasks
- Database migrations for schema updates
- Asset optimization for performance
