# How it works

Both workflows start with a webhook that Retell calls as a custom function, and both end by returning JSON that the voice agent reads and acts on.

The `Normalize` nodes accept the data either at `body.args.<field>` (how Retell sends function arguments) or directly at `body.<field>`, so you can also test with a plain POST.

## Workflow 1: Check Availability

**Webhook path:** `retell-check-availability` (POST)

**Request**

```json
{ "args": { "date": "2026-10-20" } }
```

**Steps**

1. `Normalize Request` reads the date and sets the time zone.
2. `Get Booked Events` pulls every event on that day from Google Calendar.
3. `Find Open Slots` builds 30-minute slots between 9:00 and 17:00 on weekdays, drops any slot that overlaps a booked event or is already in the past, and treats an all-day event as a blocked day.
4. `Send Slots to Retell` returns the result.

**Response**

```json
{
  "date": "2026-10-20",
  "timezone": "America/Chicago",
  "available_slots": [
    { "start_time": "2026-10-20T09:00:00-05:00", "label": "9:00 AM" }
  ],
  "result": "Open 30-minute start times on Tuesday, October 20 (Central Time): 9:00 AM, ... Offer two or three of these, spread out. Use the matching start_time when booking."
}
```

The `result` sentence is written for the voice agent. It tells the agent what to say next, which keeps the prompt shorter. If the date cannot be read, `result` tells the agent to ask the caller for a date again. If the day is full or a weekend, it tells the agent to offer another weekday.

## Workflow 2: Book Appointment

**Webhook path:** `retell-book-appointment` (POST)

**Request**

```json
{
  "args": {
    "name": "Sample Caller",
    "phone": "+15125550100",
    "service": "Consultation",
    "start_time": "2026-10-20T09:00:00-05:00"
  }
}
```

If `phone` is missing, the workflow falls back to the caller's number Retell sends in `call.from_number`. If `service` is missing it uses "Consultation".

**Steps**

1. `Normalize Booking` cleans the fields and works out the end time (start plus 30 minutes) and a spoken-friendly label like "Tuesday, October 20 at 9:00 AM Central".
2. `Create Calendar Event` creates the event. If this step errors, the workflow takes the error output.
3. On success, three things run in parallel: `Log to Call Sheet` (appends a row), `Email Clinic Owner` (sends a summary) and `Confirm Booking to Retell` (returns the confirmation).
4. On failure, `Report Booking Failure` returns a message telling the agent to apologize, take a name and number, and promise a callback.

**Success response**

```json
{ "status": "booked", "result": "Booked Consultation for Sample Caller on Tuesday, October 20 at 9:00 AM Central. Confirm this to the caller." }
```

**Failure response**

```json
{ "status": "failed", "result": "The booking did not go through. Apologize, take the caller name and number, and tell them the clinic will call back to confirm a time." }
```

## Why it is built this way

- **The workflow writes the sentence the agent says.** Putting the wording in the response means a change to what the agent says can be made in n8n without touching the Retell prompt.
- **A failed booking is never reported as a success.** The error output of the calendar step goes to its own response branch.
- **Sheet and email steps do not block the booking.** The caller gets their confirmation even if logging fails.
