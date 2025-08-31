# Airbnb Clone - User Stories 

user_stories:
  - id: US-001
    role: guest
    story: "As a guest, I want to register/login (email/JWT or OAuth) so that I can access and use the platform."
    acceptance_criteria:
      - "User can create an account with email + password."
      - "User can login and receive JWT access/refresh tokens."
      - "OAuth sign-in (e.g., Google) signs in or creates an account."
      - "Profile page is accessible after authentication."

  - id: US-002
    role: guest
    story: "As a guest, I want to search and filter listings by location, dates, price, guests, and amenities so that I can find suitable stays."
    acceptance_criteria:
      - "Search by city/country returns matching listings."
      - "Filters for price range, guests, and amenities narrow results."
      - "Availability filter excludes already-booked dates."
      - "Results are paginated."

  - id: US-003
    role: guest
    story: "As a guest, I want to view property details (photos, amenities, reviews, availability, price) so that I can make an informed decision."
    acceptance_criteria:
      - "Property detail endpoint returns title, description, address, images."
      - "Amenities and nightly price are visible."
      - "Calendar shows available/blocked dates."
      - "Recent reviews are listed with rating and comments."

  - id: US-004
    role: guest
    story: "As a guest, I want to create a booking for available dates so that I can reserve a stay without double-booking."
    acceptance_criteria:
      - "Booking fails if dates overlap existing bookings for the property."
      - "Booking status is initially 'pending' or 'confirmed' per flow."
      - "Total price is calculated and returned."

  - id: US-005
    role: guest
    story: "As a guest, I want to pay securely and receive confirmation so that my reservation is guaranteed."
    acceptance_criteria:
      - "Payment is initiated via provider (Stripe/PayPal) client secret or checkout URL."
      - "On success, booking status updates and receipt/confirmation is sent."
      - "On failure, booking remains pending and user is notified."

  - id: US-006
    role: guest
    story: "As a guest, I want to message the host before/after booking so that I can clarify details."
    acceptance_criteria:
      - "Authenticated guests can send messages to the host tied to a property or booking."
      - "Messages are persisted with timestamps and sender/recipient IDs."
      - "Recipient can fetch a conversation thread."

  - id: US-007
    role: guest
    story: "As a guest, I want to leave a booking-linked review after checkout so that I can share my experience."
    acceptance_criteria:
      - "Review creation only allowed after booking end_date."
      - "Rating must be 1–5; comment is required."
      - "One review per booking per author."

  - id: US-008
    role: host
    story: "As a host, I want to create, edit, and delete listings (with images) so that I can manage my properties."
    acceptance_criteria:
      - "Only users with host role can create listings."
      - "Host can upload images and update listing fields."
      - "Host can soft-delete or archive a listing they own."

  - id: US-009
    role: host
    story: "As a host, I want to manage availability and pricing so that I can control bookings and revenue."
    acceptance_criteria:
      - "Host can block/unblock dates for a listing."
      - "Host can set base nightly rate and adjustments."
      - "Availability changes reflect in search and booking checks."

  - id: US-010
    role: host
    story: "As a host, I want to view and manage incoming bookings so that I can plan for guests."
    acceptance_criteria:
      - "Host sees pending/confirmed/canceled/completed bookings for their listings."
      - "Host can cancel per policy and trigger notifications/refunds as applicable."

  - id: US-011
    role: host
    story: "As a host, I want to receive automatic payouts after completed stays so that I’m compensated on time."
    acceptance_criteria:
      - "System marks bookings 'completed' after stay."
      - "Payout job records payment to host with status and reference."
      - "Host can view payout history."

  - id: US-012
    role: admin
    story: "As an admin, I want to manage users, listings, bookings, and payments so that I can keep the marketplace healthy and compliant."
    acceptance_criteria:
      - "Admin endpoints to list/suspend users and listings."
      - "Admin can view/correct payment records with audit logs."
      - "Admin can export metrics/reports."

  - id: US-013
    role: platform
    story: "As the platform, I want to process payment webhooks so that booking and payment records remain accurate."
    acceptance_criteria:
      - "Webhook endpoint verifies signatures and idempotency."
      - "Successful events update payment and booking statuses."
      - "Failures are retried and logged with alerts."

  - id: US-014
    role: platform
    story: "As the platform, I want to send email and in-app notifications for booking confirmations, cancellations, and payment updates so that users stay informed."
    acceptance_criteria:
      - "On key events, notification records are created and dispatched."
      - "Emails are sent via provider (e.g., SendGrid) with templating."
      - "Users can fetch unread notifications."
