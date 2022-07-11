Add scopes to Access Keys


The current access keys defined in drogue cloud have the same permission granularity as their owner have. 
One may want to give access to drogue cloud to a third party app without trusting it with write access 
on all the other apps. 

So here is a proposed list of scopes that may be attached to an access key : 

Create / delete access tokens
Create / edit / delete applications

Specific per apps :
  Edit application members
  Create / edit / delete devices
  Read application data
  Read events 
  Write commands

For the per app scopes, we add a optional list of Application IDs to which those scopes apply.
Note : there is one list of apps per access key. If you want to have differents scopes for app A and app B then you should create 2 access keys. However you can create a key that allow only to read and write commands for apps A and B.

## Implementation 

During the authN process, the scopes for the token would be attached to the `UserInformation` object that is passed 
to the authZ service. The authZ service would then math the scopes to the current operations.

If an Oauth bearer token was used for authentication, we automatically match the "All" permission.
For the anonymous user, we only set the read only scopes
