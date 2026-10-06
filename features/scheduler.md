# Scheduler

Sections:

1. Retreiving shift-assignments
2. Saving and editing shift-assignments

## Retreiving shift-assignments

### 1. When the app starts, a http GET request is sent to the server. Please see the details on how the app starts on `startup.md`.

```typescript
// this is the service method that does the GET request on the server
getWeek(year: number, weekNumber: number): Observable<WeekDTO> {
    return this.http.get<WeekDTO>(`${this.baseUrl}/${year}/${weekNumber}`, {
        withCredentials: true,
    });
}
```

### 2. The request is received by the ScheduleController:

```java
@GetMapping("/{year}/{week}")
public WeekDTO getWeek(@PathVariable int year, @PathVariable int week) {
    return scheduleService.getWeek(year, week);
}
```

### 3. And then processed by the ScheduleService:

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

### 4. I have 2 method the returns different WeekDTO objects to the client:

- **buildWeekScheduleDTO(`week`)** is invoked when a year-weekNumber pair exists in the database. It returns a WeekDTO object:

    ```json
    // WeekDTO payload response example:
    {
        
        "assignments": [
            {
                "employeeId": 1,
                "shifts": [
                    {
                        "actualDate": "2026-10-05",
                        "endsAt": "16:00:00",
                        "startsAt": "08:00:00"
                    }
                ]
            },
            // other employeeId's here with their shifts...
        ],
        "employees": [
            {
                "employeeId": 1,
                "firstName": "Aaron",
                "lastName": "Adams",
                "middleName": "Ash"
            },
            // other employees here...
        ],
        "existingWeek": true,
        "startDate": "2026-10-05",
        "status": "DRAFT",
        "weekNumber": 41,
        "year": 2026
        
    }
    ```
    ```java
    // Full code
    private WeekDTO buildWeekScheduleDTO(Week week) {
        List<EmployeeDTO> employees = employeeService.getAllEmployees();

        List<Shift> shifts = shiftRepository
                .findByWeekId(week.getId());

        Map<Long, List<Shift>> shiftsByEmployeeId = shifts.stream()
                .collect(Collectors.groupingBy(shift -> shift.getEmployee().getId()));

        List<ShiftAssignmentDTO> assignments = employees.stream()
                .map(employee -> {
                    ShiftAssignmentDTO dto = new ShiftAssignmentDTO();
                    dto.setEmployeeId(employee.getEmployeeId());

                    List<ShiftDTO> employeeShifts = shiftsByEmployeeId
                            .getOrDefault(employee.getEmployeeId(), List.of())
                            .stream()
                            .map(this::mapToShiftAssignmentDTO)
                            .toList();

                    dto.setShifts(employeeShifts);
                    return dto;
                })
                .toList();

        WeekDTO dto = new WeekDTO();
        dto.setYear(week.getYear());
        dto.setWeekNumber(week.getWeekNumber());
        dto.setStartDate(week.getWeekStartDate());
        dto.setStatus(week.getStatus());
        dto.setEmployees(employeeService.getAllEmployees());
        dto.setAssignments(assignments);
        dto.setExistingWeek(true);

        return dto;
    }
    ```

- **buildEmptyTemplate(`year`, `weekNumber`)** is invoked when a year-weekNumber pair DOESN'T exist, and returns a WeekDTO with empty shift-assignments:

    ```json
    // WeekDTO with empty shift-assignments payload response example:
    {
        "assignments": [],
        "employees": [
            {
                "employeeId": 1,
                "firstName": "Aaron",
                "lastName": "Adams",
                "middleName": "Ash"
            },
            // other employees here...
        ],
        "existingWeek": false,
        "startDate": "2026-10-12",
        "status": null,
        "weekNumber": 42,
        "year": 2026
    }
    ```
    ```java
    // Full code
    private WeekDTO buildEmptyTemplate(int year, int weekNumber) {
        LocalDate weekStart = WeekUtil.getStartOfIsoWeek(year, weekNumber);

        WeekDTO dto = new WeekDTO();
        dto.setYear(year);
        dto.setWeekNumber(weekNumber);
        dto.setStartDate(weekStart);
        dto.setStatus(null);
        dto.setEmployees(employeeService.getAllEmployees());
        dto.setAssignments(List.of());
        dto.setExistingWeek(false);

        return dto;
    }
    ```

