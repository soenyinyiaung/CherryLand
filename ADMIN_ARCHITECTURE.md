# Cherryland Admin Panel - Architecture Documentation

## Project Overview

The Cherryland Admin Panel is a comprehensive Vue.js-based administration system for managing a delivery/rider service in Taunggyi, Myanmar. The system provides real-time tracking, order management, financial tracking, user management, and marketplace functionality.

**Tech Stack:**
- **Frontend:** Vue 3 (Composition API)
- **Styling:** Tailwind CSS
- **State Management:** Composables pattern
- **Real-time Communication:** Socket.IO
- **Client-side Storage:** IndexedDB (Dexie.js)
- **Backend:** PHP APIs
- **Routing:** Vue Router
- **Maps:** MapLibre GL JS
- **Date Handling:** Day.js

## Directory Structure

```
├── API layer
│   ├── Marketplace API calls
│   └── Orders/chat API calls
├── Static assets
├── Reusable Vue components
│   ├── Chat-related components
│   ├── Alert/confirmation modal
│   ├── Toast notifications
│   ├── Desktop header
│   ├── Desktop sidebar navigation
│   ├── Mobile header
│   ├── Mobile bottom navigation
│   └── Mobile overflow menu
├── Vue composables for reusable logic
│   ├── Authentication & permissions
│   ├── Alert & toast notifications
│   ├── Permission checks
│   ├── Chat data management
│   ├── Network status
│   ├── Chat scroll utilities
│   └── Socket client management
├── IndexedDB databases
│   ├── Chat messages & images
│   └── User data cache
├── Layout components
│   └── Main admin layout
├── Vue Router configuration
│   └── Route definitions & guards
├── Socket.IO client
│   └── Socket client implementation
├── Utility functions
│   ├── Date/time utilities
│   └── ...
├── Page components
│   ├── Login page
│   ├── Signup page
│   ├── Chat previews
│   ├── Main dashboard
│   ├── Real-time rider tracking
│   ├── Map visualization
│   ├── Settlement management
│   ├── Cash flow tracking
│   ├── User management
│   ├── Settings page
│   ├── Business management
│   ├── Category management
│   ├── Product management
│   ├── Overall posts
│   ├── Pickup orders
│   ├── COD management
│   ├── Expense tracking
│   ├── Shop management
│   ├── Morning duty
│   └── Notifications
├── Root component
├── Application entry point
├── Global styles
└── Service worker
```

## Page Functionalities

### Authentication Pages

#### Login
- **Purpose:** User authentication
- **Features:**
  - Username/password login
  - Token-based authentication (stored in cookie)
  - Auto-redirect to dashboard on success
  - Error handling for failed login

#### Signup
- **Purpose:** User registration
- **Features:**
  - Username, name, phone, password fields
  - Role selection (rider)
  - Commission type selection (thirty)
  - Password visibility toggle
  - Auto-redirect to login on success

### Main Dashboard

#### Dashboard
- **Purpose:** Main system overview and rider performance tracking
- **Features:**
  - **Advanced Filtering System:**
    - Rider status filter (all riders, present riders, absent riders, low ways <5, completed accounts, incomplete accounts)
    - Date range filtering with from/to date inputs
    - Individual person selection dropdown
    - Single date filter for quick daily view
  - **Statistics Cards:**
    - Dynamic grid layout based on screen size
    - Color-coded gradient cards for different metrics
    - Auto-hides cards with zero values
    - Displays total orders, ways, delivery fees, penalties, etc.
  - **Net Status Banner:**
    - Shows profit/loss/neutral status
    - Color-coded (green for profit, orange for loss)
    - Only displays when no specific user is selected
  - **Charts Visualization:**
    - Revenue chart showing trends over time
    - Distribution chart for order breakdown
    - Uses Chart.js library
    - Only visible when viewing all riders
  - **Detailed Rider Table:**
    - Sortable columns (ID, rider, orders, ways, delivery fee, penalty)
    - 30/70 split display (rider 70%, office 30%)
    - Export CSV functionality (all riders or 15-day total)
    - Pagination support
  - **Individual Rider View:**
    - Orders table with inline editing (double-click to edit)
    - Editable fields: username, pickup location, delivery location, delivery fee, way count, timestamp
    - Delete order functionality with confirmation
    - Deposits table showing stake/deposit/withdraw transactions
    - Settlements table with message links to Telegram
    - Toggle settlement status (received/pending)
  - **Billing Calculation Section:**
    - Automatic calculation of total delivery fee
    - 70/30 split breakdown (rider vs office)
    - Balance calculation from deposits/withdrawals
    - Penalty deduction
    - Maintenance fee addition
    - Final settlement amount
    - Copy billing summary as image functionality
  - **Account Submission Form:**
    - Cash in input
    - KPay in input
    - Cashback out input
    - KPayback out input
    - Real-time balance calculation
    - Submit account button

