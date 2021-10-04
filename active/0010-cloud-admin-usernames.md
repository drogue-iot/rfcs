# Using usernames in the Drogue-cloud application Management API


The drogue cloud management API allows users to share their application to other users. 
One can allow read or write access to another user, and transfer application ownership to 
someone else. 

Here I propose a breaking change to this API tpo improve usability.


## Motivation

Currently, the API requires the application owner to input the keycloack ID of the user she/he wishes to share his app with. 
Theses are UUIDs used internally by keycloak. This requires asking a user his ID (exposed in the console) before sharing.

This makes it quite a cumbersome process. However the usual username cannot be used instead because this will result 
in the username reused issue :
 > You share an app with bob. Then bob deletes his account. Then someone else create a bob account
 -> new bob has access to your apps.

So the solution is to add a lookup between usernames and user IDs.

Here are the two solutions I came up with : 

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

There is one issue with that : \
If ctron deletes his account, and another ctron creates an account, the next time I make a change to my app, the new user will receive access.
Solving this requires exposing the ids. 

## Exposing the IDs

The admin service still expose the user ids but the property is optional.
Here I replace the user "dejanb" with "jcrossley" as a reader for my app : 
```json
{
 "resourceVersion": "ced63698-a0da-11eb-97e8-d45d6455d2cc",
 "members": {
    "jbtrystram": {
       "role": "admin",
       "id": "d84eb308-a0da-11eb-9e90-d45d6455d2cc"
  },
    "ctron": {
       "role": "reader", 
       "id": "03e32c1a-a0db-11eb-9e9b-d45d6455d2cc"
  },
    "jcrossley": {
       "role": "reader"
  }
 }
}
```

In that case entries containing a UUID will be checked against keycloak to make sure they exist,
and entries without one will be populated with the adequate UUID.


The benefit of this change (either one) is that we make sure that the user don't share apps with someone that does not exist.
