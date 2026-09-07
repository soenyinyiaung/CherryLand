# Cherryland Pickup Frontend Architecture

## Overview

The Cherryland Pickup Frontend is a Progressive Web App (PWA) and React Native application designed for merchants and customers to manage pickup orders, track deliveries, and handle COD collections. It is built with PHP, Alpine.js, Tailwind CSS, and integrates with Telegram Bot API and Socket.IO for real-time features.

## Technology Stack

### Frontend Framework
- **Alpine.js** - Lightweight JavaScript framework for reactive UI components
- **Tailwind CSS** - Utility-first CSS framework for styling
- **i18next** - Internationalization library for multi-language support
- **Chart.js** - Data visualization for dashboard charts
- **MapLibre GL** - Interactive map display
- **React Native** - Mobile app wrapper for Android

### Backend
- **PHP** - Server-side logic and API endpoints
- **MySQL** - Database for orders, ways, merchants, and other data

### PWA Features
- **Service Worker** - Offline caching and background sync
- **Web App Manifest** - Installable as a mobile app
- **App Shortcuts** - Quick access to dashboard, ways, and profile

### Integrations
- **Telegram Bot API** - Order notifications to rider group
- **Socket.IO** - Real-time bidirectional communication
- **Expo Location** - GPS location services for mobile app

## Directory Structure

```
├── API endpoints
│   ├── Authentication endpoint
│   ├── User registration endpoint
│   ├── User logout endpoint
│   ├── Token verification endpoint
│   ├── Pickup orders management endpoint
│   ├── Ways management endpoint
│   ├── Profile management endpoint
│   ├── Password change endpoint
│   ├── Feedback submission endpoint
│   └── Configuration and helper functions
├── Language translations
│   ├── English language translations
│   └── Myanmar language translations
├── Main application page
├── Login page
├── Signup page
├── React Native app wrapper
├── Service worker
├── PWA manifest
├── Version alert script
├── Version information
└── Environment configuration
```

## Key Pages and Features

### Authentication

#### Login Page
- **Purpose:** Merchant authentication
- **Features:**
  - **Login Form:**
    - Phone number or KPay number input
    - Password input
    - Password visibility toggle
    - Remember me functionality
  - **UI Features:**
    - Animated gradient background (sky to blue to cyan)
    - Glass morphism card design
    - Logo display
    - Fade-in-up animation
    - Dark mode support
  - **Session Management:**
    - Token-based authentication
    - Cookie-based token storage
    - Auto-redirect to main app on success
  - **Security:**
    - Bot detection and blocking
    - Token validation via API
    - Secure password handling

#### Signup Page
- **Purpose:** New merchant registration
- **Features:**
  - **Registration Form:**
    - Shop name input
    - Phone number input
    - KPay number input
    - Address input
    - Password input
    - Confirm password input
    - Password validation
  - **UI Features:**
    - Animated gradient background
    - Glass morphism card design
    - Multi-step form with progress indicator
    - Real-time validation
  - **Account Verification:**
    - Pending approval state
    - Warning message for unverified accounts
    - Restricted access until approved

### Main Application

#### Dashboard
- **Purpose:** Overview of pickup operations and statistics
- **Features:**
  - **Statistics Summary Cards:**
    - Total orders count with gradient background
    - Pending ways count
    - Delivered ways count
    - Canceled ways count
    - Color-coded status indicators
  - **Charts Visualization:**
    - Pickup trends line chart
    - Status distribution pie chart
    - Hourly activity bar chart
    - Chart.js with data labels plugin
    - Responsive chart sizing
  - **Filtering Options:**
    - Filter by status (all, pending, delivered, canceled)
    - Filter by date range (from/to)
    - Search by phone number
    - Include date filter toggle
  - **Real-time Updates:**
    - Auto-refresh statistics
    - Live order count updates
    - Status change notifications
  - **Quick Actions:**
    - Navigate to pickup ways
    - Navigate to orders
    - Navigate to profile

#### Pickup Ways
- **Purpose:** Manage delivery orders (ways)
- **Features:**
  - **Ways List Display:**
    - Way ID with clickable link
    - Customer phone number
    - Delivery address
    - Amount (COD amount)
    - Delivery fee
    - Status indicator (pending, delivered, canceled)
    - Delivery time (when delivered)
    - Created timestamp
    - Collection status (collected/not collected)
  - **Create New Way:**
    - Customer phone input
    - Address input
    - Amount input
    - Date picker
    - Form validation
    - Auto-generate way ID
  - **Search and Filter:**
    - Search by phone number
    - Filter by status
    - Filter by date
    - Filter by date range (from/to)
    - Real-time filtering
  - **Pagination:**
    - Page navigation (previous/next)
    - Items per page selection
    - Total count display
    - Page number display
  - **Status Management:**
    - Update status to delivered
    - Update status to canceled
    - Auto-set delivery time on delivered
  - **COD Collection:**
    - Mark as collected/not collected
    - COD amount calculation (with/without delivery fee deduction)
    - Separate collected and not collected views
    - COD summary dashboard
  - **Account Restrictions:**
    - Warning message for pending approval accounts
    - Read-only mode for unverified accounts
    - Create way button disabled for unverified accounts