### Real-time Features

#### Realtime
- **Purpose:** Real-time rider status tracking and monitoring
- **Features:**
  - **Rider Status Filtering:**
    - Filter tabs: All, Ready (green), On Order (red), Meal (pink), Busy (amber), Offline (gray)
    - Real-time status badge showing current activity
    - Dynamic badge icons based on status (truck for orders, utensils for meal, wrench for busy, check for ready)
    - Ways count display for riders with active orders
  - **Rider Card Display:**
    - Profile picture with online/offline indicator dot
    - Username and name display
    - Status badge with gradient colors
    - Nearest place/location display with spinning location icon
    - Expandable details section on click
  - **Expanded Rider Details:**
    - Thermal Box status indicator (B tag - blue when active)
    - Home Way status indicator (H tag - red when active)
    - Battery level display with charging animation
    - Last seen time (shows "Active" when online, time ago when offline)
  - **Real-time Updates via Socket.IO:**
    - Location updates with automatic nearest place fetching
    - Rider offline detection
    - Status changes (activity, thermal box, home way)
    - Order assignment/removal updates
    - Admin row updates for order management
    - Full UI sync on demand
  - **Location Services:**
    - Batch nearest place fetching for multiple riders
    - Client-side caching for place names
    - Automatic retry on location fetch failure
  - **User Profile Integration:**
    - IndexedDB user profile caching
    - Avatar display with fallback to generated avatars
    - Profile image error handling
  - **Socket Reconnection Handling:**
    - Automatic data refresh on socket reconnect
    - User profile reload on reconnection
    - Data-updated event emission

#### Map
- **Purpose:** Visual map of rider locations in real-time
- **Features:**
  - **Map Display:**
    - MapLibre GL JS map rendering
    - Google Maps tiles as base layer
    - Full-screen map view
    - Responsive design for mobile and desktop
  - **Rider Markers:**
    - Real-time rider position markers
    - Color-coded markers based on status (green for ready, red for on-order, etc.)
    - Marker labels with rider username
    - Click to focus on rider
  - **Rider Search:**
    - Search bar to find riders by username
    - Auto-complete suggestions
    - Jump to rider on map
  - **Sidebar:**
    - Rider list with status indicators
    - Online/offline status
    - Current activity display
    - Click to focus on map
  - **Focused Rider Card:**
    - Selected rider details
    - Profile picture
    - Status information
    - Battery level
    - Last seen time
    - Close button to unfocus
  - **Real-time Updates:**
    - Socket.IO integration for live location updates
    - Automatic marker position updates
    - Status change updates

#### Chat
- **Purpose:** Chat system for order communication and coordination
- **Features:**
  - **Chat Types:**
    - Order Chat - for order-related discussions
    - စိုက်ငွေ Chat - for deposit-related discussions
    - ရုံးရှင်း Chat - for settlement-related discussions
  - **Chat Preview Cards:**
    - Chat type display with icon
    - Last message preview
    - Unread count badge
    - Click to open chat detail view
  - **Message Storage:**
    - IndexedDB (chatDB) for local message storage
    - Offline message support
    - Message persistence across sessions
  - **Synchronization:**
    - MongoDB incremental sync for new messages
    - Sync metadata tracking
    - Last sync timestamp
  - **Real-time Messaging:**
    - Socket.IO integration for live updates
    - New message notifications
    - Message edit/delete updates
    - Reaction updates
    - Read status updates
  - **User Integration:**
    - User profile caching via IndexedDB (usersDB)
    - Avatar display
    - Username display

