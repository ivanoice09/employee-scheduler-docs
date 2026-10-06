# Demo

Allow visitors/users to assign shifts on employees on which upon saving it, they can modify or delete it. Their access are session-based and will expire after 30 minutes (3 mins for my tests), once expired their newly added shifts will be deleted.

To make the demo beginner friendly this app won't have any authentication, they will be able to test the app right away.

For simplicity I used a cookie-based session which is storing a UUID inside the browser, the session's duration will be based on the cookie's expiration.

NOTE: Througout the rest of this documentation the words "Session", "Cookie", or "UUID" are treated as one and be called just "`cookie`" even if they're not really the same in a literal sense, just for the sake of consistency and to avoid confusion.

## Implementation

1. I created a new Entity called "DemoSession" and a database table called "demo_sessions". This table is essential for checking whether the `cookie` is expired or not. Its primary key is the `cookie` itself and it is also a foreign key for the table shifts, weeks, and task_assignments. The goal is when the `cookie` expires, the other tables whose related to this row will be deleted and then the `cookie` will be deleted lastly.

- `demo_session_id` as PRIMARY KEY
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

2. A row of demo_sessions and a `cookie` is generated or read by `DemoSessionHelper` through _any_ requests(GET, POST, DELETE, UPDATE).

    ```java
    // Inside ScheduleController
    // This snippet of code will be inside of all existing endpoints.
    @GetMapping("/{year}/{week}")
    public WeekDTO getWeek(@PathVariable int year,
                           @PathVariable int week,
                           HttpServletRequest request,
                           HttpServletResponse response
    ) {
        // extract the UUID received from the backend, if it doesn't exist...
        UUID demoSessionId = demoSessionHelper
                .readOrCreateDemoSessionID(request, response) // create it
                .orElseThrow(() -> new IllegalStateException(
                        "Could not resolve cookie"
                ));

        return scheduleService.getWeek(year, week, demoSessionId);
    }
    ```

3. If the `cookie` which was read from the request is invalid, null or expired then a new one will be created by `DemoSessionHelper`.

    ```java
    // this function is located inside DemoSessionHelper.
    public Optional<UUID> readOrCreateDemoSessionID(
        HttpServletRequest request, 
        HttpServletResponse response
    ) {
        Optional<UUID> demoSessionId = readCookie(request);

        DemoSession demoSessionRow = demoSessionId
                .flatMap(demoSessionRepository::findById)
                .orElse(null);

        // creates new a demo_sessions row if the cookie read from the request is NULL or EXPIRED.
        if (demoSessionRow == null || isExpired(demoSessionRow)) {
            return createDemoSessionRow(response);
        }

        return demoSessionId;
    }
    ```

    If the user happened to have saved shift-assignments before the expiry of their `cookie`, once it's expired, they won't see those shift-assignments anymore after they've sent another request, because they would've have already received their new `cookie` which refreshes everything.

4. Every hour an utility function is called to cleanup the demo_sessions' expired rows, **which will cascade delete its child tables**.

    ```java
    @Scheduled(fixedDelay = 3_600_000)
    @Transactional
    public void removeExpiredSession() {
        OffsetDateTime expirationTime = OffsetDateTime.now().minusMinutes(3);
        demoSessionRepository.deleteByCreatedAtBefore(expirationTime);
    }
    ```
    ```java
    // Don't forget to annotate the application runner with @EnableScheduling to run the method above
    @SpringBootApplication
    @EnableScheduling
    public class EmployeeSchedulerApplication {
        public static void main(String[] args) {
            SpringApplication.run(EmployeeSchedulerApplication.class, args);
        }
    }
    ```

5. For user experience I displayed a timer on the UI which counts down from the expiry timestamp of the cookie received from the backend:

    ```java
    private Optional<UUID> createDemoSessionRow(HttpServletResponse response) {

    UUID newDemoSessionId = UUID.randomUUID();

    // saves a new row of "demo_sessions"
    DemoSession newDemoSessionRow = new DemoSession();
    newDemoSessionRow.setId(newDemoSessionId);
    newDemoSessionRow.setCreatedAt(OffsetDateTime.now());
    newDemoSessionRow.setLastActiveAt(OffsetDateTime.now()); // unused but needed in the future
    demoSessionRepository.save(newDemoSessionRow);

    // sets the cookie
    setCookie(response, newDemoSessionId);

    // send expiration timestamp to the client (epoch millis)
    long expiresAtMillis = System.currentTimeMillis() + (MAX_AGE_SECONDS * 1000L);
    response.setHeader("X-Demo-Session-Expires-At", String.valueOf(expiresAtMillis));

    return Optional.of(newDemoSessionId);
    }
    ```