#### Pickup Orders
- **Purpose:** Manage pickup orders for rider assignment
- **Features:**
  - **Orders List Display:**
    - Order ID
    - Package count
    - Rider count requested
    - Status (pending, confirmed, canceled)
    - Assigned riders list
    - Created timestamp
  - **Create New Order:**
    - Package count input (1-70)
    - Rider count input (1-5)
    - Validation for package and rider limits
    - Auto-generate order ID
  - **Telegram Integration:**
    - Automatic notification to rider group
    - Shop name and phone included
    - Package and rider count displayed
    - Clickable phone number link
    - Message ID stored for cancellation
  - **Order Management:**
    - Update order status (pending to confirmed)
    - Cancel order (within 10 minutes or if pending)
    - Delete order from rider app via socket
    - Telegram cancel notification (reply to original)
  - **Assigned Riders:**
    - View list of assigned riders
    - Rider usernames displayed
    - Real-time assignment updates

#### Profile
- **Purpose:** Manage merchant profile and account settings
- **Features:**
  - **Profile Information Display:**
    - Shop name
    - Phone number
    - KPay number
    - Address
    - Account status (verified, pending)
    - Registration date
  - **Profile Editing:**
    - Update shop name
    - Update phone number
    - Update KPay number
    - Update address
    - Real-time validation
    - Save button with loading state
  - **Password Change:**
    - Current password input
    - New password input
    - Confirm password input
    - Password strength indicator
    - Password visibility toggle
    - Error message for mismatched passwords
  - **Account Status:**
    - View verification status
    - Pending approval warning
    - Verified account badge

#### Settings
- **Purpose:** Application settings and preferences
- **Features:**
  - **Default Page Selection:**
    - Dashboard
    - Pickup Ways
    - Orders
    - Profile
  - **Language Selection:**
    - English
    - Myanmar
    - Real-time language switch
    - Persist preference
  - **Password Change:**
    - Same as profile password change
  - **Feedback:**
    - Feedback form
    - Message input
    - Submit button
    - Success/error messages
  - **User Guide:**
    - Link to documentation
    - Help resources
  - **Logout:**
    - Clear token from cookie
    - Redirect to login page
    - Confirmation dialog

#### COD Collection
- **Purpose:** Track cash on delivery collections
- **Features:**
  - **COD List Display:**
    - Collected items list
    - Not collected items list
    - Customer phone
    - Address
    - Amount
    - Delivery fee
    - COD amount (calculated)
    - Collection status
  - **Filtering:**
    - Filter by date range
    - Filter by phone number
    - Filter by amount
  - **COD Amount Calculation:**
    - Automatic calculation based on delivery fee deduction
    - Display original amount
    - Display delivery fee
    - Display final COD amount
  - **Collection Management:**
    - Toggle collected status
    - Separate collected and not collected views
    - Summary totals

### Real-time Features

#### Socket.IO Integration
- **Purpose:** Live order updates and rider communication
- **Features:**
  - **Order Removal:**
    - Send remove action to riders
    - Remove order from rider app
    - Real-time rider notification
  - **Order Updates:**
    - Status change notifications
    - Rider assignment updates
    - Package count updates
  - **Connection Management:**
    - Auto-reconnection
    - Connection status tracking
    - Error handling

## Key JavaScript Modules

### React Native App Wrapper
- **Purpose:** Wrap web app for mobile distribution
- **Features:**
  - WebView integration for web app
  - Version management with cache busting
  - Location permission handling
  - GPS location fetching
  - PostMessage communication with web
  - Hardware back button handling
  - Loading screen with branding
  - Cache mode configuration
  - User agent customization

### Service Worker
- **Purpose:** Offline caching and PWA support
- **Features:**
  - Cache static assets (HTML, manifest)
  - Cache CDN resources (Tailwind, Alpine, Chart.js)
  - Network-first for API requests
  - Cache-first for static resources
  - Offline fallback for HTML requests
  - Cache version management (v1)
  - Automatic cache cleanup on update
  - Skip API requests for dynamic data

### Version Alert
- **Purpose:** App version checking and updates
- **Features:**
  - Fetch version information from server
  - Compare with local version
  - Force reload if version mismatch
  - Display update notification
  - Cache busting with version parameter

## API Integration

### Authentication
- **Token-based authentication** using Bearer tokens
- **Cookie-based token storage** for web
- **Token verification** on each request
- **Session management** via database
- **Bot detection** and blocking
- **Rate limiting** per merchant

### Pickup Orders
- **Create pickup order** with Telegram notification
- **Fetch pickup orders** with status filter
- **Update order status** (pending to confirmed)
- **Cancel pickup order** with time limit (10 minutes)
- **Delete from rider app** via socket
- **Telegram cancel notification** (reply to original)
- **View assigned riders**

### Ways Management
- **Create new way** with customer details
- **Fetch ways** with pagination
- **Filter ways** by status, date, phone
- **Update way status** (pending, delivered, canceled)
- **Auto-set delivery time** on delivered
- **Dashboard data** for statistics
- **COD tracking** with collection status
- **Phone search** for dashboard

