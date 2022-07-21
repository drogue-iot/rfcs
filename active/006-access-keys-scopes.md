Add scopes to Access Keys


The current roles defined in drogue cloud have a permission granularity implemented in levels, where each level include all the permissions defined in the level above.

This prevents for being able to read devices data but not publish commands, and so on. 

One may want to give access to drogue cloud to a third party appto consume events without trusting it with devices credentials data.  

Here i propose to redefine the matrix of roles and permission, and allow a user to have more than one role. 

### Roles/Scopes and associated permissions

- Subscribe
  - can read events (WS/MQTT integration)
- Publish
  - can send commands to devices
- Read
  - Can read devices and applications resources
- Manage
  - Can create/read/modify devices and modify application.
- Admin
  - Can edit members of an app and delete the app

A user can have multiple roles for an application: 
```json
{
  "resourceVersion": "3e4f352f-a238-4233-ae3c-cd0daef235ff",
  "members": {
    "admins": [
        "ctron",
        "dejanb"
     ],
     "publisher": [
       "jbtrystram",
       "dejanb"
     ]
   }
}
```

### Access tokens

Users should be able to creates access tokens that have less 
permission than their owner have.
As it's not practical to maintain a reverse DB with which apps 
a user have roles to, access tokens
could contain claims.
The claims are checked against the application member list using
a AND operation, so one cannot use claims to escalate permissions.

## Discussion 

=> can anyone logged in create an app and be the owner ?
