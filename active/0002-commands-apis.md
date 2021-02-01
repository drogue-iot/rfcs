# Commands APIs

Define APIs that will allow IoT applications to send `commands` to devices.

## Commands 

IoT applications send commands to devices in order to modify their states. There are two general command types:

* One-way command - is a command sent by the IoT application for which it does not expect to receive any response if device received or executed a command
* Request-reply command - is a command sent by the IoT application whose execution must be acknowledged by the device. A solution will wait for a specified `timeout` for a response or assume that execution has failed. Note that this is not response that the infrastructure delivered a command to the device. A device must explicitly send a response by calling the appropriate API.

## IoT applications API

IoT applications use a `Command endpoint` to send commands to devices. Commands are targeted to a single device.

### Send one-way commands

* `POST /command?application=<application>&device=<device>&command=<command>`
  * Parameters:
    * Application ID: `<application>`
    * Device ID: `<device>`
    * Command: Command to execute
  * Body: Optional payload for the command

* `POST /command?application=<application>&device=<device>&command=<command>&timeout=0`
  * Parameters:
    * Application ID: `<application>`
    * Device ID: `<device>`
    * Command: Command to execute    
    * Timeout: Explicitly set timeout to 0
  * Body: Optional payload for the command  

### Send request-response commands

* `POST /command?application=<application>&device=<device>&command=<command>&timeout=<timeout>`
  * Parameters:
    * Application ID: `<application>`
    * Device ID: `<device>`
    * Command: Command to execute
    * Timeout: Set amount of time in seconds to wait for the command response from the device.
  * Body: Optional payload for the command

* HTTP Response
  * Headers:
    * Status: Status code of command execution
  * Body: Optional payload of the command response

## Devices API

Devices need to provide a way to receive commands and send responses back if needed. These mechanisms are specific depending on the endpoint device is using to connect to the cloud.

### HTTP

#### Receive commands

##### Receive commands as response to data publish

* `POST /<channel>?application=<application>&device=<device>&commandTimeout=<command-timeout>`
  * Parameters:
    * Application ID: `<application>`
    * Device ID: `<device>`
    * Command Timeout: Wait for commands for specified amount of time in seconds. Short alternative: `ct`

* HTTP Response
  * Headers:
    * Command: Command to be executed 
    * Request: Optional request identifier used to correlate response with.
  * Body: Optional payload for the command
  * Status:
    * 200 (OK) - in case data is successfully published and the response DOES CONTAIN command.
    * 202 (Accepted) - in case data is successfully published and the response DOES NOT CONTAIN command.
    * 406 (Not Acceptable) - in case data is not successfully published (the response DOES NOT CONTAIN the command in this case).
    

##### Explicitly wait for commands

* `GET /command/inbox?application=<application>&device=<device>&timeout=<timeout>`
  * Parameters:
    * Application ID: `<application>`
    * Device ID: `<device>`
    * Timeout: Wait for commands for specified amount of time in seconds.

* HTTP Response
  * Headers:
    * Command: Command to be executed 
    * Request: Optional request identifier used to correlate response with.
  * Body: Optional payload for the command  

#### Command responses

* `POST /command/outbox?application=<application>&device=<device>&request=<request>&status=<status>`
  * Parameters:
    * Application ID: `<application>`
    * Device ID: `<device>`
    * Request: Request identifier used to correlate response with.
    * Status: Status code of commands execution.
  * Body: Optional payload for the command response  


### MQTT

#### Receive commands

* `CONNECT` with credentials for desired application and device
* `SUBSCRIBE` to `command/inbox` topic
* Receive `PUBLISH` with `command/outbox/<command>/<request>` topic and command payload
  * Command: Command to execute
  * Request: Request identifier used to correlate response with. 
  * Body: Optional payload for the command response  

### Command responses

* `PUBLISH` to `command/outbox/<request>/<status>`
  * Request: Request identifier used to correlate response with.
  * Status: Status code of command execution  
  * Body: Optional payload for the command