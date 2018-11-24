# Understanding the application's code

This section describes in details the code of the application. Since the code itself is commented it should be self explanatory. Therefore, you can skip reading this document and get back to it in case you need clarifications.

- [Structure of the application](./understanding_the_code.md#structure-of-the-application)
- [Ingesting the device messages (events)](./understanding_the_code.md#ingesting-the-device-messages-events)
- [Persisting the events](./understanding_the_code.md#persisting-the-events)
- [Reading the persisted events](./understanding_the_code.md#reading-the-persisted-events)
- Publishing updates to the dashboard
- Building the dashboard

## Structure of the application

The application is decomposed in three layers:

- /view: it contains the dashboard script. The dashboard invokes the scripts in the /api layer
- /api: it defines a simple API layer on top of the - simple - logic of the application and contains 3 simple scripts that implement API operation (getLatestData, getHistoricalData and inject)
- /entities: it contains scripts that implement the logic of the application. 

[back](./understanding_the_code.md#understanding-the-applications-code)

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
[back](./understanding_the_code.md#understanding-the-applications-code)

## Persisting the events

The **/entities/datamanager** script deals with the CRUD operation executed on the data that are received from the devices. More particularily, the **saveData(event)** method persists it into scriptr's NoSQL data store.

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

[back](./understanding_the_code.md#understanding-the-applications-code)

## Reading the persisted events

Our applications' dashboard has to display the historical and latest events values, as well as the average speed of the bus. Querying the persisted events to provide the latter is done by 3 functions defined in the **/entities/datamanager** script

### Reading historical event values

This is the job of the **listHistoricalData(deviceId, pageNumber, resultsPerPage, eventType)** function. It is a generic function that retrieves the events of a given type (i.e "moving" or "stop" as described earlier), for a given device id. If the latter is not provided, it reads the events of the provided type, for all devices.

- first thing the function does it to prepare the query that will be executed agains scriptr's data store. The query has to specify:
  - the fields to return. In our case, we instruct scriptr to return all fields  - '*'
  - the number (index) of the page to return (scriptr automatically paginates the results if their total number is > 50)
  - the max number of results to return per page (this number has to be <= 50, which is the default value)
  - the count field to instruct scriptr to return the total count of results
  - most importantly, the query expression, which a string that defines the filtering criteria to query the events. In our case, we will be filtering by event type and - if available - device id, e.g. 'type="moving" and id="scriptr-bus-002"
```
var queryParams = {        

     fields: "*",
     query: 'type="' + (eventType ? eventType :  "moving") + '"',
     count: true
 };

 if (deviceId){
     queryParams.query = 'deviceId="' + deviceId + '"';
 }

 if (pageNumber) {
     queryParams.pageNumber = pageNumber;
 }

 if (resultsPerPage) {
     queryParams.resultsPerPage = resultsPerPage;
 }
```
- next thing is to execute our query by passing the above to the  query() function of the native **document** module, required ealier in the script
```
var resp = document.query(queryParams); 
return resp.result.documents;
```

### Reading latest event values

The **getLatestData(deviceId, eventType)** is in charge of returning the latest event data of a given event type for a givend device id. If the device id is not provided, it just returns the latest event of the specified event type. 

By default, queries in scriptr always retun a list of record sorted by descending creation date. Therefore, we can reuse the listHistoricalData() function and only return the first item on the list.
```
var latestData = listHistoricalData(deviceId, 1, 1, eventType)[0];
```

### Calculating the average speed

Scriptr's simple yet powerful querying mechanism allows you to execute aggregates functions on your data (min, max, sum, average, etc.). We leverage this ability in the **getAverageSpeed(deviceId)** function.

- first, we defined the query parameters by specifying:
  - the fields to return, in our case, we're only interested in "speed"
  - the aggregate expression, i.e. the aggregate function to apply. In our case, it is the average function applied to speed
  - the aggregare scope, i.e. specify if the aggregation should apply on all the recorded events or only on a page by page basis
  - the query expression, i.e. the filter. In our case, we only need to consider the events that contain speed, i.e. those of type 
  "moving", for a given device id when the latter is provided.
 ```
 var queryParams = {

     query: 'type="moving"',
     fields: "speed",
     aggregateExpression: "avg($speed)",
     aggregateGlobal: true
 };

 if (deviceId) {
     queryParams.query = queryParams.query + ' and deviceId="' + deviceId + '"';
 }
 ```
- next, we execute the query by passing the above parameters to the query() function of the native **document** module.
```
var resp = document.query(queryParams);
return resp.result.aggregate.globalScope.value;
```

[back](./understanding_the_code.md#understanding-the-applications-code)
