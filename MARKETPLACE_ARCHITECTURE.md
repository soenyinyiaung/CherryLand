# Marketplace Architecture

## Overview

The Marketplace is a Laravel-based e-commerce platform that provides business listings, product catalogs, and category management for the CherryLand delivery platform. It serves as the backend for marketplace functionality, offering RESTful APIs for managing businesses, products, categories, and promotional content. The system integrates with the image service for media management and implements role-based access control for content management.

## Technology Stack

- **Framework**: Laravel (PHP 8.2)
- **Web Server**: Apache
- **Database**: Relational database (MySQL)
- **Authentication**: Custom middleware-based authentication
- **Authorization**: Permission-based access control
- **Image Management**: External image service integration
- **API**: RESTful API with JSON responses
- **ORM**: Eloquent ORM for database operations
- **Validation**: Laravel request validation

## Directory Structure

```
laravel/
├── Application entry point
├── Artisan command-line interface
├── Application logic
│   ├── HTTP layer
│   │   ├── Controllers
│   │   │   ├── API controllers
│   │   │   │   ├── Market controllers
│   │   │   │   └── Adminer controllers
│   │   │   ├── Web controllers
│   │   │   └── Other controllers
│   │   ├── Middleware
│   │   │   ├── Authentication middleware
│   │   │   ├── Permission middleware
│   │   │   └── Other middleware
│   │   ├── Requests
│   │   │   └── Form request validators
│   │   └── Resources
│   │       └── API resource transformers
│   └── Models
│       ├── Business model
│       ├── Category model
│       ├── Product model
│       ├── Overall model
│       └── OverallImage model
├── Configuration files
├── Database
│   ├── Migrations
│   │   ├── Business table migration
│   │   ├── Category table migration
│   │   ├── Product table migration
│   │   ├── Overall table migration
│   │   ├── Overall image table migration
│   │   └── Cache table migration
│   └── Seeders
├── Public assets
├── Resources
│   ├── Views
│   └── Other resources
├── Routes
│   ├── API routes
│   ├── Web routes
│   └── Console routes
├── Storage
├── Vendor dependencies
├── Docker configuration
└── Environment configuration
```

## Core Components

### 1. Data Models

The application uses Eloquent models to represent marketplace entities with relationships and scopes.

#### Business Model

Represents business entities (restaurants, shops, etc.) in the marketplace:

**Fillable Fields**
- **name**: Business name
- **slug**: URL-friendly unique identifier
- **type**: Business type (restaurant, cake_shop, flower_shop, gift_shop, online_shop, retail, other)
- **description**: Business description
- **logo**: Business logo image URL
- **cover_image**: Cover image URL
- **phone**: Contact phone number
- **address**: Physical address
- **township**: Township location
- **latitude**: Geographic latitude
- **longitude**: Geographic longitude
- **is_active**: Active status flag
- **is_featured**: Featured status flag
- **sort_order**: Display order

**Casts**
- **is_active**: Boolean
- **is_featured**: Boolean
- **latitude**: Decimal with 8 precision
- **longitude**: Decimal with 8 precision

**Relationships**
- **products**: Has many products
- **activeProducts**: Has many available products

**Scopes**
- **active**: Filter active businesses
- **featured**: Filter featured businesses
- **byType**: Filter by business type
- **byTownship**: Filter by township

**Route Key**: Uses slug for route model binding

#### Category Model

Represents product categories for organization:

**Fillable Fields**
- **name**: Category name
- **slug**: URL-friendly unique identifier
- **type**: Category type (food, cake, flower, gift, general)
- **description**: Category description
- **image**: Category image URL
- **is_active**: Active status flag
- **sort_order**: Display order

**Casts**
- **is_active**: Boolean

**Relationships**
- **products**: Has many products
- **activeProducts**: Has many available products

**Scopes**
- **active**: Filter active categories
- **byType**: Filter by category type
- **ordered**: Order by sort_order and name

**Route Key**: Uses slug for route model binding