### 5. Inside Schedule.ts is located the OnInit() method which invoke the getWeek() from step #1:

```typescript
ngOnInit(): void {
this.route.paramMap
    .pipe(
        switchMap((params) => {
            const year = Number(params.get('year'));
            const week = Number(params.get('week'));
            return this.scheduleService.getWeek(year, week); // right here.
        }),
    )
    .subscribe((week) => {
        this.weekSubject.next(week);
        
        // other code here that seeds the UI with the fetched data...
    });

    // NOTE: again, if you haven't already, please read the details on how the startup works in startup.md
}
```

## Saving and editing shift-assignments

### 1. Once the user has added a shift at least on one of the shift-cells they're now able to save it to the backend through this form template:

```html
<form [formGroup]="form" (ngSubmit)="save()">

    <div class="schedule-grid">
        <!-- Header row -->
        <div class="header-cell employee-header">
            <span>Employees</span>
        </div>

        @for (dayName of dayNames; track $index) {
        <div class="header-cell specific-date">
            <span>{{ dayName | slice:0:3 | uppercase }}</span>
            <span>
                {{ getDateForDay(week.startDate, $index) | date: 'dd' }}
            </span>
        </div>
        }

        <!-- Employee rows -->
        <ng-container formArrayName="assignments">
            @for (assignment of assignmentsFormArray.controls; track $index; let employeeIndex = $index) {
                <ng-container [formGroupName]="employeeIndex">
                    <div class="employee-cell">
                        <span>{{ assignment.get('employeeName')?.value }}</span>
                    </div>

                    <ng-container formArrayName="shifts">
                        @for (
                            shift of getShifts(employeeIndex).controls;
                            track $index; 
                            let dayIndex = $index
                        ) {
                            
                            <div class="shift-cell">
                                @if (viewMode === 'edit') {
                                    <div [formGroupName]="dayIndex" class="shift-editor">
                                        <input 
                                            type="text"
                                            inputmode="numeric"
                                            maxlength="2"
                                            pattern="[0-9]{0,2}"
                                            placeholder="HH"
                                            formControlName="startHour"
                                            data-time-type="hour"
                                            (blur)="formatTimeInputOnBlur($event)"
                                            (focus)="openTimePicker(employeeIndex, dayIndex, 'startHour')"
                                        />

                                        <span>:</span>

                                        <input
                                            type="text"
                                            inputmode="numeric"
                                            maxlength="2"
                                            pattern="[0-9]{0,2}"
                                            placeholder="MM"
                                            formControlName="startMinute"
                                            data-time-type="minute"
                                            (blur)="formatTimeInputOnBlur($event)"
                                            (focus)="openTimePicker(employeeIndex, dayIndex, 'startMinute')"
                                        />

                                        <span>-</span>

                                        <input
                                            type="text"
                                            inputmode="numeric"
                                            maxlength="2"
                                            pattern="[0-9]{0,2}"
                                            placeholder="HH"
                                            formControlName="endHour"
                                            data-time-type="hour"
                                            (blur)="formatTimeInputOnBlur($event)"
                                            (focus)="openTimePicker(employeeIndex, dayIndex, 'endHour')"
                                        />

                                        <span>:</span>

                                        <input
                                            type="text"
                                            inputmode="numeric"
                                            maxlength="2"
                                            pattern="[0-9]{0,2}"
                                            placeholder="MM"
                                            formControlName="endMinute"
                                            data-time-type="minute"
                                            (blur)="formatTimeInputOnBlur($event)"
                                            (focus)="openTimePicker(employeeIndex, dayIndex, 'endMinute')"
                                        />
                                    </div>
                                } @else {
                                    <span class="shift-text">
                                        {{ formatShiftText(employeeIndex, dayIndex) }}
                                    </span>
                                }
                            </div>
                        }
                    </ng-container>
                </ng-container>
            }
        </ng-container>
    </div>
        <button class="save-button" type="submit" [class.hidden]="viewMode === 'read'">Save</button>
</form>
```

### 2. When the save button is clicked, this function is invoked:

