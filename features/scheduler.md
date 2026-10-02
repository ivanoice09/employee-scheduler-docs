# Scheduler

When the app starts, a http GET request is sent to the server. Please see the details on how the app starts on `startup.md`.

```typescript
// this is the service method that does the GET request on the server
getWeek(year: number, weekNumber: number): Observable<WeekDTO> {
return this.http.get<WeekDTO>(`${this.baseUrl}/${year}/${weekNumber}`, {
    withCredentials: true,
});
}
```

The request is received by the ScheduleController:

```java
@GetMapping("/{year}/{week}")
public WeekDTO getWeek(@PathVariable int year, @PathVariable int week) {
    return scheduleService.getWeek(year, week);
}
```

And then processed by the SchduleService:

```java
@Transactional(readOnly = true)
public WeekDTO getWeek(int year, int weekNumber) {
    Week existingWeek = weekRepository.findByYearAndWeekNumber(year, weekNumber);

    if (existingWeek != null) {
        return buildWeekScheduleDTO(existingWeek);
    }

    return buildEmptyTemplate(year, weekNumber);
}
```

I have 2 method the returns different WeekDTO object to the client:

1. buildWeekScheduleDTO(year, weekNumber) is invoked when an existing year-weekNumber pair existss

