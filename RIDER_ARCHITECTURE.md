# Cherryland Rider Frontend Architecture

## Overview

The Cherryland Rider Frontend is a Progressive Web App (PWA) designed for delivery riders to manage orders, track earnings, and communicate with the office in real-time. It is built with PHP, Alpine.js, Tailwind CSS, and integrates with Firebase Cloud Messaging and Socket.IO for real-time features.

## Technology Stack

### Frontend Framework
- **Alpine.js** - Lightweight JavaScript framework for reactive UI components
- **Tailwind CSS** - Utility-first CSS framework for styling
- **Chart.js** - Data visualization for dashboard charts
- **Socket.IO Client** - Real-time bidirectional communication
- **Firebase Cloud Messaging** - Push notifications
- **Font Awesome** - Icon library

### Backend
- **PHP** - Server-side logic and API endpoints
- **MySQL** - Database for orders, users, settlements, and other data

### PWA Features
- **Service Worker** - Offline caching and background sync
- **Web App Manifest** - Installable as a mobile app
- **Firebase Messaging** - Background push notifications

## Directory Structure

```
├── API endpoints
│   ├── API version 2 endpoints
│   │   ├── Authentication helper functions
│   │   ├── User login endpoint
│   │   ├── User registration endpoint
│   │   ├── User logout endpoint
│   │   ├── Dashboard statistics and charts
│   │   ├── Fetch active orders for rider
│   │   ├── Fetch order history
│   │   ├── Fetch account status
│   │   ├── Real-time rider data
│   │   ├── Fetch pickup ways
│   │   ├── Fetch contact list
│   │   ├── Accept/reject orders
│   │   ├── Order status updates
│   │   ├── Update order details
│   │   ├── Update rider profile
│   │   ├── Check COD collection status
│   │   ├── App version check
│   │   └── Get nearest place from location
│   └── API version 3 endpoints (similar structure)
├── Assets
│   ├── Stylesheets
│   │   ├── Compiled Tailwind CSS
│   │   ├── Tailwind source
│   │   ├── Main page styles
│   │   ├── Home page styles
│   │   ├── History page styles
│   │   └── Other page-specific styles
│   ├── JavaScript
│   │   └── Version 2 scripts
│   │       ├── Home page logic
│   │       ├── History page logic
│   │       ├── Agreement/terms handling
│   │       ├── Version update alerts
│   │       ├── Alpine.js framework
│   │       ├── Socket.IO client
│   │       ├── Chart.js library
│   │       ├── Chart.js plugin
│   │       └── Image cropping library
│   ├── User profile images
│   └── Settlement voucher images
├── Configuration
│   ├── Database configuration
│   ├── Environment variables
│   └── Environment loader
├── Main application page
├── Login/Signup page
├── Account verification page
├── Offline fallback page
├── PWA service worker
├── PWA manifest
├── Node.js dependencies
├── Tailwind configuration
├── PostCSS configuration
├── App configuration (version, force reload)
├── Version information
└── Environment variables
```

## Key Pages and Features

### Authentication

#### Login/Signup Page
- **Purpose:** User authentication and new rider registration
- **Features:**
  - **Login Form:**
    - Username or phone number input
    - Password input
    - Remember me functionality
    - Error display for failed attempts
  - **Signup Form:**
    - Full name input
    - Phone number input
    - Password input
    - Confirm password input
    - Password validation
  - **UI Features:**
    - Animated mesh gradient background
    - Glass morphism card design
    - Loading overlay during authentication
    - Auto-redirect to main app on success
  - **Session Management:**
    - Token-based authentication
    - User data stored in localStorage
    - React Native WebView integration for mobile app
  - **Security:**
    - Bot detection and blocking
    - Token validation via authentication endpoint
    - Secure password handling

### Main Application

