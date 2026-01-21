# Monk Commerce - Coupons Management API

This project implements a RESTful API for managing and applying different types of discount coupons (cart-wise, product-wise, and BxGy) for an e-commerce platform. The design is extensible to support future coupon types using an abstract `Coupon` class with single-table inheritance.

## Setup Instructions

1. Clone the repository: `git clone https://github.com/Srinivasjinna06/Monk_Commerce.git` 
2. Navigate to the project directory: `cd demo`
3. Build and run the application using Maven:
   - `mvn clean install`
   - `mvn spring-boot:run`
4. Access the H2 database console at `http://localhost:8080/h2-console` (JDBC URL: `jdbc:h2:mem:testdb`, User: `sa`, Password: blank).
5. Test APIs using Postman or curl (see API Details below).

## 🛠 Technical Architecture & Design Patterns

This project follows **Domain-Driven Design (DDD)** principles and utilizes several key patterns to ensure a production-ready codebase:

* **Strategy Pattern (via JPA Inheritance):** Implemented the `SINGLE_TABLE` inheritance strategy for the `Coupon` entity. This makes the system **Open-Closed**; new coupon types (like Category-wise or Flash Sales) can be added by creating a new class without modifying existing database schemas or complex joins.
* **Global Exception Handling:** Centralized error management using `@ControllerAdvice` to provide meaningful API feedback and consistent HTTP status codes.
* **Service Layer Isolation:** Encapsulated complex discount logic within the Service layer, keeping Controllers thin and ensuring high maintainability.


## 🚀 API Reference

| Method | Endpoint | Description | Key Feature |
| :--- | :--- | :--- | :--- |
| `POST` | `/coupons` | Create Coupon | Single-table persistence |
| `GET` | `/coupons` | List All | Metadata retrieval |
| `POST` | `/applicable-coupons` | Analysis | Dynamic cart evaluation |
| `POST` | `/apply-coupon/{id}` | Execution | Real-time price calculation |


## 🧠 Logic Breakdown: BxGy (Buy X Get Y)

The BxGy implementation handles sophisticated retail scenarios that go beyond basic discounts:

* **Repetition Limit:** Prevents "revenue leakage" by capping how many times a deal can apply (e.g., "Buy 1 Get 1" valid only for the first 3 pairs).
* **Smart Selection:** The algorithm automatically identifies the **lowest-priced qualifying items** to mark as "Free," ensuring business margins are protected while delivering value to the customer.
* **Atomic Eligibility:** The system performs a dual-check on both "Buy" requirements and "Get" availability before applying any discount.



## API Details

### 1. POST /coupons

- **Purpose**: Create a single coupon.
- **Request Body**:
  ```json
  {
    "coupon_type": "CART_WISE",
    "name": "10PercentOff",
    "expirationDate": "2025-09-29T12:00:00",
    "threshold": 100,
    "discountPercentage": 10
  }
  ```
- **Response (200 OK)**:
  ```json
  {
  "id": 1,
  "name": "10PercentOff",
  "expirationDate": "2025-09-29T12:00:00",
  "threshold": 100,
  "discountPercentage": 10
  } 
  ```
  - **Exceptions/Errors**:
  1.400 Bad Request: Invalid coupon_type or missing fields (e.g., "Type definition error").
  2.500 Internal Server Error: Database save failure (rare, log for details).

### 2. POST /coupons/bulk

- **Purpose: Create multiple coupons in one request (extension).**
- **Request Body** :

````json
[
  {
    "coupon_type": "CART_WISE",
    "name": "10PercentOff",
    "expirationDate": "2025-09-29T12:00:00",
    "threshold": 100,
    "discountPercentage": 10
  },
  {
    "coupon_type": "PRODUCT_WISE",
    "name": "ProductDiscount",
    "expirationDate": "2025-09-29T12:00:00",
    "productId": 1,
    "discountPercentage": 20
  }
]
````
- **Response (200 OK)**:
````json
[
  {
    "id": 1,
    "name": "10PercentOff",
    "expirationDate": "2025-09-29T12:00:00",
    "threshold": 100,
    "discountPercentage": 10
  },
  {
    "id": 2,
    "name": "ProductDiscount",
    "expirationDate": "2025-09-29T12:00:00",
    "productId": 1,
    "discountPercentage": 20
  }
]
````
-**Exceptions/Errors**:
400 Bad Request: Empty or null list ("Coupon list cannot be empty").
400 Bad Request: Invalid coupon data in the list.



### 3. GET /coupons
Purpose: Retrieve all coupons.
Request: No body.
Response (200 OK):
```json
[
  {
    "id": 1,
    "name": "10PercentOff",
    "expirationDate": "2025-09-29T12:00:00",
    "threshold": 100,
    "discountPercentage": 10
  }
]
```
Response (200 OK, if empty): []
Exceptions/Errors: None (empty list is valid).

### 4. GET /coupons/{id}
Purpose: Retrieve specified coupon deatils.
Request: No body.
Response (200 OK):
```json
{
  "coupon_type": "CART_WISE",
  "id": 3,
  "name": "15PercentOff",
  "expirationDate": "2025-10-29T12:00:00",
  "threshold": 150.0,
  "discountPercentage": 15.0
}
```

Exceptions/Errors:
400 Not Found: "Coupon not found" if ID doesn’t exist.


