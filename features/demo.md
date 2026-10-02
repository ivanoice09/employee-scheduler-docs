# Demo

Allow visitors to add shifts on employees on a specific week, after adding it they can edit or delete it. Their access are session-based and will expire after 30 minutes (3 mins for my tests), once expired their newly added shifts will be deleted.

To make the demo beginner friendly this app won't have any authentication, they will be able to test the app right away.

## Implementation

1. I created a new Entity called "DemoSession" and a database table called "demo_sessions".

2. A new session will be stored inside "demo_sessions" on the first request. The request is when saving a new "Week":

```Java
@PostMapping("/save")
public void saveWeek(@RequestBody SaveWeekDTO dto) {
    scheduleService.saveWeek(dto);
}
```

3. This new row will have its "created_at" field which will track the expiration time.
