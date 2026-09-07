# Deployment Architecture

## Overview

The CherryLand delivery platform uses a containerized deployment architecture with Docker Compose orchestration, integrating multiple microservices including web applications, APIs, databases, and infrastructure components. The system follows a microservices pattern with services communicating through a shared Docker network. Public exposure is achieved through Cloudflare Tunnel, which provides secure tunneling without requiring public IP addresses or direct port exposure.

## Technology Stack

- **Container Orchestration**: Docker Compose
- **Reverse Proxy**: Nginx
- **Web Server**: Apache (PHP 8.2)
- **Backend Frameworks**: Laravel (PHP), NestJS (TypeScript), FastAPI (Python), Go
- **Databases**: Relational database (MySQL 8.0), Document database (MongoDB 6.0)
- **Database Admin Tools**: Web-based database administration interfaces
- **Tunnel Service**: Cloudflare Tunnel for secure public exposure
- **Network**: Docker bridge network for inter-service communication
- **Volume Management**: Docker volumes for persistent data storage

## Infrastructure Services

### Cloudflare Tunnel

- **Purpose**: Provides secure public internet exposure without requiring public IP addresses
- **Architecture**: Cloudflare Tunnel service runs as a container within the Docker network
- **Configuration**: Uses tunnel token for authentication and routing configuration
- **Network Integration**: Connected to internal Docker network for service access
- **Benefits**: 
  - No public IP required
  - DDoS protection through Cloudflare
  - SSL/TLS termination at Cloudflare edge
  - Secure tunneling without direct port exposure
  - Automatic HTTPS
- **Restart Policy**: Always running for continuous availability

### Relational Database

- **Database Type**: MySQL 8.0
- **Deployment**: Containerized with persistent volume storage
- **Port Exposure**: Localhost-only binding for security (not exposed publicly)
- **Environment Configuration**: 
  - Timezone: Asia/Yangon
  - Root password and database name from environment variables
- **Storage**: Persistent Docker volume for data persistence across container restarts
- **Temporary Storage**: Tmpfs for temporary file operations
- **Network**: Internal Docker network for service communication
- **Configuration**: Optimized for high connection limits
- **Restart Policy**: Always running for data availability
- **Use Case**: Primary relational database for PHP applications, Python services, and Go services

### Document Database

- **Database Type**: MongoDB 6.0
- **Deployment**: Containerized with persistent volume storage
- **Port Exposure**: Localhost-only binding for security (not exposed publicly)
- **Environment Configuration**: 
  - Root username, password, and database name from environment variables
- **Storage**: Persistent Docker volume for data persistence
- **Network**: Internal Docker network for service communication
- **Health Monitoring**: Built-in health check with ping command
- **Restart Policy**: Always running for data availability
- **Use Case**: Document database for chat and order storage in TypeScript backend

## Application Services

### Frontend (PHP/Apache)

- **Technology**: PHP 8.2 with Apache web server
- **Deployment**: Containerized with live code mounting
- **Port Exposure**: Localhost-only binding (not exposed publicly)
- **Environment Configuration**:
  - Image service URL for internal service communication
  - Internal authentication key for service-to-service communication
- **Volumes**: Frontend code directory mounted for live updates
- **Network**: Internal Docker network
- **PHP Extensions**: MySQL and PDO extensions for database connectivity
- **Apache Configuration**:
  - KeepAlive enabled for connection reuse
  - Optimized for concurrent connections
- **PHP Configuration**:
  - Memory limit: 512M
  - Max execution time: 60 seconds
  - Timezone: Asia/Yangon
- **Public Access**: Exposed through Cloudflare Tunnel
- **Use Case**: Admin dashboard and web interface

### Nginx Reverse Proxy

- **Technology**: Nginx reverse proxy
- **Deployment**: Containerized with custom configuration
- **Port Exposure**: Internal port only (not exposed publicly)
- **Network**: Internal Docker network
- **Dependencies**: Backend services (PHP applications)
- **Restart Policy**: Always running
- **Public Access**: Exposed through Cloudflare Tunnel
- **Use Case**: Request routing and load balancing for backend services

### Laravel Backend

- **Technology**: Laravel PHP framework
- **Deployment**: Containerized with live code mounting
- **Port Exposure**: Localhost-only binding (not exposed publicly)
- **Environment Configuration**:
  - Database host: internal database container
  - Database credentials from environment variables
- **Volumes**: Laravel code directory mounted for live updates
- **Network**: Internal Docker network
- **Dependencies**: Relational database
- **Restart Policy**: Always running
- **Public Access**: Exposed through Cloudflare Tunnel
- **Use Case**: PHP backend for marketplace and administrative operations

### FastAPI Backend