### 5. PUT /coupons/{id}
Purpose: Update an existing coupon.
Request Body (e.g., update coupon ID 1):
```json
{
  "coupon_type": "CART_WISE",
  "name": "15PercentOff",
  "expirationDate": "2025-10-29T12:00:00",
  "threshold": 150,
  "discountPercentage": 15
}
```
- **Response (200 OK)**:
```json
{
  "id": 1,
  "name": "15PercentOff",
  "expirationDate": "2025-10-29T12:00:00",
  "threshold": 150,
  "discountPercentage": 15
}
```
- **Exceptions/Errors**:
404 Not Found: "Coupon not found with id: {id}" if ID doesn’t exist.
400 Bad Request: "Coupon type cannot be changed" if coupon_type mismatches.
400 Bad Request: Invalid input (e.g., negative threshold, add @Positive for validation).


### 6. DELETE /coupons/{id}

Purpose: Delete a coupon by ID.
Request: No body.
Response (204 No Content): No content (success).
-**Exceptions/Errors**:
404 Not Found: "Coupon not found with id: {id}" if ID doesn’t exist (current implementation no-op; enhance for 404).


### 7. POST /applicable-coupons
Purpose: Fetch all applicable coupons for a cart.
Request Body:
```json
{
  "items": [
    {
      "productId": 1,
      "quantity": 4,
      "price": 50
    },
    {
      "productId": 2,
      "quantity": 1,
      "price": 30
    }
  ]
}
```

- **Response (200 OK, with BxGyCoupon)**:
```json
[
  {
    "couponId": 1,
    "type": "bxgy",
    "discount": 50.0
  }
]
```
- **Response (200 OK, if none applicable)**: []
- **Exceptions/Errors**:
400 Bad Request: "Cart or items cannot be null" if body is invalid.


### 8. POST /apply-coupon/{id}

Purpose: Apply a specific coupon to a cart.
Request Body (e.g., for coupon ID 1):
```json
{
  "items": [
    {
      "productId": 1,
      "quantity": 4,
      "price": 50
    },
    {
      "productId": 2,
      "quantity": 1,
      "price": 30
    }
  ]
}
```

- **Response (200 OK, for BxGyCoupon)**:
````json
{
  "items": [
    {
      "productId": 1,
      "quantity": 4,
      "price": 50,
      "totalDiscount": 50.0
    },
    {
      "productId": 2,
      "quantity": 1,
      "price": 30,
      "totalDiscount": 0.0
    }
  ],
  "totalPrice": 230.0,
  "totalDiscount": 50.0,
  "finalPrice": 180.0
}
````
- **Exceptions/Errors**:
404 Not Found: "Coupon not found with id: {id}".
400 Bad Request: "Coupon not applicable to this cart" if conditions aren’t met.


### Implemented Use Cases:

- Cart-wise: Discount (percentage) if cart total exceeds a threshold.
- Product-wise: Discount (percentage) on specific product IDs.
- BxGy: Buy X of certain products, get Y free with a repetition limit (e.g., buy 2 of 1 and 1 of 2,  get 1 of 1 free).
- Bulk Creation: Add multiple coupons efficiently.
- Update Flexibility: Update all fields of existing coupons.

### Unimplemented Use Cases:

- Stackable Coupons: Allow multiple coupons to apply simultaneously (complex conflict resolution).
- User-Specific Limits: Restrict coupons per user (requires auth, memberships).
- Category-Based Discounts: Apply to product categories (needs category schema).
- Fixed Amount Discounts: Support fixed discounts vs percentages.
- Exclusions: Exclude certain products from discounts.
- Flash Sales: Time-bound coupons with start dates.
- Bulk Apply/Update/Delete: Apply/update/delete multiple coupons.
- Conflict Resolution: Handle overlapping coupon rules.
- Auditing: Track coupon usage history.
- Large-Scale Performance: Optimize for thousands of coupons/carts.
- Multi-Currency: Handle different currencies.
- Tax Integration: Adjust discounts post-tax.

### Assumptions:

- Carts are request-based (no persistence beyond the API call).
- All prices are pre-tax and positive.
- No authentication/authorization is required.
- In-memory H2 database is sufficient for this task.
- Quantities and thresholds are integers.
- Expiration dates are in the future or null (no start dates).

### Limitations:

- No coupon stacking; only one applies per cart.
- No pagination or filtering for GET /coupons.
- Basic error handling (e.g., no custom validation beyond null checks).
- In-memory H2 resets on app restart (file-based option available).
- No caching for performance (e.g., Redis).
- No rollback for bulk creation failures.

### Future Provisions:

- Scalability: Add Redis caching for frequent coupon lookups.
- Security: Implement Spring Security for user roles (e.g., admin for coupon management).
- Persistence: Switch to MySQL/PostgreSQL for production with JPA.
- Batch Operations: Extend with PUT /coupons/bulk and DELETE /coupons/bulk.
- Advanced Validation: Add @Valid with constraints (e.g., @Positive for prices).
- Analytics: Track coupon usage with a separate audit table.
- Dynamic Rules: Support JSON-based custom coupon logic.
- Multi-Tenant Support: Allow coupon isolation per store/region.

### Exception Handling and Error Indications:

- 400 Bad Request: Invalid input (e.g., null cart, type mismatch, empty bulk list).
- 404 Not Found: Coupon not found (e.g., invalid ID for update/delete).
- 405 Method Not Allowed: Incorrect HTTP method (e.g., POST instead of PUT).
- 500 Internal Server Error: Unexpected errors (e.g., database issues); logged for debugging.
- Errors return a message (e.g., "Coupon not applicable to this cart") via GlobalExceptionHandler.

### Suggestions for Improvement:

- Add unit tests with JUnit and Mockito.
- Implement pagination for GET /coupons (e.g., ?page=0&size=10).
- Enhance error handling with specific exception types (e.g., CouponValidationException).
- Deploy with Docker for consistency.

````
