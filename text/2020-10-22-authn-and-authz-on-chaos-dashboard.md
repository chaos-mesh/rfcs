# Authentication and authorization on chaos dashboard

## Summary

We need using authentication and authorization to restrict user's permission and
action.

## Motivation

Users could create, suspend, get, and delete chaos experiments via chaos-dashboard.
But chaos-dashboard does NOT contains and features about permission management, users
could do anything, as long as he could access the chaos-dashboard UI. It is a secure
issue.

We need permission management about:

- access control for **Resource**.
  - User A could create/get IO chaos experiments.
  - User B could create/get Network chaos experiments.
- access control for **Action**.
  - User A could create/get chaos experiments.
  - User B could only get chaos experiments.

## Detailed design

We want chaos-dashboard working like kubernetes-dashboard: it ask user
for a **Service Account token** to login.

Here is unfinished works we need to do:

- frontend asking user input token to login
- frontend will attach the token while sending requests to backend
- backend will use a certain token to create a new kube client
- backend need support multi-user

For users, administrators could create

We will provide some pre-set **Role**, like:

- Admin: could create/get any chaos experiments.
- Viewer: could only get chaos experiments.

System administrators could use role chaos-dashboard provided to create service
accounts which hold different permissions. System administrators could also create
roles, for more advanced permission control.

> More implementation detail required.

## Drawbacks

- This solution need chaos-dashboard has permission about create/get/update/delete
  on Role/RoleBinding/Service Account/Secrets.
- Users should understand basic concepts about kubernetes rbac.

## Alternatives

- Just using kubernetes rbac, reduce many logics for permission management.
- System administrators could change each user's permission as their requirements,
  it's very flexible.
- Using kubernetes rbac could also restrict permission with `kubectl`.

## Unresolved questions

None.