- **Technology**: FastAPI Python framework
- **Deployment**: Containerized with live code mounting
- **Port Exposure**: Localhost-only binding (not exposed publicly)
- **Environment File**: Environment variables from .env file
- **Volumes**: Backend code directory mounted for live updates
- **Network**: Internal Docker network
- **Dependencies**: Relational database
- **Restart Policy**: Always running
- **Public Access**: Exposed through Cloudflare Tunnel
- **Use Case**: Order parsing service for natural language processing

### NestJS Backend

- **Technology**: NestJS TypeScript framework
- **Deployment**: Containerized with live code mounting
- **Port Exposure**: Localhost-only binding (not exposed publicly)
- **Environment Variables**:
  - Image server URL from environment
- **Volumes**:
  - Application code mounted for live updates
  - Node modules volume for dependency caching
- **Network**: Internal Docker network
- **Restart Policy**: Always running
- **Public Access**: Exposed through Cloudflare Tunnel
- **Use Case**: Real-time communication backend with WebSocket support

### Image Service (Go)

- **Technology**: Go language with Gin framework
- **Deployment**: Containerized with persistent storage
- **Port Exposure**: Localhost-only binding (not exposed publicly)
- **Environment Variables**:
  - Database host: internal database container
  - Database credentials from environment variables
  - Internal authentication key for service protection
- **Volumes**: Image storage directory mounted for persistence
- **Network**: Internal Docker network
- **Restart Policy**: Always running
- **Public Access**: Exposed through Cloudflare Tunnel
- **Use Case**: Image upload, storage, and WebP conversion service

## Database Administration Tools

### Database Administration Interfaces

Web-based database administration tools for database management:

- **Purpose**: Provide web interfaces for database administration
- **Deployment**: Containerized with direct database connectivity
- **Port Exposure**: Localhost-only binding for security (not exposed publicly)
- **Network**: Internal Docker network for database access
- **Restart Policy**: Always running for administrative access
- **Public Access**: Not exposed publicly (admin-only access)
- **Use Case**: Database management, query execution, and data inspection

## Network Architecture

### Docker Network

- **Network Type**: External Docker bridge network
- **Purpose**: Enable inter-service communication within the deployment
- **Isolation**: Services communicate using container names as hostnames
- **Security**: Services isolated from external network except through Cloudflare Tunnel
- **Management**: Pre-created external network for service integration

### Service Communication

Services communicate internally through the Docker network:

- **Frontend → Image Service**: Internal HTTP requests for image operations
- **PHP Applications → Relational Database**: Internal database connections
- **Python Services → Relational Database**: Internal database connections
- **TypeScript Backend → Document Database**: Internal database connections
- **Go Service → Relational Database**: Internal database connections
- **Nginx Proxy → Backend Services**: Internal load balancing and routing

### Public Exposure Architecture

**Cloudflare Tunnel Integration**

The system uses Cloudflare Tunnel as the single public entry point:

- **No Public IP Required**: Services run without public IP addresses
- **Secure Tunneling**: Cloudflare Tunnel creates secure outbound connection
- **Routing Configuration**: Cloudflare Tunnel routes public requests to internal services
- **SSL/TLS Termination**: SSL handled at Cloudflare edge
- **DDoS Protection**: Cloudflare provides DDoS protection
- **Automatic HTTPS**: All public traffic automatically encrypted

**Service Access Patterns**

- **Public Services**: Frontend, backend APIs exposed through Cloudflare Tunnel
- **Internal Services**: Databases, admin tools accessible only internally
- **Port Security**: All services bound to localhost only
- **Network Isolation**: Services isolated in private Docker network
- **Single Entry Point**: Cloudflare Tunnel as only public access point

## Volume Management

### Persistent Data Volumes

**Relational Database Volume**
- **Purpose**: Persistent storage for relational database data
- **Mount Point**: Database data directory
- **Use Case**: Preserves database data across container restarts and updates

**Document Database Volume**
- **Purpose**: Persistent storage for document database data
- **Mount Point**: Database data directory
- **Use Case**: Preserves document database data across container restarts

**Node Modules Volume**
- **Purpose**: Cache for Node.js dependencies
- **Mount Point**: Application node_modules directory
- **Use Case**: Speeds up container rebuilds by caching dependencies

### Application Data Mounts

**Code Directories**
- **Frontend Code**: Mounted for live code updates without rebuild
- **Laravel Code**: Mounted for live code updates without rebuild
- **FastAPI Code**: Mounted for live code updates without rebuild
- **NestJS Code**: Mounted for live code updates without rebuild

**Storage Directories**
- **Image Storage**: Mounted for persistent image storage across container restarts
- **Use Case**: Preserves uploaded images and converted WebP files