#### Home Page
- **Purpose:** Main rider interface for order management
- **Features:**
  - **Order Display:**
    - Active orders list with parsed details
    - Order summary cards with gradient backgrounds
    - Photo gallery for order photos with aspect ratio detection
    - Portrait photos displayed at half width
    - Landscape photos displayed at full width
    - Edit indicator badge for modified orders
    - Click to view full image in modal
    - Lazy loading for images
  - **Order Text Highlighting:**
    - Automatic syntax highlighting for order text
    - Phone numbers highlighted with click-to-call functionality
    - Delivery fee highlighted in green
    - Way count highlighted in blue
    - Delivery address section highlighted in violet
    - Pickup locations color-coded by type (food=blue, shopping=rose, passenger=purple, gate=cyan)
    - Items with quantities highlighted
    - Action keywords highlighted (ကနေ, ဝယ်ပေးပါ, ပို့ပေးပါ, etc.)
    - Manual notes highlighted in red
    - Payment alerts highlighted with animation
    - Separator lines styled as dashed lines
  - **Waiting States:**
    - Account restricted warning with red icon (disable status)
    - Account warning state with amber icon
    - Normal waiting state with animated motorcycle icon
    - Status-specific messages in Burmese
  - **Real-time Updates:**
    - Socket.IO integration for live order updates
    - Auto-refresh order list on new orders
    - Order removal notifications
    - Order edit notifications
    - Photo update notifications
    - Web notifications for non-React Native browsers
    - Vibration feedback for React Native app
    - Duplicate notification prevention when FCM is active
  - **Location Services:**
    - GPS location fetching with high accuracy
    - React Native WebView location support
    - Fallback to last known location
    - Timeout handling (10 seconds)
    - Maximum age for cached location (30 seconds)
  - **Notification System:**
    - Custom notification function with service worker
    - Permission request on first use
    - Notification options (title, body, icon, badge)
    - Fallback for browsers without service worker
  - **Socket Connection Management:**
    - Auto-reconnection handling
    - Data reload on reconnect
    - Pickup ways reload on reconnect
    - Retry mechanism for native app (5 retries)
    - Connection status tracking

#### Dashboard
- **Purpose:** Rider performance statistics and earnings overview
- **Features:**
  - **Statistics Summary Cards:**
    - Total ways (deliveries) count with gradient background
    - Total income (100% of delivery fees) display
    - Income share (30% or total based on user type) calculation
    - Total stake (deposits made) tracking
    - Total deposit (withdrawals) tracking
    - Balance calculation (stake - deposit)
    - Penalty amount display
    - Maintenance fee (fixed at 500)
    - Total settled amount from settlements
    - Final summary (net earnings) calculation
  - **Charts Visualization:**
    - Hourly ways distribution bar chart
    - Hourly income distribution bar chart
    - Only active hours displayed (hours with data)
    - Chart.js with data labels plugin
    - Responsive chart sizing
    - Color-coded bars for ways and income
  - **Date Filtering:**
    - From date picker input
    - To date picker input
    - Default to current day
    - Date format validation
    - Auto-refresh on date change
  - **Detailed Breakdown Sections:**
    - Orders list with timestamps and IDs
    - Deposits list with types (stake/deposit) and actions
    - Settlements list with payer details and amounts
    - Expandable/collapsible sections
  - **User Type Support:**
    - Total type (100% income share calculation)
    - Thirty type (30% income share calculation)
    - Dynamic income share based on user type

#### History Page
- **Purpose:** View and manage past orders
- **Features:**
  - **Order List Display:**
    - Order ID with clickable link
    - Delivery address with truncation for long addresses
    - Delivery fee with inline edit capability
    - Way count with inline edit capability
    - Timestamp with date and time
    - Settlement status indicator
    - Voucher images thumbnail display
    - Loading spinner during data fetch
  - **Search Functionality:**
    - Search by order ID (case-insensitive)
    - Search by delivery address
    - Real-time filtering as user types
    - Clear search button
  - **Advanced Filtering:**
    - Date range filter (from/to date inputs)
    - Fee range filter (minimum/maximum inputs)
    - Toggle button to show/hide advanced filter panel
    - Live fetch on filter change
  - **Inline Editing:**
    - Click to edit delivery fee
    - Input field appears on click
    - Auto-save on blur (focus lost)
    - Local state update for immediate UI feedback
    - API call to update database
    - Error handling for failed updates
  - **Settlement Modal:**
    - Modal with glass morphism design
    - Upload voucher images (multiple file support)
    - Add settlement notes in textarea
    - Parse settlement amounts from text automatically
    - Multiple image upload with drag-and-drop
    - Image preview grid (3 columns)
    - Delete existing images with confirmation
    - Image security validation (MIME type check)
    - Safe filename generation
  - **Office Settlement Text Parsing:**
    - Automatic text parsing for payer amounts
    - Burmese numeral to English conversion
    - Automatic amount extraction from lines
    - Skip lines starting with "total" or "စုစုပေါင်း"
    - Minimum amount threshold (100)
    - Payer name and amount extraction
  - **Dashboard Mode:**
    - Simplified view for dashboard integration
    - No advanced filter panel
    - Render as dashboard history list

### Real-time Features

