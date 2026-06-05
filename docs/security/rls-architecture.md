# RLS Architecture

## RLS Philosophy
PostgreSQL Row Level Security (RLS) is the absolute source of truth for authorization. No client or intermediate server is trusted to make security decisions.

## Trust Model
We operate on a **Zero Trust** internal model. An authenticated JWT simply proves *identity*. *Authorization* is derived entirely from the relational data (`club_memberships`) at the exact moment a query executes.

## Policy Design Principles
1. **Minimize Subqueries**: RLS policies execute on every row. `EXISTS` subqueries must be highly optimized and heavily indexed.
2. **Explicit Grants**: Policies must explicitly `GRANT` access. Default state is `DENY`.

## Security Invoker Views
Views used to filter records execute with the permissions of the calling user (`security_invoker = true`), meaning RLS cascades securely through views.

## RLS Lifecycle
Policies are defined in standard SQL migrations and managed entirely via source control.\n