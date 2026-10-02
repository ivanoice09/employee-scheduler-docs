# Demo

Allow visitors to add shifts on employees on a specific week, after adding it they can edit or delete it. Their access are session-based and will expire after 30 minutes (3 mins for my tests), once expired their newly added shifts will be deleted.

To make the demo beginner friendly this app won't have any authentication, they will be able to test the app right away.

## Implementation plan

1. Create a new Entity called "DemoSession" and a database table called "demo_sessions". Here are the fields:

- `demo_session_id` as PRIMARY KEY
- `created_at`
- `last_active_at` (Please ignore for now, will be used in the future versions)

2. The very first request of my app is the getWeek(). See startup.md and scheduler.md for details. I did not add { withCredentials: true } on its service method because this is the first request that will be sent on startup to avoid triggering the 401 error. So when does the server will add a new session?

3. A row of demo_sessions will be generated when the user saves a schedule. See scheduler.md to see how saving schedules work. At the same time a cookie will also be generated, and together with the response they will be sent and set on the browser.