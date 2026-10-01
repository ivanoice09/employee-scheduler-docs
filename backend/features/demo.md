# Demo
Allow visitors to have their own sandbox: they can edit, create and delete data. Their access are session-based and will expire after 30 minutes (3 mins for my tests). Once expired data deletion will be executed.

## Implementation plan

1. Create a new Entity(DemoSession) and Database Table(demo_sessions).

```sql
-- migration file:

```