#### Product Model

Represents products offered by businesses:

**Fillable Fields**
- **business_id**: Foreign key to business
- **category_id**: Foreign key to category
- **name**: Product name
- **slug**: URL-friendly unique identifier
- **description**: Product description
- **image**: Product image URL
- **price**: Regular price
- **sale_price**: Sale price (nullable)
- **product_type**: Product type (food, cake, flower, gift, general)
- **is_available**: Availability status
- **is_featured**: Featured status flag
- **sort_order**: Display order

**Casts**
- **price**: Decimal with 2 precision
- **sale_price**: Decimal with 2 precision
- **is_available**: Boolean
- **is_featured**: Boolean

**Relationships**
- **business**: Belongs to business
- **category**: Belongs to category

**Scopes**
- **available**: Filter available products
- **featured**: Filter featured products
- **byType**: Filter by product type
- **byBusiness**: Filter by business
- **byCategory**: Filter by category

**Accessors**
- **displayPrice**: Returns formatted sale price or regular price
- **originalPrice**: Returns formatted regular price
- **isOnSale**: Returns true if sale price is less than regular price

**Route Key**: Uses slug for route model binding

#### Overall Model

Represents promotional posts and announcements:

**Fillable Fields**
- **title**: Post title
- **content**: Post content
- **image**: Post image URL
- **is_active**: Active status flag
- **sort_order**: Display order

**Relationships**
- **images**: Has many overall images

#### OverallImage Model

Represents multiple images for overall posts:

**Fillable Fields**
- **overall_id**: Foreign key to overall
- **image_url**: Image URL
- **sort_order**: Display order

**Relationships**
- **overall**: Belongs to overall

### 2. API Controllers

Controllers handle HTTP requests and coordinate business logic.

#### Market Controllers

**BusinessController**
- **index**: List all businesses with filtering and pagination
- **show**: Get single business by slug
- **store**: Create new business (requires permission)
- **update**: Update existing business (requires permission)
- **destroy**: Delete business (requires permission)

**CategoryController**
- **index**: List all categories with filtering and ordering
- **show**: Get single category by slug
- **store**: Create new category (requires permission)
- **update**: Update existing category (requires permission)
- **destroy**: Delete category (requires permission)

**ProductController**
- **index**: List all products with filtering and pagination
- **show**: Get single product by slug
- **store**: Create new product (requires permission)
- **update**: Update existing product (requires permission)
- **destroy**: Delete product (requires permission)

#### Adminer Controllers

**OverallController**
- **index**: List all overall posts
- **show**: Get single overall post
- **store**: Create new overall post (requires permission)
- **update**: Update existing overall post (requires permission)
- **destroy**: Delete overall post (requires permission)

### 3. Middleware

Middleware provides authentication and authorization for API endpoints.

#### Marketplace Auth Middleware

Validates authentication for marketplace API access:
- Checks for valid authentication token
- Verifies user session
- Returns 401 for unauthenticated requests
- Applied to all marketplace API routes

#### Permission Middleware

Validates user permissions for specific operations:
- **market.permission:market_businesses**: Permission for business operations
- **market.permission:market_categories**: Permission for category operations
- **market.permission:market_products**: Permission for product operations
- **market.permission:marketplace_overall**: Permission for overall post operations
- Returns 403 for unauthorized operations
- Applied to create, update, and delete operations

### 4. API Routes

API routes are organized under the `/api/market` prefix with authentication middleware.

#### Business Routes

- **GET endpoint for listing businesses**: List all businesses
- **GET endpoint for single business**: Get business by slug
- **POST endpoint for creating business**: Create new business (permission required)
- **PUT endpoint for updating business**: Update business (permission required)
- **DELETE endpoint for deleting business**: Delete business (permission required)

#### Category Routes

- **GET endpoint for listing categories**: List all categories
- **GET endpoint for single category**: Get category by slug
- **POST endpoint for creating category**: Create new category (permission required)
- **PUT endpoint for updating category**: Update category (permission required)
- **DELETE endpoint for deleting category**: Delete category (permission required)