### Financial Management

#### Settlement
- **Purpose:** Financial settlement management and tracking
- **Features:**
  - **Filtering System:**
    - Person/rider selection dropdown with name and username display
    - Single date filter for daily view
    - Date range toggle switch
    - From/to date inputs when date range is enabled
    - Refresh data button
  - **Statistics Cards:**
    - Total settlement amount (blue gradient card)
    - Received amount (white card with green text)
    - Pending amount (white card with red text)
  - **Settlements Table:**
    - Source column with Telegram message link (opens in new tab)
    - Users column showing rider username and submitter
    - Amount details column with payer breakdown
    - Multiple payers supported with individual amounts
    - Total calculation for multi-payer settlements
    - DateTime column with date and time display
    - Action column with toggle button
  - **Settlement Status Toggle:**
    - Toggle between "ရပြီး" (received) and "မရသေး" (pending)
    - Color-coded buttons (green for received, red for pending)
    - Instant status update without page reload
    - Automatic total recalculation on status change
  - **Loading State:**
    - Loading spinner with text during data fetch
    - Empty state message when no settlements found
  - **Data Management:**
    - Local state updates for immediate UI feedback
    - Optimistic UI updates
    - Error handling with console logging

#### Cashflow
- **Purpose:** Cash flow tracking and transaction management
- **Features:**
  - **Filtering System:**
    - Type filter (All Types, KPay, Cash)
    - Action filter (All Actions, Stake/စိုက်ငွေ, Deposit/အပ်ငွေ)
    - User filter with rider and office staff separation
    - Date filter for specific day transactions
  - **Transaction Table:**
    - Timestamp column with date and time display
    - Type/Action column with color-coded badges (blue for KPay, amber for Cash)
    - Action indicator (green for deposit, amber for stake)
    - Amount display with Ks suffix
    - From/User column showing sender and username
    - Note column with truncation for long notes
    - Manage column with edit and delete buttons
  - **Edit Modal:**
    - Type and action input fields
    - Amount input with green emphasis
    - From person and username inputs
    - Note textarea for additional details
    - Save and cancel buttons
  - **Delete Functionality:**
    - Confirmation alert before deletion
    - Success toast notification after deletion
    - Error handling with user feedback
  - **Empty State:**
    - Icon display when no records found
    - Helpful message to adjust filters

#### COD
- **Purpose:** Cash on delivery status management for shops
- **Features:**
  - **Filtering System:**
    - Shop selection dropdown
    - Collected status filter (all, collected, pending)
    - Date range filtering with from/to inputs
  - **Statistics Cards:**
    - Transferred amount (green)
    - Pending amount (red)
    - Total amount (blue)
  - **Shop-wise Breakdown:**
    - Individual shop COD amounts
    - Transfer status per shop
    - Toggle collected/pending status per shop
  - **Transfer Status Toggle:**
    - Instant status update
    - Color-coded indicators
    - Automatic statistics recalculation

#### Expense
- **Purpose:** Expense and promotion tracking
- **Features:**
  - **Filtering System:**
    - Date range filtering with from/to inputs
    - Type filter (expense, promotion)
    - Search functionality for notes
  - **Summary Display:**
    - Total amount calculation
    - Color-coded based on type
  - **Expense Table:**
    - Timestamp display with date and time
    - Note column for expense description
    - Amount display
    - Type indicator
    - Edit and delete actions
  - **CRUD Operations:**
    - Add new expense via modal
    - Edit existing expense
    - Delete with confirmation
    - Form fields: timestamp, note, amount, type

