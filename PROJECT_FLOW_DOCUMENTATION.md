# Online Food Delivery System - Project Flow Documentation

## 🚀 Project Overview

The Online Food Delivery System is a comprehensive microservices-based application built with:
- **Frontend**: Angular 18+ (CosmicCartAngular)
- **Backend**: Spring Boot Microservices Architecture
- **Database**: Multiple databases per service (H2/MySQL)
- **Security**: JWT-based authentication with API Gateway
- **Architecture**: Microservices with Eureka Service Discovery

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Security & Authentication Flow](#security--authentication-flow)
3. [Microservices Breakdown](#microservices-breakdown)
4. [Database Schema & Operations](#database-schema--operations)
5. [Complete User Journeys](#complete-user-journeys)
6. [API Flow & Communication](#api-flow--communication)
7. [Interview Ready Explanations](#interview-ready-explanations)

---

## 🏗️ Architecture Overview

### System Architecture
```
Frontend (Angular) → API Gateway → Microservices → Databases
                                      ↓
                               Eureka Service Discovery
```

### Key Components:
1. **CosmicCartAngular**: Angular frontend application
2. **API Gateway**: Central entry point with security filtering
3. **Eureka Server**: Service discovery and registration
4. **Auth Service**: JWT token generation and validation
5. **Customer Service**: Customer management and profiles
6. **Restaurant Service**: Restaurant and menu management
7. **Order Service**: Order processing and lifecycle
8. **Payment Service**: Payment processing
9. **Delivery Service**: Delivery management

---

## 🔐 Security & Authentication Flow

### Authentication Process

#### 1. Customer Login Flow
```
Customer Login → Auth Service → Customer Service → JWT Generation → Frontend Storage
```

**Step-by-step Process:**

1. **Frontend (Angular)**:
   - User enters credentials in `customer-login.component.ts`
   - Form validation occurs
   - POST request to `http://localhost:9095/api/auth/customer/login`

2. **API Gateway** (`SecurityConfig.java`):
   - Allows unauthenticated access to `/api/auth/**`
   - No JWT validation required for login endpoints
   
   **Why This Security Exception is Needed:**
   ```java
   // The Problem: Users need JWT tokens to access protected endpoints,
   // but they can't get JWT tokens without logging in first!
   // Solution: Create security exceptions for authentication endpoints
   
   .pathMatchers(HttpMethod.POST,"/api/auth/**").permitAll()  // ← Login endpoints
   .pathMatchers(HttpMethod.POST,"/api/customers/register").permitAll()  // ← Registration
   .anyExchange().authenticated()  // Everything else requires JWT
   ```
   
   **What This Means:**
   - **Public Endpoints**: `/api/auth/customer/login`, `/api/auth/restaurant/login`, `/api/customers/register`
   - **Protected Endpoints**: All other URLs require valid JWT token
   - **Bootstrap Solution**: Solves the chicken-and-egg problem of getting initial authentication

3. **Auth Service** (`AuthController.java`):
   - Receives login request at `/api/auth/customer/login`
   - Calls `AuthService.authenticateUser()`
   - Can process without JWT because API Gateway permitted access

4. **Authentication Process** (`AuthService.java`):
   ```java
   // Fetch user details from Customer Service
   UserAuthDetailsDTO userDetails = customerServiceClient.getUserByUsername(email);
   
   // Validate password
   passwordEncoder.matches(password, userDetails.getHashedPassword());
   
   // Create JWT claims
   Map<String, Object> claims = new HashMap<>();
   claims.put("userId", userDetails.getId());
   claims.put("username", userDetails.getUsername());
   claims.put("email", userDetails.getEmail());
   claims.put("roles", userDetails.getRoles());
   
   // Generate JWT token
   String jwtToken = jwtService.generateToken(userDetails.getUsername(), claims);
   ```

5. **JWT Token Generation** (`JwtService.java`):
   ```java
   // Token contains user information and expires based on configuration
   .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION_TIME_MS))
   .signWith(getSigningKey(), SignatureAlgorithm.HS256)
   ```

6. **Frontend Token Storage** (`AuthService.ts`):
   ```typescript
   // Store JWT in localStorage
   localStorage.setItem('jwt', token);
   
   // Extract user ID from token
   const userId = this.getUserIdFromToken();
   this.userIdSubject.next(userId);
   ```

### Authorization Flow

#### 1. Protected Route Access
```
Request → API Gateway → JWT Validation → Microservice → Response
```

**Security Filter Chain:**

1. **API Gateway Security** (`SecurityConfig.java`):
   ```java
   // Public endpoints
   .pathMatchers(HttpMethod.POST,"/api/auth/**").permitAll()
   .pathMatchers(HttpMethod.POST,"/api/customers/register").permitAll()
   
   // Protected endpoints require authentication
   .anyExchange().authenticated()
   ```

2. **JWT Validation**:
   - JWT is automatically validated by Spring Security
   - Token signature verified using secret key
   - Claims extracted for authorization

3. **Authorization Filter** (`CentralizedAuthFilter.java`):
   ```java
   // Role-based access control
   boolean hasRequiredRole = roles.contains(requiredRole);
   
   // User ID matching for resource access
   Long pathId = extractIdFromPath();
   if (!userIdFromToken.equals(pathId)) {
       return denyAccess();
   }
   
   // Add user context to request headers
   .header("X-Internal-User-Id", userId)
   .header("X-Internal-User-Roles", roles)
   ```

### Security Features

1. **JWT Secret Management**: Configured in `application.properties`
2. **Token Expiration**: Configurable expiration time
3. **Role-Based Access**: CUSTOMER, RESTAURANT roles
4. **Resource Protection**: Users can only access their own data
5. **CORS Configuration**: Proper cross-origin handling

---

## 🔧 Microservices Breakdown

### 1. Auth Service (Port: 8081)
**Purpose**: Centralized authentication and JWT management

**Key Classes**:
- `AuthController.java`: Login endpoints
- `AuthService.java`: Authentication logic
- `JwtService.java`: JWT operations

**Endpoints**:
- `POST /api/auth/customer/login`
- `POST /api/auth/restaurant/login`

### 2. Customer Service (Port: 8082)
**Purpose**: Customer registration, profile management

**Database Schema**:
```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255),
    phone BIGINT,
    address VARCHAR(255)
);
```

**Key Operations**:
- Customer registration with password encryption
- Profile retrieval and updates
- Email uniqueness validation

### 3. Restaurant Service (Port: 8083)
**Purpose**: Restaurant management and menu items

**Database Schema**:
```sql
CREATE TABLE restaurants (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    location VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) UNIQUE NOT NULL
);
```

### 4. Order Service (Port: 8084)
**Purpose**: Order processing, cart management

**Database Schema**:
```sql
-- Orders table
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    restaurant_id BIGINT NOT NULL,
    status VARCHAR(20) CHECK (status IN ('PENDING', 'ACCEPTED', 'DECLINED', 'IN_COOKING', 'OUT_FOR_DELIVERY', 'COMPLETED', 'CANCELLED')),
    total_amount DECIMAL(10, 2) NOT NULL,
    order_time TIMESTAMP NOT NULL,
    delivery_address TEXT NOT NULL,
    payment_id BIGINT,
    idempotency_key VARCHAR(255) UNIQUE
);

-- Order items
CREATE TABLE order_items (
    id BIGINT PRIMARY KEY,
    order_id BIGINT NOT NULL,
    menu_item_id BIGINT NOT NULL,
    item_name VARCHAR(255) NOT NULL,
    quantity INTEGER NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- Cart tables
CREATE TABLE carts (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    restaurant_id BIGINT
);

CREATE TABLE cart_items (
    id BIGINT PRIMARY KEY,
    cart_id BIGINT NOT NULL,
    menu_item_id BIGINT NOT NULL,
    item_name VARCHAR(255) NOT NULL,
    quantity INTEGER NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (cart_id) REFERENCES carts(id)
);
```

### 5. Payment Service (Port: 8085)
**Purpose**: Payment processing and transaction management

**Payment Methods**:
- Cash on Delivery (COD)
- Card Payment
- Online Payment (Future)

**Payment Flow**:
```java
// Payment status based on method
if (paymentMethod == PaymentMethod.Cash || paymentMethod == PaymentMethod.Card) {
    payment.setPaymentStatus(PaymentStatus.Success);
} else {
    payment.setPaymentStatus(PaymentStatus.Pending);
}
```

---

## 💾 Database Schema & Operations

### Data Storage Pattern
Each microservice maintains its own database following the **Database per Service** pattern:

1. **Customer Service**: Customer profiles and authentication data
2. **Restaurant Service**: Restaurant information and menu items
3. **Order Service**: Orders, order items, and cart data
4. **Payment Service**: Payment transactions and status

### Key Database Operations

#### Customer Registration:
```java
// Password encryption before storage
customer.setPassword(passwordEncoder.encode(dto.getPassword()));
repository.save(customer);
```

#### Order Creation with Transaction:
```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public OrderDTO placeOrder(OrderRequestDTO request) {
    // 1. Create order
    Order order = createOrder(request, idempotencyKey, totalAmount);
    
    // 2. Save order and items
    Order savedOrder = orderRepository.save(order);
    orderItemRepository.saveAll(orderItems);
    
    // 3. Process payment
    PaymentResponseDTO paymentResponse = paymentServiceClient.processPayment(paymentRequest);
    
    // 4. Update order with payment ID
    savedOrder.setPaymentId(paymentResponse.getPaymentId());
    orderRepository.save(savedOrder);
}
```

#### Idempotency Handling:
```java
// Prevent duplicate orders
if (orderRepository.existsByIdempotencyKey(idempotencyKey)) {
    return orderRepository.findByIdempotencyKey(idempotencyKey);
}
```

---

## 👤 Complete User Journeys

### Customer Registration Journey

1. **Frontend Navigation**: User clicks "Register" on landing page
2. **Form Submission**: `customer-register.component.ts` validates and submits data
3. **API Gateway**: Routes request to Customer Service
4. **Validation**: Email uniqueness check in `CustomerServiceImpl.java`
5. **Password Encryption**: BCrypt encoding before database storage
6. **Database Storage**: Customer record saved in customers table
7. **Response**: Success message returned to frontend

### Customer Login Journey

1. **Login Form**: User enters credentials in `customer-login.component.ts`
2. **Authentication**: Auth Service validates credentials
3. **JWT Generation**: Token created with user claims
4. **Frontend Storage**: Token stored in localStorage
5. **Navigation**: User redirected to main page
6. **Profile Loading**: User profile fetched and displayed

### Food Ordering Journey

1. **Restaurant Selection**: User browses restaurants on main page
2. **Menu Display**: `restaurant-page.component.ts` loads menu items
3. **Cart Management**: Items added via `cart.service.ts`
   ```typescript
   // Cart validation for restaurant consistency
   if (currentRestaurant && currentRestaurant.id !== restaurant.id) {
       this.clearCart(); // Clear cart if different restaurant
   }
   ```
4. **Cart Review**: User reviews items in cart page
5. **Checkout Process**: Address and payment method selection
6. **Order Placement**: Order created with idempotency key
7. **Payment Processing**: Payment service handles transaction
8. **Order Confirmation**: Success page with order tracking details

### Order Tracking Journey

1. **Order Status**: Real-time tracking in `order-tracking.component.ts`
2. **Status Updates**: 
   - PENDING → ACCEPTED → IN_COOKING → OUT_FOR_DELIVERY → COMPLETED
3. **Timeline Display**: Visual progress indicator
4. **Auto-refresh**: Every 30 seconds for non-completed orders

---

## 🌐 API Flow & Communication

### Inter-Service Communication

#### Feign Clients
Services communicate using OpenFeign clients:

```java
// Auth Service calls Customer Service
@FeignClient(name = "customer-service")
public interface CustomerServiceClient {
    @GetMapping("/internal/auth/{email}")
    UserAuthDetailsDTO getUserByUsername(@PathVariable String email);
}

// Order Service calls Payment Service
@FeignClient(name = "payment-service")
public interface PaymentServiceClient {
    @PostMapping("/api/payment/initiate")
    PaymentResponseDTO processPayment(@RequestBody PaymentRequestDTO request);
}
```

#### Service Discovery
All services register with Eureka Server for dynamic discovery:

```properties
# Eureka registration
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
eureka.instance.instance-id=${spring.application.name}:${spring.application.instance_id:${random.value}}
```

### Request Flow Example: Place Order

1. **Frontend Request**: 
   ```typescript
   POST /api/orders/place
   Headers: { Authorization: "Bearer <JWT>" }
   ```

2. **API Gateway**: 
   - Validates JWT token
   - Extracts user claims
   - Adds internal headers
   - Routes to Order Service

3. **Order Service**:
   - Receives request with user context
   - Validates user authorization
   - Creates order transaction
   - Calls Payment Service via Feign

4. **Payment Service**:
   - Processes payment
   - Updates payment status
   - Returns payment ID

5. **Response Chain**:
   - Order Service updates order with payment ID
   - Returns order details
   - API Gateway forwards response
   - Frontend displays success page

---

## 🎯 Interview Ready Explanations

### 1. "How does authentication work in your system?"

**Answer**: 
"Our system uses JWT-based stateless authentication with a centralized Auth Service. Here's the complete flow:

**Initial Authentication (Bootstrap Problem Solution):**
1. We have a two-tier security approach where the API Gateway creates exceptions for authentication endpoints (`/api/auth/**`) that allow unauthenticated access
2. This solves the bootstrap problem - users can call the login endpoint without a token to get their initial JWT
3. The Auth Service validates credentials against the Customer Service via Feign client
4. Upon successful validation, it generates a JWT token containing user claims (ID, email, roles)
5. The token is signed with a secret key and has a configurable expiration

**Subsequent Requests:**
6. Frontend stores the token in localStorage
7. For all other requests, the API Gateway validates the JWT and extracts user context
8. Internal services receive user information via headers (X-Internal-User-Id, X-Internal-User-Roles)
9. This eliminates the need for repeated token validation across microservices

**Security Tiers:**
- **Public**: Login, registration endpoints
- **Protected**: All business functionality requires JWT authentication"

### 2. "How do you ensure data consistency across microservices?"

**Answer**:
"We implement several patterns for data consistency:

1. **Database per Service**: Each microservice owns its data, preventing tight coupling
2. **Transactional Boundaries**: Critical operations like order placement use database transactions
3. **Idempotency**: Orders include idempotency keys to prevent duplicate processing
4. **Compensating Transactions**: If payment fails, we can cancel the order
5. **Event Sourcing**: Order status changes are tracked with timestamps for audit trails"

### 3. "How does your system handle security?"

**Answer**:
"Security is implemented at multiple layers with a smart bootstrap approach:

1. **API Gateway Security**: 
   - Central entry point with JWT validation for protected endpoints
   - Strategic security exceptions for authentication endpoints to solve the bootstrap problem
   - Configuration: `permitAll()` for `/api/auth/**` and registration, `authenticated()` for everything else

2. **Authentication Flow**:
   - Public endpoints allow initial token acquisition
   - JWT tokens contain user claims and expire based on configuration
   - Stateless authentication eliminates server-side session management

3. **Authorization & Access Control**:
   - Role-Based Access Control (RBAC) with CUSTOMER/RESTAURANT roles
   - Resource-level authorization: Users can only access their own data
   - Path-based user ID matching prevents data leakage

4. **Data Security**:
   - BCrypt encryption for all passwords before database storage
   - JWT tokens signed with secret keys for integrity verification
   - Internal service communication via trusted headers

5. **Cross-Cutting Concerns**:
   - CORS configuration for proper cross-origin request handling
   - Request context propagation via custom headers (X-Internal-User-Id, X-Internal-User-Roles)"

### 4. "Explain the order flow from cart to delivery"

**Answer**:
"The order flow follows these steps:

1. **Cart Management**: Items stored in localStorage, validated for restaurant consistency
2. **Order Creation**: Atomic transaction creates order and order items
3. **Payment Processing**: Immediate payment processing with status updates
4. **Order Confirmation**: Order ID generated, tracking initiated
5. **Status Updates**: Order progresses through states (PENDING → COOKING → DELIVERY → COMPLETED)
6. **Real-time Tracking**: Frontend polls for updates every 30 seconds
7. **Completion**: Final status update triggers completion flow"

### 5. "How do your microservices communicate?"

**Answer**:
"Services communicate through:

1. **Service Discovery**: Eureka Server manages service registration and discovery
2. **Feign Clients**: Type-safe HTTP clients for service-to-service calls
3. **Load Balancing**: Ribbon provides client-side load balancing
4. **API Gateway**: Single entry point routing requests to appropriate services
5. **Internal APIs**: Services expose internal endpoints for inter-service communication
6. **Header Propagation**: User context passed via custom headers"

### 6. "How do you handle failures and ensure reliability?"

**Answer**:
"We implement several reliability patterns:

1. **Idempotency**: Duplicate requests are safely handled
2. **Timeout Configuration**: Prevent hanging requests
3. **Graceful Degradation**: Critical paths continue working even if some services fail
4. **Transaction Management**: Database consistency maintained with proper transaction boundaries
5. **Error Handling**: Comprehensive exception handling with meaningful error messages
6. **Circuit Breakers**: (Future) Prevent cascade failures between services"

---

## 🔄 System Flow Summary

### Complete User Flow (Login to Order Completion):

1. **Authentication**: Login → JWT Generation → Token Storage
2. **Browse**: Restaurant List → Menu Display → Item Selection
3. **Cart**: Add Items → Validate Restaurant → Review Cart
4. **Checkout**: Address Entry → Payment Method → Place Order
5. **Processing**: Order Creation → Payment Processing → Confirmation
6. **Tracking**: Status Updates → Real-time Monitoring → Completion
7. **Profile**: Order History → User Management → Logout

### Data Flow:
```
Frontend ↔ API Gateway ↔ Auth Service ↔ Customer Service ↔ Database
                    ↕                ↕            ↕
            Order Service ↔ Payment Service ↔ Database
                    ↕
              Restaurant Service ↔ Database
```

This documentation provides a comprehensive overview of the system architecture, security implementation, and complete user journeys that you can confidently explain in any interview setting. 