#### Product Routes

- **GET endpoint for listing products**: List all products
- **GET endpoint for single product**: Get product by slug
- **POST endpoint for creating product**: Create new product (permission required)
- **PUT endpoint for updating product**: Update product (permission required)
- **DELETE endpoint for deleting product**: Delete product (permission required)

#### Overall Routes

- **GET endpoint for listing posts**: List all overall posts
- **GET endpoint for single post**: Get overall post by ID
- **POST endpoint for creating post**: Create new overall post (permission required)
- **PUT endpoint for updating post**: Update overall post (permission required)
- **DELETE endpoint for deleting post**: Delete overall post (permission required)

### 5. Database Schema

The database schema defines the structure for marketplace data storage.

#### Business Table

Stores business information:

- **id**: Primary key (auto-increment)
- **name**: Business name (string)
- **slug**: Unique slug for URLs (string, unique)
- **type**: Business type (enum: restaurant, cake_shop, flower_shop, gift_shop, online_shop, retail, other)
- **description**: Business description (text, nullable)
- **logo**: Logo image URL (string, nullable)
- **cover_image**: Cover image URL (string, nullable)
- **phone**: Phone number (string, nullable)
- **address**: Physical address (string, nullable)
- **township**: Township (string, nullable)
- **latitude**: Geographic latitude (decimal, nullable)
- **longitude**: Geographic longitude (decimal, nullable)
- **is_active**: Active status (boolean, default true)
- **is_featured**: Featured status (boolean, default false)
- **sort_order**: Display order (unsigned integer, default 0)
- **timestamps**: Created and updated timestamps

**Indexes**
- Composite index on (is_active, is_featured)
- Index on type
- Index on township

#### Category Table

Stores category information:

- **id**: Primary key (auto-increment)
- **name**: Category name (string)
- **slug**: Unique slug for URLs (string, unique)
- **type**: Category type (enum: food, cake, flower, gift, general)
- **description**: Category description (text, nullable)
- **image**: Category image URL (string, nullable)
- **is_active**: Active status (boolean, default true)
- **sort_order**: Display order (unsigned integer, default 0)
- **timestamps**: Created and updated timestamps

**Indexes**
- Composite index on (is_active, sort_order)
- Index on type

#### Product Table

Stores product information:

- **id**: Primary key (auto-increment)
- **business_id**: Foreign key to business table (constrained, cascade on delete)
- **category_id**: Foreign key to category table (constrained, set null on delete, nullable)
- **name**: Product name (string)
- **slug**: Unique slug for URLs (string, unique)
- **description**: Product description (text, nullable)
- **image**: Product image URL (string, nullable)
- **price**: Regular price (decimal, 10, 2)
- **sale_price**: Sale price (decimal, 10, 2, nullable)
- **product_type**: Product type (enum: food, cake, flower, gift, general)
- **is_available**: Availability status (boolean, default true)
- **is_featured**: Featured status (boolean, default false)
- **sort_order**: Display order (unsigned integer, default 0)
- **timestamps**: Created and updated timestamps

**Indexes**
- Composite index on (business_id, is_available)
- Composite index on (category_id, is_available)
- Composite index on (is_featured, is_available)
- Index on product_type

**Foreign Keys**
- business_id references business table primary key (cascade delete)
- category_id references category table primary key (set null on delete)

#### Overall Table

Stores promotional post information:

- **id**: Primary key (auto-increment)
- **title**: Post title (string)
- **content**: Post content (text)
- **image**: Post image URL (string, nullable)
- **is_active**: Active status (boolean, default true)
- **sort_order**: Display order (unsigned integer, default 0)
- **timestamps**: Created and updated timestamps

#### Overall Image Table

Stores multiple images for overall posts:

- **id**: Primary key (auto-increment)
- **overall_id**: Foreign key to overall
- **image_url**: Image URL (string)
- **sort_order**: Display order (unsigned integer, default 0)
- **timestamps**: Created and updated timestamps