#### Real-time Updates
- **Purpose:** Live rider status and order updates
- **Features:**
  - **Socket.IO Integration:**
    - Automatic connection establishment
    - Connection management with retry logic
    - Event listeners for order updates
    - Reconnection handling with data reload
    - Connection status tracking
  - **Location Tracking:**
    - Periodic location broadcasting
    - Wake-up notification handling for location updates
    - Battery level reporting
    - Thermal box status (on/off indicator)
    - Home way status (on/off indicator)
    - Last seen timestamp
  - **Order Assignment:**
    - New order notifications via socket
    - Order removal notifications
    - Order edit notifications
    - Order photo update notifications
    - Order acceptance/rejection capability
    - Order status updates (pending, delivered, canceled)
  - **Admin Communication:**
    - Receive admin messages
    - Status change requests
    - Wake-up only silent notifications
  - **Notification Handling:**
    - Web notifications for browsers
    - Vibration feedback for React Native app
    - Duplicate prevention when FCM is active
    - Custom notification titles based on action type

### Profile Management

#### Profile Update
- **Purpose:** Manage rider profile information
- **Features:**
  - **Profile Picture:**
    - Upload new profile image from device
    - Image cropping with cropper.js library
    - Aspect ratio preservation
    - Preview before upload
    - Circular crop mask
    - Zoom and pan controls
    - Reset crop button
  - **Personal Information:**
    - Full name input field
    - Phone number input field
    - Real-time validation
    - Save button with loading state
  - **Password Change:**
    - Current password input for verification
    - New password input
    - Confirm password input
    - Password strength indicator
    - Password visibility toggle
    - Error message for mismatched passwords
  - **Account Status Display:**
    - View account status (healthy, warning, disable)
    - Color-coded status indicators
    - View account type (total, thirty)
    - Registration date display
  - **Profile Information Display:**
    - Username display
    - Role display
    - Phone number display
    - Profile image display with fallback

### Contact Management

#### Contacts Page
- **Purpose:** View and manage contact list
- **Features:**
  - **Contact List Display:**
    - Contact names with avatars
    - Phone numbers with formatting
    - Contact type indicators (shop, customer, etc.)
    - Alphabetical sorting
    - Grouped by first letter
  - **Search Functionality:**
    - Search by contact name
    - Search by phone number
    - Real-time filtering
    - Clear search button
  - **Quick Actions:**
    - Call contact directly via phone app
    - React Native WebView call integration
    - Web tel: protocol fallback
    - One-tap dialing
  - **Contact Management:**
    - Add new contact
    - Edit existing contact
    - Delete contact with confirmation
    - Import contacts from phone

## Key JavaScript Modules

### GPS Module
- **Purpose:** Get rider location for tracking
- **Features:**
  - Browser geolocation API integration
  - React Native WebView location support
  - PostMessage communication with native app
  - Fallback to last known location
  - Timeout handling (5 seconds for native, 10 seconds for web)
  - High accuracy mode enabled
  - Maximum age for cached location (30 seconds)
  - Error handling for location failures
  - Coordinate format (latitude,longitude)

### Notification Module
- **Purpose:** Display notifications to rider
- **Features:**
  - Service worker integration for background notifications
  - Permission request on first use
  - Custom notification options (title, body, icon, badge)
  - Vibration support for mobile devices
  - Fallback for browsers without service worker
  - Notification click handling
  - Duplicate notification prevention
  - Notification queue management

### History Module
- **Purpose:** Manage order history and settlements
- **Features:**
  - Data fetching with Bearer token authentication
  - Real-time search and filter
  - Inline editing for delivery fee and way count
  - Settlement modal with image upload
  - Local state management for immediate UI updates
  - Auto-save on blur
  - Error handling with user feedback
  - Dashboard mode integration
  - Advanced filtering with date and fee ranges

## API Integration

### Authentication
- **Token-based authentication** using Bearer tokens
- **Session management** via database sessions table
- **Token expiration** handling
- **Bot detection** and blocking

### Order Management
- **Fetch active orders** with parsed data and photos
- **Accept/reject orders** via API endpoints
- **Update order status** (pending, delivered, canceled)
- **Update delivery fee** inline with auto-save
- **Update way count** inline with auto-save
- **Upload order photos** via API with validation
- **Delete order photos** via API
- **View order details** with parsed information

### Settlement
- **Submit settlement** with text parsing for amounts
- **Upload voucher images** with security validation
- **Parse settlement amounts** from text automatically
- **View settlement history** with payer details
- **Delete existing settlement images**
- **Update settlement notes**
- **Burmese numeral conversion** for amounts

### Dashboard
- **Fetch statistics** for date range
- **Get hourly data** for charts visualization
- **Calculate earnings** based on user type (total/thirty)
- **Fetch deposits** and withdrawals records
- **Fetch settlements** with payer breakdown
- **Calculate final summary** with deductions

