# Smartcar Node SDK [![Build Status][ci-image]][ci-url] [![npm version][npm-image]][npm-url]

The official Smartcar Node SDK.

## Overview

The [Smartcar API](https://smartcar.com/docs) lets you read vehicle data
(location, odometer) and send commands to vehicles (lock, unlock) using HTTP requests.

To make requests to a vehicle from a web or mobile application, the end user
must connect their vehicle using
[Smartcar Connect](https://smartcar.com/docs/api#smartcar-connect).
This flow follows the OAuth spec and will return a `code` which can be used to
obtain an access token from Smartcar.

The Smartcar Node SDK provides methods to:

1. Generate the link to redirect to Connect.
2. Make a request to Smartcar with the `code` obtained from Connect to obtain an
   access and refresh token
3. Make requests to the Smartcar API to read vehicle data and send commands to
   vehicles using the access token obtained in step 2.

Before integrating with Smartcar's SDK, you'll need to register an application
in the [Smartcar Developer portal](https://developer.smartcar.com). If you do
not have access to the dashboard, please
[request access](https://smartcar.com/subscribe).

### Flow

- Create a new `AuthClient` object with your `clientId`, `clientSecret`,
  `redirectUri`.
- Redirect the user to Smartcar Connect using `getAuthUrl` with required `scope` or with one
  of our frontend SDKs.
- The user will login, and then accept or deny your `scope`'s permissions.
- Handle the get request to `redirectUri`.
  - If the user accepted your permissions, `req.query.code` will contain an
    authorization code.
    - Use `exchangeCode` with this code to obtain an access object
      containing an access token (lasting 2 hours) and a refresh token
      (lasting 60 days).
      - Save this access object.
    - If the user denied your permissions, `req.query.error` will be set
      to `"access_denied"`.
    - If you passed a state parameter to `getAuthUrl`, `req.query.state` will
      contain the state value.
- Get the user's vehicles with `getVehicles`.
- Create a new `Vehicle` object using a `vehicleId` from the previous response,
  and the `access_token`.
- Make requests to the Smartcar API.
- Use `exchangeRefreshToken` on your saved `refreshToken` to retrieve a new token
  when your `accessToken` expires.

### Installation

```shell
npm install smartcar --save
```

### Example

```javascript
'use strict';

const express = require('express');
const { Smartcar, AuthClient, Vehicle } = require('smartcar');

const app = express();

const port = 8000;

const smartcar = new Smartcar();

const client = new AuthClient({
  clientId: 'clientId',
  clientSecret: 'clientSecret',
  redirectUri: 'http://localhost:8000/exchange',
});

app.get('/', function (req, res) {
  /** scope info
   * read_compass: Know the compass direction your vehicle is facing
   * read_engine_oil: Read vehicle engine oil health
   * read_battery: Read EV battery's capacity and state of charge
   * read_charge: Know whether vehicle is charging
   * control_charge: Start or stop your vehicle's charging
   * read_thermometer: Read temperatures from inside and outside the vehicle
   * read_fuel: Read fuel tank level
   * read_location: Access location
   * control_security: Lock or unlock your vehicle
   * read_odometer: Retrieve total distance traveled
   * read_speedometer: Know your vehicle's speed
   * read_tires: Read vehicle tire pressure
   * read_vehicle_info: Know make, model, and year
   * read_vin: Read VIN
   */
  const scope = ['read_vehicle_info'];
  const authUrl = client.getAuthUrl(scope);
  res.redirect(authUrl);
});

app.get('/exchange', async function (req, res) {
  const code = req.query?.code;
  console.log(code);
  if (!code) return;

  const access = await client.exchangeCode(code);
  const { vehicles } = await smartcar.getVehicles(access.accessToken);
  console.log(access.accessToken);
  const vehicle = new Vehicle(vehicles[0], access.accessToken);
  const attributes = await vehicle.attributes();

  console.log(attributes);
});

app.listen(port, () => console.log(`Listening on port ${port}`));

```

## SDK Reference

For detailed documentation on parameters and available methods, please refer to
the [SDK Reference](doc/readme.md).

## Contributing

To contribute, please:

1. Open an issue for the feature (or bug) you would like to resolve.
2. Resolve the issue and add tests in your feature branch.
3. Open a PR from your feature branch into `develop` that tags the issue.

To test:

```shell
npm run test
```

Note: In order to run tests locally the following environment variables would have to be set :

- `E2E_SMARTCAR_CLIENT_ID` - Client ID to be used.
- `E2E_SMARTCAR_CLIENT_SECRET` - Client secret to be used.
- `E2E_SMARTCAR_AMT` - AMT from dashboard for webhooks tests.
- `E2E_SMARTCAR_WEBHOOK_ID` - Webhook ID use in the webhook tests success case.

Your application needs to have https://example.com/auth set as a valid redirect URI

[ci-url]: https://travis-ci.com/smartcar/node-sdk
[ci-image]: https://travis-ci.com/smartcar/node-sdk.svg?token=jMbuVtXPGeJMPdsn7RQ5&branch=master
[npm-url]: https://badge.fury.io/js/smartcar
[npm-image]: https://badge.fury.io/js/smartcar.svg

## Supported Node.js Versions

Smartcar aims to support the SDK on all Node.js versions that have a status of "Maintenance" or "Active LTS" as defined in the [Node.js Release schedule](https://github.com/nodejs/Release#release-schedule).

In accordance with the Semantic Versioning specification, the addition of support for new Node.js versions would result in a MINOR version bump and the removal of support for Node.js versions would result in a MAJOR version bump.
