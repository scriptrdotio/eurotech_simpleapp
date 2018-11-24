# Understanding the application's code

- Ingesting the device messages
- Persisting the data

## Ingesting the device messages

The /api/inject script automatically receives the mqtt messages that are published by the devices and broadcast by Everyware:

- the mqtt message is received in the rawBody property of the native **request** object and it is parsed into a JSON object
```
var message = JSON.parse(request.rawBody);
```
- the script then extracts the device id from the name of the topic to which the message was published (this name is provided in the mqtt message)
```
var msgParts = message.topic.split("/");
var data = {

   deviceId: msgParts[1],
   payload: message.payload
}; 

```
- the message payload is then transformed into a flat key/value pairs structure. Note that a device can publish two types of messages:   
  - one type is related to the bus speed and its position (speed, lat, lon) 
  - the other type is related to the number of passengers gettings of/on the bus and its positon (absolutein, absoluteout, absoluteopop, lat, lon)
```
var event = {
    id: data.deviceId
};
   
if (data.payload.position.speed) {    
    event.speed = data.payload.position.speed;
}
    
if (data.payload.position.latitude) {
        
    event.lat = Number(data.payload.position.latitude).toFixed(4);
    event.long = Number(data.payload.position.longitude).toFixed(4);
}

for (var i = 0; data.payload.metric && i < data.payload.metric.length; i++) {
   event[data.payload.metric[i].name.toLowerCase()] = data.payload.metric[i].int_value // this code assumes all values are of type int
}
```
- once this data structure is ready, the inject script hands it over to the saveData() function defined in the datamanager script
```
var dataManager = require("../entities/datamanager");
return dataManager.saveData(event);
```
