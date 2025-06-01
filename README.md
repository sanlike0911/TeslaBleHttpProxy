# TeslaBleHttpProxy

TeslaBleHttpProxy is a program written in Go that receives HTTP requests and forwards them via Bluetooth to a Tesla vehicle. The program can, for example, be easily used together with [evcc](https://github.com/evcc-io/evcc).

The program stores the received requests in a queue and processes them one by one. This ensures that only one Bluetooth connection to the vehicle is established at a time.

## Table of Contents

- [How to install](#how-to-install)
  - [Docker compose](#docker-compose)
  - [Build yourself](#build-yourself)
- [Generate key for vehicle](#generate-key-for-vehicle)
- [Setup EVCC](#setup-evcc)
- [API](#api)
  - [Vehicle Commands](#vehicle-commands)
  - [Vehicle Data](#vehicle-data)
  - [Body Controller State](#body-controller-state)
- [Vehicle Compatibility and Requirements](#vehicle-compatibility-and-requirements)

## How to install

You can either compile and use the Go program yourself or install it in a Docker container. ([detailed instruction](docs/installation.md))

### Docker compose

Below you will find the necessary contents for your `docker-compose.yml`:

```
services:
  tesla-ble-http-proxy:
    image: wimaha/tesla-ble-http-proxy
    container_name: tesla-ble-http-proxy
    environment:
      - cacheMaxAge=30 # Optional, but recommended to set this to anything more than 0 if you are using the vehicle data
    volumes:
      - ~/TeslaBleHttpProxy/key:/key
      - /var/run/dbus:/var/run/dbus
    restart: always
    privileged: true
    network_mode: host
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
```

Please remember to create an empty folder where the keys can be stored later. In this example, it is `~/TeslaBleHttpProxy/key`.

Pull and start TeslaBleHttpProxy with `docker compose up -d`.

Note that you can optionally set environment variables to override the default behavior. See [environment variables](docs/environment_variables.md) for more information.

### Build yourself

Download the code and save it in a folder named 'TeslaBleHttpProxy'. From there, you can easily compile the program.

```
go build .
./TeslaBleHttpProxy
```

Please remember to create an empty folder called `key` where the keys can be stored later.

Note that you can optionally set environment variables to override the default behavior. See [environment variables](docs/environment_variables.md) for more information.

## Generate key for vehicle

*(Here, the simple, automatic method is described. Besides the automatic method, you can also generate the keys [manually](docs/manually_gen_key.md).)*

To generate the required keys browse to `http://YOUR_IP:8080/dashboard`. In the dashboard you will see that the keys are missing:

<img src="docs/proxy1.png" alt="Picture of the Dashboard with missing keys." width="40%" height="40%">

Please click on `generate Keys` and the keys will be automatically generated and saved.

<img src="docs/proxy2.png" alt="Picture of the Dashboard with success message and keys." width="40%" height="40%">

After that please enter your VIN under `Setup Vehicle`. Before you proceed make sure your vehicle is awake! So you have to manually wake the vehicle before you send the key to the vehicle.

<img src="docs/proxy3.png" alt="Picture of Setup Vehicle Part of the Dashboard." width="40%" height="40%">

Finally the keys is send to the vehicle. You have to confirm by tapping your NFC card on center console.

<img src="docs/proxy6.png" alt="Picture of success message sent add-key request." width="40%" height="40%">

You can now close the dashboard and use the proxy. 🙂

## Setup EVCC

You can use the following configuration in evcc (recommended):

```
vehicles:
  - name: tesla
    type: template
    template: tesla-ble
    title: Your Tesla (optional)
    capacity: 60 # Akkukapazität in kWh (optional)
    vin: VIN # Erforderlich für BLE-Verbindung
    url: IP # URL des Tesla BLE HTTP Proxy
    port: 8080 # Port des Tesla BLE HTTP Proxy (optional)
```

If you want to use this proxy only for commands, and not for vehicle data, you can use the following configuration. The vehicle data is then fetched via the Tesla API by evcc.

```
- name: model3
    type: template
    template: tesla
    title: Tesla
    icon: car
    commandProxy: http://YOUR_IP:8080
    accessToken: YOUR_ACCESS_TOKEN
    refreshToken: YOUR_REFRSH_TOKEN
    capacity: 60
    vin: YOUR_VIN
```

(Hint for multiple vehicle support: https://github.com/wimaha/TeslaBleHttpProxy/issues/40)

## API

### Vehicle Commands

The program uses the same interfaces as the Tesla [Fleet API](https://developer.tesla.com/docs/fleet-api#vehicle-commands). Currently, the following requests are supported: 

- wake_up
- charge_start
- charge_stop
- set_charging_amps
- set_charge_limit
- auto_conditioning_start
- auto_conditioning_stop
- charge_port_door_open
- charge_port_door_close
- flash_lights
- honk_horn
- door_lock
- door_unlock
- set_sentry_mode

By default, the program will return immediately after sending the command to the vehicle. If you want to wait for the command to complete, you can set the `wait` parameter to `true`.

#### Example Request

*(All requests with method POST.)*

Start charging:
`http://localhost:8080/api/1/vehicles/{VIN}/command/charge_start`

Start charging and wait for the command to complete:
`http://localhost:8080/api/1/vehicles/{VIN}/command/charge_start?wait=true`

Stop charging:
`http://localhost:8080/api/1/vehicles/{VIN}/command/charge_stop`

Set charging amps to 5A:
`http://localhost:8080/api/1/vehicles/{VIN}/command/set_charging_amps` with body `{"charging_amps": "5"}`

### Vehicle Data

The vehicle data is fetched from the vehicle and returned in the response in the same format as the [Fleet API](https://developer.tesla.com/docs/fleet-api/endpoints/vehicle-endpoints#vehicle-data). Since a ble connection has to be established to fetch the data, it takes a few seconds before the data is returned.

#### Example Request

*(All requests with method GET.)*

Get vehicle data:
`http://localhost:8080/api/1/vehicles/{VIN}/vehicle_data`

Currently you will receive the following data:

- charge_state
- climate_state

If you want to receive specific data, you can add the endpoints to the request. For example:

`http://localhost:8080/api/1/vehicles/{VIN}/vehicle_data?endpoints=charge_state`

This is recommended if you want to receive data frequently, since it will reduce the time it takes to receive the data.

### Body Controller State

The body controller state is fetched from the vehicle and returnes the state of the body controller. The request does not wake up the vehicle. The following information is returned:

- `vehicleLockState`
  - `VEHICLELOCKSTATE_UNLOCKED`
  - `VEHICLELOCKSTATE_LOCKED`
  - `VEHICLELOCKSTATE_INTERNAL_LOCKED`
  - `VEHICLELOCKSTATE_SELECTIVE_UNLOCKED`
- `vehicleSleepStatus`
  - `VEHICLE_SLEEP_STATUS_UNKNOWN`
  - `VEHICLE_SLEEP_STATUS_AWAKE`
  - `VEHICLE_SLEEP_STATUS_ASLEEP`
- `userPresence`
  - `VEHICLE_USER_PRESENCE_UNKNOWN`
  - `VEHICLE_USER_PRESENCE_NOT_PRESENT`
  - `VEHICLE_USER_PRESENCE_PRESENT`

#### Request

*(All requests with method GET.)*

Get body controller state:
`http://localhost:8080/api/1/vehicles/{VIN}/body_controller_state`

## Vehicle Compatibility and Requirements

TeslaBleHttpProxy requires your Tesla vehicle to support **Phone Key** functionality, as it relies on Bluetooth Low Energy (BLE) for communication. Most Tesla models from 2021 onward support Phone Key, but some older models (e.g., Model X 2015–2020) may not. Please verify Phone Key support in your Tesla app or consult Tesla's [Vehicle Keys Support Page](https://www.tesla.com/support/tesla-vehicle-keys) before setting up the proxy. Ensure the device running TeslaBleHttpProxy is within Bluetooth range (~5–10 meters) of your vehicle.
