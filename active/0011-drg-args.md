# Drg arguments rework

As part of my big drg rework (that will hopefully brings new great features) I want to simplify the cli structure.
There are a lot of subcommands which have sometimes inconsistencies between them.


## How it works now 

Currently drg have 14 subcommands: 

    admin      Manage application members and autorizations.
    cmd        Send a command to a device
    context    Manage contexts in the configuration file.
    create     create a resource in the drogue-cloud registry [aliases: add]
    delete     delete a resource in the drogue-cloud registry [aliases: remove]
    edit       Update a resource from the drogue-cloud registry [aliases: update]
    get        Display one or many resources from the drogue-cloud registry
    help       Prints this message or the help of the given subcommand(s)
    login      Log into a drogue cloud installation.
    set        Configure apps or devices resources
    stream     Stream application events
    trust      Manage trust-anchors and device certificates.
    version    Print version information.
    whoami     Print cluster adress, version and default app(if any)

## Proposition

I propose to move the config and admin subcommands under the `create/edit/get/delete` model.
Here is every operation drg support : 

| Command  | Resource        | Parameters                   | Options                              | Flags |
|----------|-----------------|------------------------------|--------------------------------------|-------|
|          |                 |                              |                                      |       |
| create   | device          | id                           | app, spec, file, gateway             | cert  |
|          | app             | id                           | spec, file                           |       |
|          | member          | id, role                     | app                                  |       |
|          | app-cert        | id                           | algo, days, key-input, key-output    |       |
|          | device-cert     | id                           | app, algo, days, ca-key, cert-output |       |
|          | token           |                              | description                          |       |
|          |                 |                              |                                      |       |
| edit     | device          | id                           | file                                 |       |
|          | app             | id                           | file                                 |       |
|          | member          | id                           | app                                  |       |
|          | context         | id                           |                                      |       |
|          |                 |                              |                                      |       |
| get      | device          | id (optional)                | app, label                           |       |
|          | app             | id (optional)                | label                                |       |
|          | members         |                              | app                                  |       |
|          | tokens          |                              |                                      |       |
|          | context         | id (optional)                |                                      |       |
|          |                 |                              |                                      |       |
| delete   | device          | id                           | app                                  |       |
|          | app             | id                           |                                      |       |
|          | member          | id                           | app                                  |       |
|          | token           | prefix                       |                                      |       |
|          | context         | id                           |                                      |       |
|          |                 |                              |                                      |       |
| transfer | init            | id, username                 |                                      |       |
|          | accept          | id                           |                                      |       |
|          | cancel          | id                           |                                      |       |
|          |                 |                              |                                      |       |
| set      | gateway         | id, id                       | app                                  |       |
|          | password        | id, password                 | username, app                        |       |
|          | alias           | id, alias                    | app                                  |       |
|          | default-app     | id                           |                                      |       |
|          | default-context | id                           |                                      |       |
|          | label           | id                           | device/app                           |       |
|          |                 |                              |                                      |       |
| stream   | id              |                              | count                                |       |
| login    | url             | token, access-token, context |                                      |       |
| version  |                 |                              |                                      |       |
| whoami   |                 |                              |                                      | token |


This would reduce the number of subcommands to 11. 
What i am not sure about is that it makes the distinction between local and remote operation unclear.
`drg edit config sandbox` and `drg set default-app myApp` are local actions,
but `drg edit device foo` and `drg set label --device myDevice foo=bar` is a remote operation.