```typescript
save(): void {
    this.blurAllTimeInputs();

    const raw = this.form.getRawValue();

    // payload is built here through the form template
    const payload: SaveWeekDTO = {
        year: raw.year ?? 0,
        weekNumber: raw.weekNumber ?? 0,
        weekStartDate: this.currentWeekStartDate,
        assignments: (raw.assignments ?? []).map((assignment: any) => {
            const shifts = (assignment.shifts ?? [])
                .map((shift: any) => {
                    const startsAt = this.toTime(shift.startHour, shift.startMinute);
                    const endsAt = this.toTime(shift.endHour, shift.endMinute);

                    if (!startsAt || !endsAt) {
                        return null;
                    }

                    return {
                        actualDate: shift.actualDate,
                        startsAt,
                        endsAt,
                    };
                })
                .filter((s: any) => s !== null);

            return {
                employeeId: assignment.employeeId,
                shifts,
            };
        }),
    };

    console.log('Sending payload:', payload);

    // and then the http call sends the payload to the backend
    this.scheduleService.saveWeek(payload).subscribe({
        next: () => console.log('Saved'),
        error: (err) => console.error(err),
    });
}
```

### 3. This is the service method that saves the shift-assignments:

```typescript
saveWeek(payload: SaveWeekDTO): Observable<void> {
  return this.http.post<void>(`${this.baseUrl}/save`, payload, { withCredentials: true });
}
```

### 4. ScheduleController saveWeek() endpoint:

```java
@PostMapping("/save")
public void saveWeek(@RequestBody SaveWeekDTO dto) {
    scheduleService.saveWeek(dto);
}
```

### 5. ScheduleService saveWeek() data process method:

```java
@Transactional
public void saveWeek(SaveWeekDTO dto) {

    DemoSession demoSession = demoSessionRepository
                .findById(demoSessionId)
                .orElseThrow(() -> new IllegalStateException(
                        "Demo session not found or invalid"
                ));

    Week week = weekRepository
            .findByYearAndWeekNumber(
                    dto.getYear(),
                    dto.getWeekNumber()
            )
            .orElseGet(() -> {
                Week newWeek = new Week();
                newWeek.setYear(dto.getYear());
                newWeek.setWeekNumber(dto.getWeekNumber());
                newWeek.setWeekStartDate(dto.getWeekStartDate());
                newWeek.setStatus(Week.Status.DRAFT);
                newWeek.setDemoSession(demoSession);
                return weekRepository.save(newWeek);
            });
    week.setWeekStartDate(dto.getWeekStartDate());
    for (ShiftAssignmentDTO assignmentDTO : dto.getAssignments()) {
        Employee employee = employeeRepository
                .findById(assignmentDTO.getEmployeeId())
                .orElseThrow(() -> new IllegalArgumentException(
                        "Employee not found: " + assignmentDTO.getEmployeeId()
                ));
        for (ShiftDTO shiftDTO : assignmentDTO.getShifts()) {
            saveOrUpdateAssignment(week, employee, shiftDTO);
        }
    }
}
```
```java
// An overwrite method. When there are already existing shifts, and the user saves it, it overwrites the previous.
private void saveOrUpdateAssignment(
        Week week,
        Employee employee,
        ShiftDTO dto
) {
    Optional<Shift> existingShiftOpt = shiftRepository
            .findByWeekAndEmployeeAndActualDate(
                    week,
                    employee,
                    dto.getActualDate()
            );

    boolean emptyShift = dto.getStartsAt() == null || dto.getEndsAt() == null;

    if (emptyShift) {
        existingShiftOpt.ifPresent(shiftRepository::delete);
        return;
    }

    if (existingShiftOpt.isPresent()) {
        Shift existinShift = existingShiftOpt.get();
        existinShift.setStartsAt(dto.getStartsAt());
        existinShift.setEndsAt(dto.getEndsAt());
        return;
    }

    Shift newShift = new Shift();
    newShift.setActualDate(dto.getActualDate());
    newShift.setStartsAt(dto.getStartsAt());
    newShift.setEndsAt(dto.getEndsAt());
    newShift.setEmployee(employee);
    newShift.setWeek(week);
    newShift.setDemoSession(demoSession);

    shiftRepository.save(newShift);
}
```