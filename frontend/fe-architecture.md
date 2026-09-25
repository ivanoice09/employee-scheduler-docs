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

1. After login, the user will be redirected to the `schedule` component which has a parameterized URL, such as: `http://localhost:4200/schedule/{year}/{week}`. But it is not possible to be redirected **directly** to a parameterized URL right after logging in, because you need to pass the arguments first.

2. Before passing those arguments, I made a **Redirect Component** (located inside `schedule.ts`) which only purpose is to trigger a specific **guard**.

    ```typescript
    @Component({
    template: '',
    })
    export class RedirectComponent {}
    ```

3. This **guard** will pass those arguments, by utilizing **week-util** service functions.

- `current-week-redirect-guard.ts`. The arguments which will be passed depend on the current week number:

    ```typescript
    export const currentWeekRedirectGuard: CanActivateFn = () => {
    const router = inject(Router);
    const weekUtil = inject(WeekUtil);

    const year = weekUtil.getCurrentYear();
    const week = weekUtil.getCurrentWeek();

    return router.createUrlTree(['/schedule', year, week]);     // Now the schedule component route have the arguments.
    };
    ```

- the guard is used by this specific route. Code is located inside `app.routes.ts`.

    ```typescript
    { 
        path: '', 
        component: RedirectComponent,
        canActivate: [currentWeekRedirectGuard],    // Once sent to this redirect component, the guard activates.
        pathMatch: 'full',
    }
    ```

- `week-util.ts`. The functions that calculate the current week number and year:

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

- Routing flow diagram:

    ```text
    login -> redirect component -> guard -> schedule component
    ```