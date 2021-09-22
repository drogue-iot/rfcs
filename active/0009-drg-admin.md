# RFC - Application and API Keys management

## Overview

In Drogue cloud, Applications scope all elements that belong together, like devices, data, and integrations. In recent improvements to DRG - we are using applications as a root of trust to authenticate devices. 

Drogue Cloud also allows an application to have multiple owners, and different people to have different roles in the same application. Here, we try to document how drg can be used for this purpose.

## Commands

We love to group commands based on their functionality so let's keep the subcommand for all the administration work to - drg `admin`. 

### Adding members

`drg admin member add --app <app_id> <username> --roles <reader/manager/admin>`

This command helps assign roles to another user. Here we would need to pass in the user id of the other user and the permission we would like to give him.

`drg admin member list --app <app_id>`

This command prints all the members with access to this application and their roles. The output of the above commands would be 
|User | Roles |
|-----|-------|
|abc  | Manager | 
|def  | Reader |

`drg admin member edit --app <app_id>`

Similar to 'device' edit, member information can be edited in YAML.

### API Keys management

`drg admin keys show` -> Displays the list of API keys.

`drg admin keys create` -> Creates a new key for the user.

`drg admin keys delete <prefix>` -> Deletes the API Key by the prefix.

These commands are similar to the subcommands offered in `context` functionality.

### Transfer ownership 

`drg admin transfer start <app_id> <username>`

This initiates the process of transferring an application. 


`drg admin transfer delete <app_id>`

Users can cancel the ownership transfer process.


`drg admin transfer accept <app_id>`

The user can accept the ownership of an application.

## Implementation

Drogue cloud API already has API implemented which can be consumed in drg to create the above-mentioned functionality.