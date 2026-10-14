# Troubleshooting

Start every investigation in n8n under **Executions**. Open the failing run and look at which node is red.

## The agent says nothing is available on a day that should be open

- The day is a weekend. Slots are only generated Monday to Friday.
- An all-day event exists on that date. Any all-day event blocks the whole day.
- The time zone is wrong. Slots use `America/Chicago`. If your business is elsewhere, change it in all four places listed in the README.
- All of today's slots are in the past. Past slots are removed.

## The agent offers a time and the booking fails

- The `start_time` was not a valid ISO date-time. `Normalize Booking` produces an invalid date and the calendar node errors. Make sure the agent passes back the exact `start_time` value from the availability response, not a time it rephrased.
- The Google Calendar credential expired. Reconnect it in n8n and re-run.
- The calendar ID in `Create Calendar Event` still says `YOUR_CALENDAR_ID@gmail.com`.

## The booking works but there is no sheet row or email

Those two nodes are set to continue on error, so the caller still gets a confirmation. Open the execution and check the red node. The usual causes are an expired Google or Gmail credential, a wrong sheet ID, or sheet columns that do not match the six names in the README.

## The agent confirms a booking that is not on the calendar

This should not happen, because a calendar failure returns the failure message. If it does, check that the Retell custom function is set to wait for the response and that the agent prompt tells it to confirm only when the response says `booked`.

## The webhook returns "not registered" or 404

The workflow is not active, or you are calling the test URL instead of the production URL. Activate the workflow and use the production URL.

## Two callers booked the same slot

The booking workflow does not re-check availability. Add a Google Calendar "get events" step before `Create Calendar Event` and branch to the failure response if the slot is taken.
