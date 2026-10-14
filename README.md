# AI Receptionist: Retell AI + n8n + Google Calendar

A voice receptionist that answers a phone call, finds an open time on a real Google Calendar, and books the appointment while the caller is still on the line. Retell AI runs the conversation. Two n8n workflows do the calendar work.

> **Demo project.** This was built and tested on a fictional sample business ("Glow Demo Med Spa"). It is a portfolio demo, not a client deployment. Every calendar ID, sheet ID and email in this repo is a placeholder.

## What it does

1. The caller says they want an appointment.
2. Retell calls the **Check Availability** workflow with the requested date.
3. n8n reads that day from Google Calendar and returns open 30-minute slots.
4. The agent offers a few times. The caller picks one.
5. Retell calls the **Book Appointment** workflow with name, phone, service and start time.
6. n8n creates the calendar event, logs the booking to a Google Sheet, emails the owner, and tells the agent it worked so the agent can confirm out loud.
7. If the calendar step fails, n8n tells the agent to apologize and take a callback number instead of pretending it booked.

## How the pieces connect

```
Caller -> Retell voice agent
              |-- check_availability(date) --> n8n: Retell - Check Availability --> Google Calendar (read)
              |-- book_appointment(name, phone, service, start_time) --> n8n: Retell - Book Appointment
                                                                          |-- Google Calendar (create event)
                                                                          |-- Google Sheets (log row)
                                                                          |-- Gmail (email owner)
                                                                          |-- reply to Retell (booked / failed)
```

## What is in this repo

| File | What it is |
| --- | --- |
| `workflows/retell-check-availability.json` | n8n workflow: returns open slots for a date |
| `workflows/retell-book-appointment.json` | n8n workflow: creates the event, logs it, emails the owner, confirms to Retell |
| `docs/how-it-works.md` | The request and response each workflow expects, step by step |
| `docs/troubleshooting.md` | Common failures and how to fix them |

## Setup (about 30 minutes)

1. In n8n, import both JSON files (**Workflows -> Import from file**).
2. Open each Google Calendar, Google Sheets and Gmail node, pick your own credentials, and replace the placeholders: `YOUR_CALENDAR_ID@gmail.com`, `YOUR_GOOGLE_SHEET_ID`, `owner@example.com`.
3. Create a Google Sheet with the columns `Timestamp, Name, Phone, Service, Appointment Time, Status`.
4. Activate both workflows and copy the two production webhook URLs.
5. In Retell, add two custom functions that POST to those URLs (see `docs/how-it-works.md` for the parameters) and mention them in the agent prompt.
6. Make a test call and watch the calendar.

## Settings you will probably change

- **Business hours:** weekdays 9:00 to 17:00, set in the `Find Open Slots` code node.
- **Slot length:** 30 minutes, set in `Find Open Slots` (`slotMinutes`) and in `Normalize Booking` (`plus({ minutes: 30 })`). Change both together.
- **Time zone:** `America/Chicago`, in `Normalize Request`, `Normalize Booking`, `Get Booked Events` and `Find Open Slots`.

## Known limits (honest list)

- One calendar, one location. Multiple providers or rooms would need a routing step.
- The booking workflow does not re-check that the slot is still free. Two callers booking the same slot at the same moment could collide. A re-check before `Create Calendar Event` would fix that.
- No reschedule or cancel flow.
- The Sheets and Gmail steps are set to continue on error, so a failure there will not stop the booking, but it also will not alert you. Watch the n8n execution log.
- Not HIPAA-certified by itself. Production use with health data needs the right agreements with each vendor in the chain.

## Tools

Retell AI, n8n, Google Calendar, Google Sheets, Gmail.