### Real-time
- **Socket.IO connection** for live updates
- **Location broadcasting** with GPS data
- **Status updates** (ready, on-order, meal, busy, offline)
- **Order notifications** via socket
- **Photo update notifications**
- **Wake-up notifications** for location updates

## PWA Features

### Service Worker
- **Caching Strategy:**
  - Cache static assets (CSS, JS, images, fonts)
  - Network-first strategy for API requests (no caching)
  - Cache-first strategy for static resources
  - Offline fallback page for navigation requests
  - Cache version management (v30)
  - Automatic cache cleanup on update
- **Background Sync:**
  - Firebase Messaging integration for push notifications
  - Wake-up notification handling for location updates
  - Version update detection via FCM
  - Automatic cache clearing on version update
  - Service worker self-unregistration on update
  - Client message broadcasting for version updates
- **Push Notifications:**
  - Firebase Cloud Messaging integration
  - Background message handling
  - Notification click handling with app focus
  - Wake-up only mode (silent notifications)
  - Custom notification options (vibrate, requireInteraction)
  - Notification data payload handling

### Manifest
- **App Name:** Cherry Land Rider
- **Display Mode:** Standalone
- **Theme Color:** #4f46e5
- **Start URL:** Rider app path
- **Icons:** 512x512 PNG

## Security Features

### Authentication
- **Bearer token** validation
- **Session expiration** checking
- **Token storage** in localStorage
- **React Native WebView** session sync

### Bot Protection
- **User agent analysis**
- **Cloudflare headers** support
- **Device type detection**
- **Comprehensive bot pattern** matching
- **IP address** validation

### Image Upload Security
- **MIME type validation**
- **File extension** enforcement
- **Image property** verification
- **Safe filename** generation
- **Upload directory** protection

## Database Integration

### Tables Used
- **Users table** - Rider account information
- **Sessions table** - Session management
- **Orders table** - Order records
- **Raw orders table** - Raw order messages
- **Order photos table** - Order photo references
- **Settlements table** - Settlement records
- **Settlement images table** - Settlement voucher images
- **Deposits table** - Deposit/withdrawal records
- **Penalties table** - Penalty records

## Mobile App Integration

### React Native WebView
- **Session sharing** via postMessage
- **Location fetching** via native API
- **Phone calls** via native API
- **Notification handling** bypass

## Version Management

### App Configuration
- **Web version** tracking
- **Force reload** flag
- **Last updated** timestamp
- **Version check** via API endpoint
- **Cache clearing** on update

## Offline Support

### Service Worker Caching
- **Static assets** cached on install
- **Offline fallback** page
- **Network error** handling
- **Cache version** management

### Data Persistence
- **User data** in localStorage
- **Token** in localStorage
- **Last known location** in memory

## Performance Optimization

### Caching
- **ETag** support for API responses
- **304 Not Modified** handling
- **Service worker** caching
- **CDN resources** cached

### Lazy Loading
- **Images** loaded on demand
- **Charts** initialized when visible
- **Socket connection** established after auth

## Error Handling

### Network Errors
- **Fallback** to cached data
- **Offline indicators**
- **Retry mechanisms**
- **User-friendly messages**

### API Errors
- **Authentication failure** handling
- **Validation error** display
- **Server error** logging
- **Graceful degradation**

## Accessibility

### UI Features
- **Dark mode** support
- **High contrast** text
- **Large touch targets**
- **Clear visual feedback**
- **Loading indicators**

### Responsive Design
- **Mobile-first** approach
- **Adaptive layouts**
- **Touch-friendly** controls
- **Readable fonts**

## Localization

### Language Support
- **Burmese numerals** handling
- **Burmese text** parsing
- **Mixed language** support
- **Unicode** handling

## Third-party Integrations

### Firebase
- **Cloud Messaging** for push notifications
- **Authentication** (optional)
- **Analytics** (optional)

### Cloudflare
- **CDN** for static assets
- **Bot protection**
- **IP detection**
- **Device detection**

## Development Workflow

### Build Process
- **Tailwind CSS** compilation
- **PostCSS** processing
- **Asset optimization**
- **Version management**

### Environment Configuration
- **Database** configuration
- **API endpoints**
- **Firebase** configuration
- **Environment variables**

## Deployment Considerations

### Service Worker Updates
- **Cache versioning**
- **Force reload** mechanism
- **Version notifications**
- **Cache clearing**

### API Versioning
- **v2** endpoints (legacy)
- **v3** endpoints (current)
- **Backward compatibility**
- **Deprecation strategy