## Security Architecture

### Network Security

- **Localhost Binding**: All service ports bound to localhost only
- **Docker Network Isolation**: Services isolated in private network
- **Cloudflare Tunnel**: Secure tunneling without public IP exposure
- **Internal Authentication**: Service-to-service authentication via shared keys
- **No Direct Exposure**: No services directly exposed to internet

### Authentication Mechanisms

- **Internal Auth Key**: Shared secret for internal service communication
- **Session Tokens**: Database-backed session validation for user authentication
- **Environment Variables**: Sensitive credentials stored in environment files
- **Service-Specific Credentials**: Separate credentials for each service

### Access Control

- **Database Admin Tools**: Localhost access only for administrative purposes
- **Service Ports**: Not exposed to external network
- **Cloudflare Tunnel**: Single controlled entry point with DDoS protection
- **SSL/TLS**: Automatic HTTPS through Cloudflare

### Data Protection

- **Volume Encryption**: Filesystem-level encryption recommended for production
- **Environment Files**: Excluded from version control
- **Database Backups**: Regular backup strategy required for data protection
- **Secret Rotation**: Periodic credential rotation recommended for security

## Public Exposure Strategy

### Cloudflare Tunnel Architecture

The deployment uses Cloudflare Tunnel to provide secure public access:

**How It Works**

1. **Tunnel Container**: Cloudflare Tunnel container runs within Docker network
2. **Outbound Connection**: Tunnel establishes outbound connection to Cloudflare edge
3. **Routing Configuration**: Cloudflare routes configured to map public URLs to internal services
4. **Request Flow**: Public requests → Cloudflare edge → Tunnel → Internal service
5. **Response Flow**: Internal service → Tunnel → Cloudflare edge → Public client

**Benefits**

- **No Port Forwarding**: No need to open ports on firewall
- **No Public IP**: No need for public IP address
- **Automatic SSL**: HTTPS provided automatically
- **DDoS Protection**: Cloudflare provides DDoS mitigation
- **Global CDN**: Content delivery through Cloudflare's global network
- **Access Control**: Cloudflare access rules for additional security

**Service Exposure**

- **Frontend**: Exposed through Cloudflare Tunnel for public web access
- **Backend APIs**: Exposed through Cloudflare Tunnel for API access
- **WebSocket Services**: Supported through Cloudflare Tunnel
- **Static Assets**: Served through Cloudflare CDN
- **Databases**: Not exposed publicly (internal only)
- **Admin Tools**: Not exposed publicly (internal only)

### Deployment Flow

**Initial Setup**

1. **Environment Preparation**: Configure environment variables and secrets
2. **Network Creation**: Create Docker network for service communication
3. **Tunnel Configuration**: Configure Cloudflare Tunnel with routing rules
4. **Service Deployment**: Deploy services using Docker Compose
5. **Public Access**: Services accessible through Cloudflare Tunnel URLs

**Service Updates**

1. **Code Changes**: Update code in mounted directories
2. **Service Restart**: Restart affected services
3. **Live Updates**: Changes reflected immediately (bind mounts)
4. **No Downtime**: Zero-downtime deployments possible

**Scaling Considerations**

- **Horizontal Scaling**: Stateless services can be scaled horizontally
- **Load Balancing**: Cloudflare provides load balancing
- **Session Management**: Database-backed sessions enable scaling
- **WebSocket Scaling**: Requires sticky session configuration

## Architecture Summary

### System Structure

The CherryLand delivery platform uses a microservices architecture with:

- **Container Orchestration**: Docker Compose for service management
- **Service Communication**: Internal Docker network for inter-service communication
- **Data Persistence**: Docker volumes for database and storage persistence
- **Code Management**: Bind mounts for live code updates
- **Public Access**: Cloudflare Tunnel for secure public exposure

### Public Access Model

The system achieves public access through:

- **Single Entry Point**: Cloudflare Tunnel as only public access point
- **Secure Tunneling**: Outbound tunnel connection to Cloudflare edge
- **SSL/TLS**: Automatic HTTPS through Cloudflare
- **DDoS Protection**: Cloudflare provides security protection
- **No Direct Exposure**: Services not directly exposed to internet
- **Localhost Binding**: All services bound to localhost for security

### Security Model

The deployment implements security through:

- **Network Isolation**: Services isolated in private Docker network
- **Authentication**: Internal authentication keys and session validation
- **Access Control**: Localhost-only port bindings
- **Data Protection**: Environment variables and volume encryption
- **Tunnel Security**: Cloudflare Tunnel provides secure connection

This architecture provides a secure, scalable, and maintainable deployment with public access achieved through Cloudflare Tunnel without exposing services directly to the internet.