### User Management

#### Users
- **Purpose:** User account management and administration
- **Features:**
  - **User List Display:**
    - User avatar with initial letter
    - Name and username display
    - Role dropdown with inline update
    - Phone number display
    - Telegram username and chat ID
    - Type indicator (total/thirty)
    - Account status (healthy/suspended/blocked)
    - Registration date
    - Status toggle (active/deactivated)
  - **Filtering System:**
    - Search by name, phone, or username
    - Role filter (all, rider, office, admin, ceo, developer)
    - Status filter (active only, all, deactivated)
  - **User Modal (Add/Edit):**
    - Username input (required)
    - Full name input (required)
    - Phone input (required)
    - Telegram username input
    - Telegram chat ID input
    - Role dropdown (rider, office, admin, ceo, developer)
    - Type dropdown (total, thirty)
    - Account status dropdown (healthy, suspended, blocked)
    - Password field with visibility toggle (for new users only)
    - Password generator button (12 characters with special characters)
  - **User Actions:**
    - Add new user button
    - Edit user button (opens modal with pre-filled data)
    - Delete user button with confirmation alert
    - Inline role update via dropdown
    - Inline status toggle via switch
  - **Notifications:**
    - Success toast on user creation/update/deletion
    - Error toast on failure
    - Confirmation alert for deletion

#### Merchants
- **Purpose:** Shop/merchant account management
- **Features:**
  - **Merchant List Display:**
    - Shop name and owner name
    - Phone number
    - KPay number
    - Status indicator (active, pending, suspended)
    - Location coordinates display
  - **Filtering System:**
    - Search by shop name or owner
    - Status filter
  - **Merchant Modal (Add/Edit):**
    - Shop name input
    - Owner name input
    - Phone number input
    - KPay number input
    - Password field with generate button
    - Status dropdown (active, pending, suspended)
    - Address input
    - Interactive map for location selection
    - Latitude and longitude inputs (auto-filled from map)
  - **Map Integration:**
    - MapLibre GL JS map display
    - Click to set location
    - Current location button (geolocation)
    - Marker display for selected location
  - **Password Management:**
    - Generate random password button
    - Copy password to clipboard

### Order Management

#### PickupWays
- **Purpose:** Pickup delivery order management
- **Features:**
  - **Statistics Cards:**
    - Total ways count
    - Pending ways count
    - Delivered ways count
    - Canceled ways count
  - **Filtering System:**
    - Search by order ID or details
    - Status filter (all, pending, delivered, canceled)
    - Date range filtering with from/to inputs
  - **Order Grouping:**
    - Orders grouped by shop
    - Orders grouped by rider within each shop
    - Collapsible shop sections
  - **Order Management:**
    - Inline add new order button per shop/rider
    - Inline edit existing orders
    - Inline delete orders
    - Order status toggle (pending/delivered/canceled)
    - Delivery fee deduction option
  - **Order Details:**
    - Way ID with Telegram link
    - Pickup location
    - Delivery location
    - Delivery fee
    - Way count
    - Timestamp
  - **Rider Assignment:**
    - Rider selection dropdown
    - Filter riders by status

### Marketplace Module

#### MarketplaceBusinesses
- **Purpose:** Business management for marketplace
- **Features:**
  - **Business List Display:**
    - Business name and description
    - Type indicator
    - Status indicator (active, inactive)
    - Featured badge
    - Logo and cover image thumbnails
    - Location coordinates
  - **Filtering System:**
    - Search by business name
    - Type filter
    - Status filter (active, inactive)
    - Featured filter
  - **Business Modal (Add/Edit):**
    - Business name input
    - Description textarea
    - Type dropdown
    - Status dropdown (active, inactive)
    - Featured toggle
    - Logo image upload
    - Cover image upload
    - Latitude and longitude inputs
  - **Image Upload:**
    - External image service integration
    - Logo upload with preview
    - Cover image upload with preview
    - URL input for external images
  - **Pagination:**
    - Page navigation
    - Items per page control

