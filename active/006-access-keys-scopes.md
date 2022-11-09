Add scopes to Roles 

# Current limitation

The current roles defined in drogue cloud have a permission granularity implemented in levels, where each level include all the permissions defined in the level above.

This prevents for being able to read devices data but not publish commands, and so on. 

One may want to give access to drogue cloud to a third party application (through an access token) to consume events without trusting it with devices credentials data for example.

Here I propose to redefine the matrix of roles and permission, and allow a user to have more than one role.
This would also remove the `owner` field from apps, as it would be saved in the `members`.

### Roles/Scopes and associated permissions

- Subscribe: read events (WS/MQTT integration)
- Command: send commands to devices
- Read: read devices and applications resources
- Manage: create/read/update devices and read/update application.
- Admin: edit members of an app and delete the app
- Owner: all the above

Then there are a couple of roles that are not tied to a specific application :
- Create: create applications
- Tokens
  - create
  - read
  - delete

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
       "admin",
       "manage"
     ], 
    "dejanb": [
      "subscribe"
    ], 
    "jcrossley": [
      "owner"
    ]
   }
}
```

### Access tokens

Users should be able to creates access tokens that have less 
permission than their owner have.
When creating an access token, the user can select permissions for applications. 
When authenticating (and on token creation), the token claims are checked against the application member list using
a AND operation, so one cannot use claims to escalate permissions.

Access token proposed structure
```yaml
description: String
scopes:
  create: boolean # allow to create tokens or not
  applications:
    - example-app:
        - subscribe
        - read
        - command
        - manage
        - admin
    - eclipsecon-hackathon:
        - publish
        - subscribe
    - myapp:
        - owner
  tokens: # allow access token management.
    - create
    - read
    - delete
```

The `scope` field can directly be deserialised as a rust Struct and be included with `UserInformation` when querying the authentication service.
Services can then use this info to allow/deny the request.

### Create application with bearer Oauth token
OAuth tokens bearers are always allowed to create applications.

