# RFC - Application and API Keys management

## Overview

In Drogue cloud, Applications scope all elements that belong together, like devices, data, and integrations. In recent improvements to DRG - we are using applications as a root of trust to authenticate devices. 

Drogue Cloud also allows an application to have multiple owners, and different people to have different roles in the same application. Here, we try to document how drg can be used for this purpose.

## Commands

We love to group commands based on their functionality so let's keep the subcommand for all the administration work to - drg `adm`. 

### Adding members

`drg adm add --app <app_id> <username> --roles <reader/manager/admin>`

This command helps assign roles to another user. Here we would need to pass in the user id of the other user and the permission we would like to give him.

`drg adm list --app <app_id>`

This command prints all the members with access to this application and their roles.

### API Keys management

`drg adm keys show` -> Displays the list of API keys.

`drg adm keys create` -> Creates a new key for the user.

`drg adm keys delete <prefix>` -> Deletes the API Key by the prefix.

These commands are similar to the subcommands offered in `context` functionality.

### Transfer ownership 

`drg adm transfer --app <app_id> <user_id>`

This initiates the process of transferring an application. 


`drg adm transfer --app <app_id> -d`

By adding an -d or --delete flag we can cancel the ownership transfer process.


`drg adm transfer --app <app_id> -a`

By adding an -a or --accept flag, the user can accept the ownership of an application.

## Implementation

Drogue cloud API already has API implemented which can be consumed in drg to create the above-mentioned functionality.