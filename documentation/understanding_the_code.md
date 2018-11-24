# Understanding the application's code

This section describes in details the code of the application. Since the code itself is commented it should be self explanatory. Therefore, you can skip reading this document and get back to it in case you need clarifications.

- [Structure of the application](./understanding_the_code.md#structure-of-the-application)
- Ingesting the device messages (events)
- Persisting the events
- Reading the persisted events
- Publishing updates to the dashboard
- Building the dashboard

## Structure of the application

The application is decomposed in three layers:

- /view: it contains the dashboard script. The dashboard invokes the scripts in the /api layer
- /api: it defines a simple API layer on top of the - simple - logic of the application and contains 3 simple scripts that implement API operation (getLatestData, getHistoricalData and inject)
- /entities: it contains scripts that implement the logic of the application. 

## Ingesting the device messages (events)

The **/api/inject** script automatically receives the mqtt messages that are published by the devices and broadcast by Everyware:

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

## Persisting the events

The **/entities/datamanager** script deals with the CRUD operation executed on the data that are received from the devices. More particularily, the saveData(event) method persists it into scriptr's NoSQL data store.

- First thing the saveData() does is to add the "type" field that will be used to determine the type of event that is received. If the latter contains speed, lat and long data, we set the type to "moving". Otherwise (we received passengers metrics and position) we set the value of type to "stop"
```
event.type = event.speed ? "moving" : "stop";  
```
- The function then creates some metadata that is used to inform scriptr's data store about the types of values it has to store. This is not a mandatory step as script will turn every value to a string by default. Adding metadata is howwever useful if we need to run some type-specific operations, such as asking script to calculate the average speed as we wil see it later on:
```
event["meta.types"] = {

  "speed": "numeric",
  "lat": "numeric",
  "long": "numeric",
  "absolutein": "numeric",
  "absoluteout": "numeric",
  "absolutepop": "numeric"
};

```
- to perist the data, the saveData() function only needs to require scriptr's native **document module** and invoke it's **create()** function, passing it the event + metadata:
```
var documentModule = require("document");
var resp = document.create(event);
```

## Reading the persisted events

Our applications' dashboard has to display the historical and latest events values 