#### MarketplaceCategories
- **Purpose:** Category management for marketplace
- **Features:**
  - **Category List Display:**
    - Category name
    - Type indicator
    - Status indicator (active, inactive)
    - Sort order number
    - Image thumbnail
  - **Filtering System:**
    - Search by category name
    - Type filter
    - Status filter (active, inactive)
  - **Category Modal (Add/Edit):**
    - Category name input
    - Type dropdown
    - Status dropdown (active, inactive)
    - Sort order input
    - Image upload
  - **Image Upload:**
    - External image service integration
    - Image upload with preview
    - URL input for external images
  - **Pagination:**
    - Page navigation
    - Items per page control

#### MarketplaceProducts
- **Purpose:** Product management for marketplace
- **Features:**
  - **Product List Display:**
    - Product name and description
    - Business name
    - Category name
    - Price and sale price
    - Product type indicator
    - Availability status
    - Featured badge
    - Image thumbnail
  - **Filtering System:**
    - Search by product name
    - Business dropdown (loaded from API)
    - Category dropdown (loaded from API)
    - Type filter
    - Availability filter
  - **Product Modal (Add/Edit):**
    - Product name input
    - Description textarea
    - Business dropdown
    - Category dropdown
    - Price input
    - Sale price input
    - Type dropdown
    - Availability toggle
    - Featured toggle
    - Image upload
  - **Image Upload:**
    - External image service integration
    - Image upload with preview
    - URL input for external images
  - **Pagination:**
    - Page navigation
    - Items per page control

#### Overall
- **Purpose:** Overall posts management for marketplace announcements
- **Features:**
  - **Posts List Display:**
    - Post title and description
    - Business name
    - Category name
    - Featured badge
    - Image thumbnails (multiple)
    - Created date
  - **Filtering System:**
    - Search by post title
    - Business dropdown (loaded from API)
    - Featured filter
  - **Post Modal (Add/Edit):**
    - Title input
    - Description textarea
    - Business dropdown
    - Category dropdown
    - Featured toggle
    - Multiple image upload
    - URL input for external images
  - **Image Upload:**
    - External image service integration
    - Multiple image upload with preview
    - URL input for external images
    - Image removal functionality

### Other Features

#### MorningDuty
- **Purpose:** Morning duty assignment for riders
- **Features:**
  - **Duty List Display:**
    - Date display
    - Assigned riders list
    - Rider count
    - Created timestamp
  - **Filtering System:**
    - Date range filtering with from/to inputs
  - **Duty Modal (Add/Edit):**
    - Date input
    - Rider selection (multiple riders)
    - Rider list loaded from API
    - Search/filter riders
  - **CRUD Operations:**
    - Add new duty assignment
    - Edit existing duty
    - Delete duty with confirmation

#### Notifications
- **Purpose:** Send notifications to riders
- **Features:**
  - **Rider Selection:**
    - Select all riders option
    - Individual rider selection from list
    - Rider list loaded from API (active riders only)
    - Checkbox selection interface
  - **Notification Types:**
    - Regular notification with message and title
    - Wake-up only mode (silent push notification)
  - **Notification Form:**
    - Title input field
    - Message textarea
    - Wake-up only toggle switch
  - **Notification Sending:**
    - Socket.IO integration for real-time delivery
    - Send button with loading state
    - Success/error feedback

#### Settings
- **Purpose:** Application settings and user profile management
- **Features:**
  - **Profile Section:**
    - Profile picture display with avatar
    - Upload new profile picture
    - External image service integration for uploads
    - Image preview after upload
  - **User Information Display:**
    - Username display
    - Name display
    - Role display
    - Phone number display
  - **Password Change:**
    - Current password input
    - New password input
    - Confirm password input
    - Password visibility toggle
    - Update password button
  - **Application Information:**
    - App version display
    - Build information
  - **Cache Management:**
    - Clear application cache button
    - Service worker unregister
    - Cache storage clearing
  - **Account Actions:**
    - Logout button with confirmation

