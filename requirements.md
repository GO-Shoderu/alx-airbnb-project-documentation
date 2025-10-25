# Airbnb - Backend Requirements Specification
This document defines the technical and functional specifications for the Airbnb Clone backend project. It describes key backend features, API endpoints, validation rules, data handling, and performance criteria to ensure the system is secure, scalable, and reliable.


## Scope
Technical & functional requirements for:
- User Authentication
- Property Management
- Booking System


## User Authentication
### Business Goal
Allow users (guest/host/admin) to register, verify email, authenticate, and manage basic profile securely.

### Functional Requirements
- Register with email + password, optional OAuth.
- Email verification before privileged actions.
- Login returns short-lived access token (e.g., 15–30 min) + refresh token (e.g., 7–30 days).
- Passwords hashed, strong password policy.
- Profile update for name, avatar, phone; role changes restricted to admin.

### Endpoints
#### **POST** `/auth/register`
Create account and send verification email.
```
// Request
{
  "email": "jane@example.com",
  "password": "S3cure!Pass",
  "name": "Jane Doe",
  "role": "guest"
}
// 201 Created
{
  "user": {"id":"...","email":"jane@example.com","name":"Jane Doe","role":"guest","verified":false},
  "message": "Verification email sent"
}
```

##### **Validation**
- `email` valid & unique
- `password` > 8 chats, 1 upper, 1 lower, 1 symbol
- `role` ∈ {guest, host} (admin set by admin)

##### **Errors**
- 409 if email exists; 422 if format invalid


#### **POST** `/auth/verify-email`
Verify email via token sent to user.
```
// Request
{"token":"<verification-token>"}
// 200 OK
{"message":"Email verified"}
```
##### **Errors**
- 400 invalid/expired token


#### **POST** `/auth/login`
```
// Request
{"email":"jane@example.com","password":"S3cure!Pass"}
// 200 OK
{"access_token":"...","refresh_token":"...","expires_in":1800}
```
##### **Errors**
- 401 invalid credentials; 423 if account locked

### Performance Targets
- P50 80ms / P95 250ms for `/auth/login`
- 99.9% uptime for auth cluster
- Email dispatch ≤ 2s (async queue)


## Property Management
### Business Goal
Enable hosts to create and manage listings; allow any user to view/search listings.

### Functional Requirements
- Hosts can CRUD their own properties.
- Public GET endpoints for details and search.
- Images uploaded via pre-signed URLs (optional, out of scope here).

### Endpoints
#### **POST** `/properties` (host auth)
```
// Request
{
  "title":"Modern Studio",
  "description":"Central, walk to station.",
  "location":{"country":"ZA","city":"Pretoria","lat":-25.746,"lng":28.188},
  "base_price": 850.00,
  "currency":"ZAR",
  "max_guests":2,
  "amenities":["wifi","parking"],
  "images":["https://.../img1.jpg"]
}
// 201 Created
{"id":"...","host_id":"...","status":"active", "...": "..."}
```

##### **Validation**
- title 3–120
- base_price ≥ 0; currency ISO 4217
- max_guests 1–20
- lat -90..90, lng -180..180
- amenities from controlled list

#### **GET** `/properties/{id}`
Public property details.
- 200 with full record
- 404 not found/disabled


## Booking System
### Business Goal
Allow guests to reserve available properties for specific dates; prevent double bookings; integrate with payments (out of scope of this file’s endpoints, but referenced).


### Endpoints
#### **POST** `/bookings` (guest auth)
Creates a pending booking & provisional hold
```
// Request
{
  "property_id":"...",
  "start_date":"2025-11-02",
  "end_date":"2025-11-05",
  "guests":2,
  "payment_method":"card",
  "payment_token":"tok_visa_..."
}
// 201 Created
{
  "id":"...",
  "status":"pending",
  "price_total": 2550.00,
  "currency":"ZAR",
  "hold_expires_at":"2025-11-02T10:31:00Z"
}
```

##### **Errors**
- 404 if property does not exist / inactive
- 409 if unavailable (overlap)
- 422 invalid input


#### Performance & Reliability
- P95 booking creation end-to-end ≤ 800ms excluding payment gateway latency (handled async).
- Calendar hold TTL ≤ 15 minutes if payment not completed.
- Idempotency: accept Idempotency-Key header on POST /bookings to prevent duplicates.
- Concurrency: use DB constraints or SELECT ... FOR UPDATE on availability rows.