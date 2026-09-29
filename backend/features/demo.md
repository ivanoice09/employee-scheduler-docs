# Demo Implementation
Allow visitors to have their own sandbox: they can edit, create and delete data. Their access are session-based and will expire after 30 minutes which will remove all the actions they've done in the sandbox.

## Version 1
- I will use the UUID in cookies and not store it anywhere.
- Employees are global and NOT per-session means they won't be altered by visitors.