## Key Composables

### useAuth
- **Purpose:** Authentication and authorization
- **Features:**
  - Token management (cookie-based)
  - User data storage (localStorage)
  - Role hierarchy (office < admin < ceo < developer)
  - Permission checking
  - Route-based access control
- **Roles:** office, admin, ceo, developer

### useAlert
- **Purpose:** Alert and toast notifications
- **Features:**
  - Alert modal for confirmations
  - Toast notifications (success, error, warning, info)
  - Auto-dismiss with configurable duration

### usePermissions
- **Purpose:** Message permission checks
- **Features:**
  - Edit message permissions
  - Delete message permissions
  - Reaction permissions
  - Reply permissions

## Database Schema

### chatDB (IndexedDB)
- **messages:** Chat messages with indexes for chatType, tempId, serverId, senderId, timestamp, status, messageType, isCancelled
- **images:** Binary image data with messageId reference
- **syncMetadata:** Sync timestamps for incremental sync
- **pinnedMessages:** Pinned messages (max 5 per user)

### usersDB (IndexedDB)
- **users:** User data with indexes for userId, username, name, avatar, role, status

## Socket.IO Integration

### Socket Events
- **Connection:** connect, disconnect, connect_error
- **Chat:** new-message, message-edited, message-deleted, reaction-updated, message-read
- **Acknowledgments:** message-acknowledged, delete-acknowledged
- **Orders:** order-cancelled
- **Rooms:** join-chat-room

### Features
- Automatic reconnection
- Message queuing when offline
- Pending message processing on reconnect
- Cross-tab sync via BroadcastChannel
- Image storage in IndexedDB

## Routing

### Route Structure
- Public routes: /login, /signup
- Protected routes: All others (require authentication)
- Permission-based access control
- Auto-redirect to first permitted route

### Navigation
- Desktop: Sidebar navigation
- Mobile: Bottom navigation with swiper
- Special handling for map and chat_order pages

## API Integration

### Backend APIs
- Authentication endpoints
- Orders and chat endpoints
- User management endpoints
- Settlement and financial tracking endpoints
- Marketplace REST API endpoints
- Image upload service

## Responsive Design

### Mobile Features
- Swiper-based page navigation
- Bottom navigation bar
- Mobile-specific header
- Touch-optimized map controls
- More sheet for overflow menu

### Desktop Features
- Sidebar navigation (collapsible)
- Desktop header
- Full-width map and chat

## Security

### Authentication
- Cookie-based token storage
- 150-day token expiration
- Periodic token validation (1 hour)
- Auto-logout on auth failure

### Permissions
- Role-based access control
- Route-level permission checks
- Component-level permission checks
- Action-specific permissions

## Performance Optimizations

### Code Splitting
- Lazy-loaded route components
- Async component loading

### Data Management
- IndexedDB for offline data
- Incremental sync with MongoDB
- Message queuing for offline scenarios
- Virtualized swiper slides

### Socket Optimization
- Message queuing when disconnected
- Batch processing of pending messages
- Cross-tab sync to avoid duplicate API calls

## Deployment

### Environment Variables
- VITE_SOCKET_URL: Socket server URL
- VITE_IMAGE_SERVER_URL: Image service URL

### Service Worker
- Offline support
- Cache management
- Version update detection

## Key Features Summary

1. **Real-time Tracking:** Live rider status and location tracking via Socket.IO
2. **Order Management:** Comprehensive order management with filtering and editing
3. **Financial Tracking:** Settlements, cashflow, COD, and expense tracking
4. **User Management:** Role-based user management with permissions
5. **Marketplace:** Full marketplace module for businesses, categories, and products
6. **Chat System:** Multi-type chat with real-time messaging and offline support
7. **Map Visualization:** Interactive map with rider locations
8. **Mobile-First:** Responsive design with mobile-specific navigation
9. **Offline Support:** IndexedDB storage and service worker
10. **Permission System:** Role-based access control with granular permissions