6. The header which will hold the expiry timestamp, will be received by the interceptor in angular:

    ```typescript
    // interceptor code here and it's called "DemoSessionInterceptor"
    ```

## Problems and solutions

NOTE: skip this part if you want to avoid headaches, this section is personal and intended for my learning.

1. **Problem**: cannot delete cookie rows from the database through requests. But scheduled/periodical cleanup utility which invokes the same function, works perfectly fine.

    ```java
    public Optional<UUID> readOrCreateDemoSessionID(
        HttpServletRequest request,
        HttpServletResponse response
    ) {
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

- 1.1. **Root cause**: In `readOrCreateDemoSessionID()`, DemoSession is loaded, then `deleteExpiredSession(...)` is also called...

    ```java
    // Loads demoSession
    DemoSession demoSessionRow = demoSessionId
                .flatMap(demoSessionRepository::findById)
                .orElse(null);

    // and then deletes it
    demoSessionUtil.deleteExpiredSession(demoSessionRow.getId());
    ```
    which deletes child rows through Spring Data derived methods and flushes each repository:

    ```java
    @Transactional
    public void deleteExpiredSession(UUID demoSessionId) {
        taskAssignmentRepository.deleteByDemoSessionId(demoSessionId);
        shiftRepository.deleteByDemoSessionId(demoSessionId);
        weekRepository.deleteByDemoSessionId(demoSessionId);

        taskAssignmentRepository.flush();
        shiftRepository.flush();
        weekRepository.flush();

        demoSessionRepository.deleteById(demoSessionId);
        demoSessionRepository.flush();
    }
    ```

    This can fail or behave inconsistently during a request because the entity/child rows may already be managed in the request’s persistence context, and the delete order must exactly satisfy every foreign key. 
    
    The scheduler works because it starts with no pre-loaded entities and runs the same deletion in an isolated scheduled execution.

- 1.2 **Solution 1**: Use `ON DELETE CASCADE` so deleting the demo_sessions row automatically removes dependent rows:

    ```sql
    -- V11__alter_dependent_to_demo_sessions_tables.sql
    ALTER TABLE task_assignments
    DROP CONSTRAINT IF EXISTS fk_task_assignments_demo_session,
    ADD CONSTRAINT fk_task_assignments_demo_session
    FOREIGN KEY (demo_session_id)
    REFERENCES demo_sessions(demo_session_id)
    ON DELETE CASCADE;

    -- do the same for shifts and weeks...
    ```

    Then simplify DemoSessionUtil to delete only the parent row:

    ```java
    @Transactional
    public void deleteExpiredSession(UUID demoSessionId) {
        demoSessionRepository.deleteById(demoSessionId);
    }
    ```

- 1.2 **Solution 2**: the first solution didnt work; the demo_sessions row still remained even after the simplification of the code. So the plan now **is to not delete the expired session during the request**. Keep the per-request expiration check only to decide whether to issue a new `cookie`, and let the scheduled cleanup remove the old database rows.

    - `Why this is better?`: The browser cookie expiration already prevents the visitor from continuing to use the old demo session: when they send the expired cookie, tje helper detects it, creates a fresh session, and returns a new cookie. The old `demo_sessions row` is then just temporary garbage. The scheduler already deletes it correctly, so synchronous deletion during the request adds complexity and a failure point without improving the user experience.

    - `Further simplification`: ON CASCADE DELETE solution was kept, and DemoSessionUtil is now removed:

    ```java
    @Scheduled(fixedDelay = 3_600_000)
    @Transactional
    public void removeExpiredSession() {
        OffsetDateTime expirationTime = OffsetDateTime.now().minusMinutes(30);
        demoSessionRepository.deleteByCreatedAtBefore(expirationTime);
    }
    ```

    - Now the deletion only happens inside this scheduled function. The rows that are expired (30 minutes after they were created) are removed from the table each hour.