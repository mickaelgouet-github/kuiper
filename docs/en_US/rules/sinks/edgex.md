# EdgeX Message Bus action

The action is used for publishing output message into EdgeX message bus.  

**Please notice that, if you're using the ZeorMQ message bus, the action will create a NEW EdgeX message bus (with the address where running Kuiper service), but not by leveraging the original message bus (normally it's the address & port exposed by application service). **

**Also, you need to expose the port number to host server before running the Kuiper server if you want to have the service available to other hosts.**

| Property name | Optional | Description                                                  |
| ------------- | -------- | ------------------------------------------------------------ |
| protocol      | true     | The protocol. If it's not specified, then use default value ``tcp``. |
| host          | true     | The host of message bus. If not specified, then use default value ``*``. |
| port          | true     | The port of message bus. If not specified, then use default value ``5563``. |
| topic         | true     | The topic to be published. If not specified, then use default value ``events``. |
| contentType   | true     | The content type of message to be published. If not specified, then use the default value ``application/json``. |
| metadata      | true     | The property is a field name that allows user to specify a field name of SQL  select clause,  the field name should use ``meta(*) AS xxx``  to select all of EdgeX metadata from message. |
| deviceName    | true     | Allows user to specify the device name in the event structure that are sent from Kuiper. |
| type          | true     | The message bus type, two types of message buses are supported, ``zero`` or ``mqtt``, and ``zero`` is the default value. |
| optional      | true     | If ``mqtt`` message bus type is specified, then some optional values can be specified. Please refer to below for supported optional supported configurations. |

Below optional configurations are supported, please check MQTT specification for the detailed information.

- optional
  - ClientId
  - Username
  - Password
  - Qos
  - KeepAlive
  - Retained
  - ConnectionPayload
  - CertFile
  - KeyFile
  - CertPEMBlock
  - KeyPEMBlock
  - SkipCertVerify

## Examples

### Publish result to a new EdgeX message bus without keeping original metadata
In this case, the original metadata value (such as ``id, pushed, created, modified, origin`` in ``Events`` structure, and ``id, created, modified, origin, pushed, device`` in ``Reading`` structure will not be kept). Kuiper acts as another EdgeX micro service here, and it has own ``device name``. A ``deviceName`` property is provided, and allows user to specify the device name of Kuiper. Below is one example,

1) Data received from EdgeX message bus ``events`` topic,
```
{
  "Device": "demo", "Created": 000, …
  "readings": 
  [
     {"Name": "Temperature", value: "30", "Created":123 …},
     {"Name": "Humidity", value: "20", "Created":456 …}
  ]
}
```
2) Use following rule,  and specify ``deviceName`` with ``kuiper`` in ``edgex`` action.

```json
{
  "id": "rule1",
  "sql": "SELECT temperature * 3 AS t1, humidity FROM events",
  "actions": [
    {
      "edgex": {
        "protocol": "tcp",
        "host": "*",
        "port": 5571,
        "topic": "application",
        "deviceName": "kuiper",
        "contentType": "application/json"
      }
    }
  ]
}
```
3) The data sent to EdgeX message bus.
```
{
  "Device": "kuiper", "Created": 0, …
  "readings": 
  [
     {"Name": "t1", value: "90" , "Created": 0 …},
     {"Name": "humidity", value: "20" , "Created": 0 …}
  ]
}
```
Please notice that, 
- The device name of ``Event`` structure is changed to ``kuiper``
- All of metadata for ``Events and Readings`` structure will be updated with new value. ``Created`` field is updated to another value generated by Kuiper (here is ``0``).

### Publish result to a new EdgeX message bus with keeping original metadata
But for some scenarios, you may want to keep some of original metadata. Such as keep the device name as original value that published to Kuiper (``demo`` in the sample), and also other metadata of readings arrays. In such case, Kuiper is acting as a filter - to filter NOT concerned messages, but still keep original data.

Below is an example,

1) Data received from EdgeX message bus ``events`` topic,
```
{
  "Device": "demo", "Created": 000, …
  "readings": 
  [
     {"Name": "Temperature", value: "30", "Created":123 …},
     {"Name": "Humidity", value: "20", "Created":456 …}
  ]
}
```
2) Use following rule,  and specify ``metadata`` with ``edgex_meta``  in ``edgex`` action.

```json
{
  "id": "rule1",
  "sql": "SELECT meta(*) AS edgex_meta, temperature * 3 AS t1, humidity FROM events WHERE temperature > 30",
  "actions": [
    {
      "edgex": {
        "protocol": "tcp",
        "host": "*",
        "port": 5571,
        "topic": "application",
        "metadata": "edgex_meta",
        "contentType": "application/json"
      }
    }
  ]
}
```
Please notice that,
- User need to add ``meta(*) AS edgex_meta`` in the SQL clause, the ``meta(*)`` returns all of metadata.
- In ``edgex`` action, value ``edgex_meta``  is specified for ``metadata`` property. This property specifies which field contains metadata of message.

3) The data sent to EdgeX message bus.
```
{
  "Device": "demo", "Created": 000, …
  "readings": 
  [
     {"Name": "t1", value: "90" , "Created": 0 …},
     {"Name": "humidity", value: "20", "Created":456 …}
  ]
}
```
Please notice that,
- The metadata of ``Events`` structure is still kept, such as ``Device`` & ``Created``.
- For the reading that can be found in original message, the metadata will be kept. Such as ``humidity`` metadata will be the ``old values`` received from EdgeX message bus.
- For the reading that can NOT be found in original message,  the metadata will not be set.  Such as metadata of ``t1`` in the sample will fill with default value that generated by Kuiper. 
- If your SQL has aggregated function, then it does not make sense to keep these metadata, but Kuiper will still fill with metadata from a particular message in the time window. For example, with following SQL, 
```SELECT avg(temperature) AS temperature, meta(*) AS edgex_meta FROM ... GROUP BY TUMBLINGWINDOW(ss, 10)```. 
In this case, there are possibly several messages in the window, the metadata value for ``temperature`` will be filled with value from 1st message that received from bus.

## Send result to MQTT message bus

Below is a rule that send analysis result to MQTT message bus, please notice how to specify ``ClientId`` in ``optional`` configuration.

```json
{
  "id": "rule1",
  "sql": "SELECT meta(*) AS edgex_meta, temperature, humidity, humidity*2 as h1 FROM demo WHERE temperature = 20",
  "actions": [
    {
      "edgex": {
        "protocol": "tcp",
        "host": "127.0.0.1",
        "port": 1883,
        "topic": "result",
        "type": "mqtt",
        "metadata": "edgex_meta",
        "contentType": "application/json",
        "optional": {
        	"ClientId": "edgex_message_bus_001"
        }
      }
    },
    {
      "log":{}
    }
  ]
}
```

