# Frontend Architecture
This app primarily uses framework angular.

## Folder layout
```text
.
└── \src\app/
    ├── core/
    │   └── layout/
    │       └── navbar
    ├── features/
    │   └── schedule
    ├── guard
    ├── shared/
    │   ├── DTO
    │   └── services
    └── utils
```

## Features

- **Scheduler** - the app's **homepage** located inside the `schedule` component folder.

## Routing

1. Upon startup the user will be redirected to the `schedule` component which has a parameterized URL, such as: `http://localhost:4200/schedule/{year}/{week}`. But it is not possible to be redirected **directly** to a parameterized URL right after logging in, because you need to pass the *arguments* first.

2. Before passing those *arguments*, I made `RedirectComponent` (located inside `schedule.ts`) which only purpose is to activate `current-week-redirect-guard.ts`. This is technically where the client ends up upon starting the app, not on schedule component.

    ```typescript
    @Component({
    template: '',
    })
    export class RedirectComponent {}
    ```

3. `current-week-redirect-guard.ts` is activated by `RedirectComponent`'s path which is located inside `app.routes.ts`.

    ```typescript
    { 
        path: '', 
        component: RedirectComponent,
        canActivate: [currentWeekRedirectGuard], 
        pathMatch: 'full',
    }
    ```

4. `current-week-redirect-guard.ts` will generate the values for the current year and weekNumber by utilizing `week-util.ts` functions. And then these values will be the *arugments* passed on the schedule parameterized URL.

    ```typescript
    export const currentWeekRedirectGuard: CanActivateFn = () => {
    const router = inject(Router);
    const weekUtil = inject(WeekUtil);

    const year = weekUtil.getCurrentYear();
    const week = weekUtil.getCurrentWeek();

    return router.createUrlTree(['/schedule', year, week]);     // Now the schedule component route have the arguments.
    };
    ```
    ```typescript
    // currentWeekRedirectGuard invokes this path through:
    // return router.createUrlTree(['/schedule', year, week]);
    { 
        path: 'schedule/:year/:week',
        component: Schedule,
    }
    ```

- `week-util.ts`. The utility functions that calculate the current week number and year:

    ```typescript
    getCurrentYear(): number {
        return new Date().getFullYear();
    }

    getCurrentWeek(): number {
        const date = new Date();
        const target = new Date(Date.UTC(date.getFullYear(), date.getMonth(), date.getDate()));
        const dayNum = target.getUTCDay() || 7;
        target.setUTCDate(target.getUTCDate() + 4 - dayNum);
        const yearStart = new Date(Date.UTC(target.getUTCFullYear(), 0, 1));
        return Math.ceil(((target.getTime() - yearStart.getTime()) / 86400000 + 1) / 7);
    }
    ```

5. And finally, `Schedule.ts`'s OnInit() method will just GET those year and week values directly from the URL route:

    ```typescript
    ngOnInit(): void {
        this.route.paramMap
      .pipe(
        switchMap((params) => {
          const year = Number(params.get('year'));
          const week = Number(params.get('week'));
          return this.scheduleService.getWeek(year, week);
        }),
      )
      .subscribe((week) => {
        this.weekSubject.next(week);
        this.currentWeekStartDate = week.startDate;

        // more code here...

      });
    }
    ```

- Routing flow diagram:

    ```text
    login -> redirect-component -> guard -> schedule-component
    ```