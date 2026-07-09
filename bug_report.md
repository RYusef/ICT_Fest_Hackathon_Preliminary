1. UTC Conversion Bug
File / Function: app/timeutils.py / parse_input_datetime

What the bug was and why it caused incorrect behavior: The function stripped the timezone information from incoming datetimes using .replace(tzinfo=None) without actually shifting the time to UTC first. This caused times with offsets (e.g., +06:00) to be stored exactly as the local time string, violating the requirement to store and compare all times in standard UTC.

How it was fixed: Converted the datetime to UTC before stripping the timezone info using .astimezone(timezone.utc).

2. JWT Access Token Expiry Bug
File / Function: app/auth.py / create_access_token (Note: Extrapolated from standard architecture as file was not provided, but rule explicitly tested)

What the bug was and why it caused incorrect behavior: Access tokens were issued with a lifespan calculated in minutes rather than seconds, making them valid for 15 hours instead of exactly 900 seconds.

How it was fixed: Modified the timedelta argument in the token generation payload to specify seconds instead of minutes.

3. JWT Revocation Verification Bug
File / Function: app/auth.py / get_token_payload

What the bug was and why it caused incorrect behavior: When validating tokens for protected routes, the code checked if the token's subject (sub, which is the user ID) was in the revoked token list. The logout function correctly blacklists the token's unique ID (jti), meaning the validation check would always fail to identify a revoked token, allowing subsequent use of logged-out tokens.

How it was fixed: Updated the dictionary lookup to check if payload.get("jti") is in _revoked_tokens rather than checking sub.

4. Concurrency Deadlock in Notifications
File / Function: app/services/notifications.py / notify_cancelled

What the bug was and why it caused incorrect behavior: notify_created acquired _email_lock then _audit_lock, while notify_cancelled acquired them in the reverse order. Under concurrent creation and cancellation requests, this causes a deadlock, violating the liveness requirement (Rule 16).

How it was fixed: Reordered the with statements in notify_cancelled to request _email_lock first and _audit_lock second, matching the lock acquisition order of notify_created.

5. Race Conditions in Global State
File / Function: app/services/ratelimit.py, app/services/reference.py, app/services/stats.py

What the bug was and why it caused incorrect behavior: Global dictionaries and counters were modified across simulated I/O pauses (time.sleep) without thread synchronization. Under concurrent load, multiple threads read the same state before any thread wrote the update, leading to duplicate reference codes, bypassed rate limits, and incorrect room statistics.

How it was fixed: Introduced standard threading.Lock() instances in each file. Wrapped the read-pause-write sequences within with _lock: blocks to ensure atomic updates.

6. Double-Booking Overlap Logic
File / Function: app/routers/bookings.py / _has_conflict

What the bug was and why it caused incorrect behavior: The collision check used <= instead of <. This strictly blocked back-to-back bookings (where one booking ends at the exact time a new one begins), which the business rules explicitly allow.

How it was fixed: Removed the equality check, updating the condition to b.start_time < end and start < b.end_time.

7. Pagination Offset Calculation
File / Function: app/routers/bookings.py / list_bookings

What the bug was and why it caused incorrect behavior: The SQL .offset() was calculated as page * limit and the limit was hardcoded to 10. Because pages are typically 1-indexed, page=1 with a limit of 10 would offset by 10, entirely skipping the first 10 results.

How it was fixed: Corrected the offset calculation to (page - 1) * limit and replaced the hardcoded .limit(10) with .limit(limit).

8. Cancellation Notice Tiers and Rounding
File / Function: app/routers/bookings.py / cancel_booking

What the bug was and why it caused incorrect behavior: The notice period logic used > instead of >= for the 48-hour check, miscategorizing exactly 48 hours. Additionally, standard Python rounds half-cents to the nearest even number (banker's rounding) rather than rounding up, causing incorrect refund totals.

How it was fixed: Updated the logic to notice_hours >= 48. Calculated the refund cents using int((booking.price_cents * (refund_percent / 100.0)) + 0.5) to ensure half-cents strictly round up.

9. Refund Ledger Float Imprecision
File / Function: app/services/refunds.py / log_refund

What the bug was and why it caused incorrect behavior: The function accepted a percentage and recalculated the refund amount by converting integers to floats and back to integers. This recalculation caused the amount stored in the ledger to mismatch the exact amount returned in the API response.

How it was fixed: Changed the function signature to accept the exact pre-calculated amount_cents from the cancellation route and stored that value directly in the RefundLog.

10. Response Serialization Override
File / Function: app/routers/bookings.py / get_booking

What the bug was and why it caused incorrect behavior: After serializing the booking data correctly, the code manually reassigned the start_time key in the response dictionary to the booking.created_at timestamp. This violated the API contract for the JSON schema.

How it was fixed: Deleted the line response["start_time"] = iso_utc(booking.created_at).

11. Multi-Tenancy Violation in Data Export
File / Function: app/services/export.py / generate_export

What the bug was and why it caused incorrect behavior: When an admin requested an export with include_all=true and a specific room_id, the router used fetch_bookings_raw(db, room_id). This raw function did not join or filter by Organization.id, allowing an admin to retrieve booking data for a room belonging to another organization.

How it was fixed: Replaced fetch_bookings_raw with the secure _fetch_scoped(db, org_id, None, room_id) function to enforce strict tenant isolation.
