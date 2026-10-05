# Demo

Allow visitors to add shifts on employees on a specific week, after adding it they can edit or delete it. Their access are session-based and will expire after 30 minutes (3 mins for my tests), once expired their newly added shifts will be deleted.

To make the demo beginner friendly this app won't have any authentication, they will be able to test the app right away.

For simplicity I used a cookie-based session which is storing a UUID inside the browser, the session's duration will be based on the cookie's expiration.

NOTE: Througout this documentation the word "Session", "Cookie", or "UUID" will be treated as one and will be called just "`cookie`", for the sake of consistency and to avoid confusion.

## Implementation plan

1. I'll create a new Entity called "DemoSession" and a database table called "demo_sessions". This table is essential for checking whether the `cookie` is expired or not. Its primary key will be the `cookie` itself and will also be a foreign key for the table shifts, weeks, and task_assignments. The goal is when the `cookie` expires, the other tables whose related to this row will be deleted and then the `cookie` will be deleted lastly.

- `demo_session_id` as PRIMARY KEY (e.g )
- `created_at`
- `last_active_at` (Please ignore for now, will be used in the future versions)

    ```java
    // DemoSession Entity
    @Entity
    @Table(name = "demo_sessions")
    @Getter
    @Setter
    public class DemoSession {

        @Id
        @Column(name = "demo_session_id")
        private UUID id;

        @Column(name = "created_at", nullable = false, updatable = false)
        private OffsetDateTime createdAt;

        @Column(name = "last_active_at", nullable = false)
        private OffsetDateTime lastActiveAt;

        @PreUpdate
        protected void onUpdate() {
            lastActiveAt = OffsetDateTime.now();
        }
    }
    ```

2. A row of demo_sessions and a `cookie` will be generated or read by `DemoSessionHelper` through _any_ requests(GET, POST, DELETE, UPDATE). Note: I changed the idea from generating the cookie just by saving the schedule to generating it through any requests, because it didn't give me the 401 problem anymore.

    ```java
    // Inside ScheduleController

    // This snippet of code will be inside of all the existing endpoints.
    UUID demoSessionId = demoSessionHelper
            .readOrCreateDemoSessionID(request, response)
                    .orElseThrow(() -> new IllegalStateException(
                            "Could not resolve demo session"
                    ));
    ```

3. If the `cookie` which was read from the request is invalid, null or expired, it is going to be created by `DemoSessionHelper`.

    ```java
    // this function is located inside DemoSessionHelper.

    public Optional<UUID> readOrCreateDemoSessionID(HttpServletRequest request, HttpServletResponse response) {

        Optional<UUID> demoSessionId = readCookie(request);

        DemoSession demoSessionRow = demoSessionId
                .flatMap(demoSessionRepository::findById)
                .orElse(null);

        // this is where the deletion of the expired cookie should happen but it doesn't
        if (demoSessionRow == null || isExpired(demoSessionRow)) {
            if (demoSessionRow != null) {
                cleanupDemoShiftAndWeekData(demoSessionRow.getId());
                demoSessionRepository.delete(demoSessionRow);
            }
            return createDemoSessionRow(response);
        }

        return demoSessionId;
    }
    ```

## Problems and solutions

1. Cannot delete cookie rows from the database through requests. But scheduled/periodical cleanup utility which invokes the
same function, works perfectly fine

    ```java
    public Optional<UUID> readOrCreateDemoSessionID(HttpServletRequest request, HttpServletResponse response) {

        Optional<UUID> demoSessionId = readCookie(request);

        DemoSession demoSessionRow = demoSessionId
                .flatMap(demoSessionRepository::findById)
                .orElse(null);

        // This part of the code doesnt seem to work:
        if (demoSessionRow == null || isExpired(demoSessionRow)) {
            if (demoSessionRow != null) {
                // this function is also utilized by DemoSessionCleanupUtil class:
                demoSessionUtil.deleteExpiredSession(demoSessionRow.getId());
            }

            return createDemoSessionRow(response);
        }

        return demoSessionId;
    }
    ```