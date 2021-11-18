# Using usernames in the Drogue-cloud application Management API


The drogue cloud management API allows users to share their application to other users. 
One can allow read or write access to another user, and transfer application ownership to 
someone else. 

Here I propose a breaking change to this API tpo improve usability.

## Motivation

Currently, the API requires the application owner to input the keycloack ID of the user she/he wishes to share his app with. 
Theses are UUIDs used internally by keycloak. This requires asking a user his ID (exposed in the console) before sharing.


So the solution is to add a lookup between usernames and user IDs.

## Hiding the ID

Here is a payload example for the admin API. 

```json
{
  "resourceVersion": "ced63698-a0da-11eb-97e8-d45d6455d2cc",
  "members": {
    "jbtrystram": {
      "role": "admin"
    },
    "ctron": {
      "role": "reader"
    },
    "dejanb": {
      "role": "reader"
    }
  }
}
```

When processing the payload, the service does a lookup with each username and associate each user with his ID.
The usersnames are replaced with their IDs when writing the database. 


## Implementation

This was implemented in drogue cloud : https://github.com/drogue-iot/drogue-cloud/pull/122