### Profile Management
- **Fetch profile information**
- **Update profile details**
- **Change password**
- **Submit feedback**

### Real-time
- **Socket.IO connection** for live updates
- **Order removal notifications** to riders
- **Status change broadcasts**

## PWA Features

### Service Worker
- **Caching Strategy:**
  - Cache static assets (HTML, manifest)
  - Cache CDN resources (Tailwind, Alpine, Chart.js, i18next)
  - Network-first for API requests
  - Cache-first for static resources
  - Offline fallback for HTML requests
- **Background Sync:**
  - No specific background sync implemented
- **Cache Management:**
  - Cache version management (v1)
  - Automatic cache cleanup on update
  - Individual URL caching with error handling

### Manifest
- **App Name:** Pickup Delivery App
- **Short Name:** Pickup
- **Display Mode:** Standalone
- **Orientation:** Portrait
- **Theme Color:** #3b82f6
- **Start URL:** Pickup app path
- **Icons:** 192x192 and 512x512 PNG
- **Categories:** business, delivery
- **Shortcuts:**
  - Dashboard shortcut
  - Pickup Ways shortcut
  - Profile shortcut

## Internationalization

### Language Support
- **English** - Primary language
- **Myanmar** - Secondary language
- **i18next** integration
- **Browser language detection**
- **Manual language switch**
- **Persistent language preference**
- **Translation files:**
  - Common terms (loading, save, cancel, etc.)
  - Navigation labels
  - Dashboard labels
  - Ways labels
  - Orders labels
  - Settings labels
  - Status labels
  - Messages
  - COD labels

## Security Features

### Authentication
- **Bearer token** validation
- **Cookie-based token** storage
- **Token expiration** checking
- **Session management** via database
- **Rate limiting** per merchant

### Bot Protection
- **User agent analysis**
- **Comprehensive bot pattern** matching
- **IP address** validation
- **Request method** validation
- **Input sanitization**

### Security Headers
- **Content Security Policy**
- **X-Frame-Options**
- **X-Content-Type-Options**
- **X-XSS-Protection**
- **Referrer-Policy**

### Input Validation
- **Sanitize all inputs**
- **Validate package count** (1-70)
- **Validate rider count** (1-5)
- **Validate phone numbers**
- **Validate pagination parameters**

## Database Integration

### Tables Used
- **Merchants table** - Merchant account information
- **Pickup orders table** - Pickup order records
- **Ways table** - Delivery way records
- **Order raw table** - Raw order messages from Telegram
- **Merchant sessions table** - Session management

## Mobile App Integration

### React Native Features
- **WebView** for web app display
- **Location permissions** for GPS
- **PostMessage** communication
- **Hardware back button** handling
- **Safe area** handling
- **Status bar** configuration
- **Loading screen** with branding
- **Cache mode** for performance

### Location Services
- **Expo Location** integration
- **Permission request** (Android/iOS)
- **Current position** fetching
- **Timeout handling** (15 seconds)
- **Accuracy configuration** (balanced)
- **Error handling** for failures

## Telegram Integration

### Order Notifications
- **Automatic notification** to rider group
- **Shop name** and phone included
- **Package count** displayed
- **Rider count** displayed
- **Clickable phone** link (tel: protocol)
- **HTML parse mode** for formatting
- **Message ID** stored for cancellation

### Cancel Notifications
- **Reply to original** message
- **Order ID** included
- **Cancel confirmation** in Burmese
- **Error handling** for failed notifications

## Version Management

### App Configuration
- **Web version** tracking
- **Force reload** flag
- **Last updated** timestamp
- **Latest version** for mobile app
- **APK file name**
- **Download URL** for APK
- **Cache busting** with version parameter

### Update Mechanism
- **Version check** on app load
- **Compare with stored version**
- **Force reload** if mismatch
- **Update notification** display
- **Automatic cache clearing**

## Offline Support

### Service Worker Caching
- **Static assets** cached on install
- **CDN resources** cached on fetch
- **Offline fallback** for HTML requests
- **Network error** handling
- **Cache version** management

### Data Persistence
- **Token** in cookie
- **Language preference** in localStorage
- **Version** in AsyncStorage (mobile)

## Performance Optimization

### Caching
- **Service worker** caching
- **CDN resources** cached
- **Cache mode** configuration (LOAD_CACHE_ELSE_NETWORK)
- **Version-based cache busting**

### Lazy Loading
- **Charts** initialized when visible
- **Maps** loaded on demand
- **Translations** loaded on init

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

## Deployment Considerations

### Service Worker Updates
- **Cache versioning**
- **Force reload** mechanism
- **Version notifications**
- **Cache clearing**

### Environment Configuration
- **Database** configuration
- **Telegram bot** configuration
- **Developer code** for sensitive operations
- **Environment variables** from .env file

## Development Workflow

### Build Process
- **No build step** for web (CDN resources)
- **React Native** build for mobile app
- **APK generation** for Android
- **Version management** via version.json

### Environment Configuration
- **.env file** for local development
- **Environment loader** for variables
- **Database connection** configuration
- **Telegram bot** configuration
