# eurotech_simpleapp

This is a simple demo app that demonstrates how easy it is to leverage data published by eurotech devices into scriptr.

## What the application does

The application is based on the Eurotech [PCN-Transport simulator](https://cs.eurotech.com/gps-pcn-simulator/). The simulator mimics the behavior of a device installed on a bus, and which publishes data to a eurotech Everyware mqtt topic. Two types of data are published:
- location and speed data. This is published while the bus is moving
- bus load data (passengers getting off, getting on, current number of passengers). This is published while the bus is stopped.

On the scriptr side, the application is subscribed to the Everyware mqtt topic. As soon as events arrive, they are are persisted in the scriptr's data store and further published to a simple dashboard that is updated in real time.

![Application dashboard on scriptr](./documentation/images/dashboard.png)

*Image 1 - The application's dashboard*

## Pre-requisites

- You need to own an [Everyware account](https://console-sandbox.everyware-cloud.com/) (sandbox in this example) and you need to create an mqtt topic in the platform to which the simulator will publish data
- You need to own a [scriptr account](https://www.scriptr.io/login)

