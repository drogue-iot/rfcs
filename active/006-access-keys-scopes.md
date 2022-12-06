# Current limitations

The current roles defined in drogue cloud have a permission granularity implemented in levels, where each level include all the permissions defined in the level above.

This prevents for being able to read devices data but not publish commands, and so on. 

One may want to give access to drogue cloud to a third party application (through an access token) to consume events without trusting it with devices credentials data for example.

Here I propose to redefine the matrix of roles and permission, and allow a user to have more than one role.

### Roles/Scopes and associated permissions

- Subscribe: read events (WS/MQTT integration)
- Publish: send commands to devices
- Read: read devices and applications resources
- Manage: create/read/update devices and read/update application.
- Admin: All, excluding transferring the app ownership and deleting an app.

### Permission table

|                  |           | **Admin** | **Manager** | **Reader** | **Subscriber** | **Publisher** | **Owner** |
|------------------|-----------|-----------|-------------|------------|----------------|---------------|-----------|
| **Applications** | Delete    |           |             |            |                |               | OK        |
|                  | Read      | OK        | OK          | OK         |                |               | OK        |
|                  | Write     | OK        | OK          |            |                |               | OK        |
|                  | Members   | OK        |             |            |                |               | OK        |
|                  | Subscribe | OK        |             |            | OK             |               | OK        |
|                  | Command   | OK        |             |            |                | OK            | OK        |
|                  | Transfer  |           |             |            |                |               | OK        |
| **Devices**      | Create    | OK        | OK          |            |                |               | OK        |
|                  | Delete    | OK        | OK          |            |                |               | OK        |
|                  | Write     | OK        | OK          |            |                |               | OK        |
|                  | Read      | OK        | OK          | OK         |                |               | OK        |

A user can have multiple roles for an application:
```json
{
  "resourceVersion": "3e4f352f-a238-4233-ae3c-cd0daef235ff",
  "members": {
    "ctron": [
        "read",
        "publish",
        "subscribe"
     ],
     "jbtrystram": [
       "admin"
     ], 
    "dejanb": [
      "subscribe"
    ]
   }
}
```

Then there are a few of roles that are not tied to a specific application :
- Create: create applications
- Tokens
  - create
  - read
  - delete

For those I propose to always allow when authenticating with an Oauth Token and limit them when using PATs.

### Personnal Access tokens

Users should be able to create access tokens that have less permission than their owner have.
When creating an access token, the user can select permissions claims.
As the PAT service does not have a postgreSQL client it cannot verifies the permissions so these are just claims.

Access token proposed structure :
```yaml
description: String
claims:
  create: boolean # allow to create applications
  applications:
    - example-app:
        - admin
    - eclipsecon-hackathon:
        - publish
        - subscribe
    - myapp:
        - read
  tokens: # allow access token management.
    - create
    - read
    - delete
```

NOTE : it should use application IDs instead of names. To avoid
getting access to a resource after it was created.

### Verification flow

```
         ┌───────────────────┐
         │                   │
      2  │      KeyCloak     │
         │                   │
         └───▲────────┬──────┘
             │        │
             │        │ User Information
             │        │
          PAT│        │ Some(Claims)                
             │        │
             │        │                        2
           ┌─┴────────▼───────┐  User     ┌────────────────┐
           │                  ├───────────►                │
           │                  │           │                │
           │  Authorization   │           │PostgreSQL      │
           │                  │           │                │
           │                  ◄───────────┤                │
           └──────────────────┘  Roles    └────────────────┘

            Roles AND Claims
     3             =
                 Outcome
```

When authenticating, the token claims are retrieved then checked against the application member list using
a AND operation, so one cannot escalate permissions.
The `claims` field can directly be deserialised as a rust Struct and be included with `UserInformation` when querying keycloak.
Services can then use this info to allow/deny the request.