#### Cache Table

Stores cached data for performance optimization:

- **id**: Primary key (auto-increment)
- **key**: Cache key (string, unique)
- **value**: Cache value (text)
- **expiration**: Expiration timestamp (integer, nullable)

## Data Flow

### Business Listing Flow

1. **Client sends GET request** to businesses endpoint
2. **Marketplace auth middleware** validates authentication token
3. **BusinessController.index** receives request
4. **Query builder** applies scopes (active, featured, type, township)
5. **Database query** retrieves businesses with relationships
6. **Eloquent models** hydrate business objects
7. **Response** returns businesses collection with pagination

### Product Creation Flow

1. **Client sends POST request** to products endpoint
2. **Marketplace auth middleware** validates authentication token
3. **Permission middleware** checks market_products permission
4. **BusinessController.store** receives validated request
5. **Request validation** validates input data
6. **Eloquent model** creates product record
7. **Foreign key relationships** established with business and category
8. **Database transaction** commits changes
9. **Response** returns created product with ID

### Category Filtering Flow

1. **Client sends GET request** to categories endpoint with filters
2. **Marketplace auth middleware** validates authentication token
3. **CategoryController.index** receives request
4. **Query builder** applies scopes (active, type, ordered)
5. **Database query** retrieves categories with relationships
6. **Eloquent models** hydrate category objects
7. **Response** returns categories collection ordered by sort_order

## Authentication & Authorization

### Authentication

The marketplace uses custom middleware-based authentication:

- **Token-based**: Authentication token in request headers
- **Session validation**: Validates user session against database
- **Middleware**: marketplace.auth middleware applied to all API routes
- **Error handling**: Returns 401 for unauthenticated requests

### Authorization

Permission-based access control for write operations:

- **Permission system**: Role-based permissions for different operations
- **Permission middleware**: market.permission middleware checks specific permissions
- **Permission types**:
  - market_businesses: Business CRUD operations
  - market_categories: Category CRUD operations
  - market_products: Product CRUD operations
  - marketplace_overall: Overall post CRUD operations
- **Error handling**: Returns 403 for unauthorized operations

## Security Considerations

- **Authentication**: Token-based authentication for all API endpoints
- **Authorization**: Permission-based access control for write operations
- **SQL Injection**: Eloquent ORM prevents SQL injection through parameterized queries
- **Input Validation**: Laravel request validation for all inputs
- **CSRF Protection**: CSRF tokens for web routes
- **XSS Protection**: Output escaping for user-generated content
- **File Upload**: Integration with image service for secure file handling
- **Environment Variables**: Sensitive credentials stored in environment files
- **Database Security**: Foreign key constraints for referential integrity
- **Index Optimization**: Database indexes for query performance

## Performance Optimizations

- **Database Indexes**: Composite indexes on frequently queried columns
- **Eager Loading**: Eloquent relationships loaded with eager loading
- **Query Scopes**: Reusable query scopes for common filters
- **Pagination**: Large datasets paginated for efficient retrieval
- **Caching**: Cache table for expensive query results
- **Foreign Key Constraints**: Optimized foreign key relationships
- **Slug-based Routing**: Slug-based URLs for SEO and performance

## External Integrations

- **Image Service**: External image service for logo, cover, and product images
- **Relational Database**: MySQL for data persistence
- **Authentication System**: Integrated with main platform authentication

## Implementation Details

### Dependency Management

- **Composer**: PHP package manager for dependencies
- **Laravel Framework**: Core framework providing MVC architecture
- **Eloquent ORM**: Database abstraction layer
- **Validation**: Built-in request validation system

### Configuration

Environment variables control application behavior:
- Database connection parameters
- Image service URL
- Authentication configuration
- Permission configuration

### Deployment

- **Docker**: Containerized deployment with Docker Compose
- **Apache**: Web server configuration
- **PHP 8.2**: Runtime environment
- **Volume Mounting**: Live code updates without rebuild
