# Entur SDK

This SDK simplifies the use of Entur's travel APIs in JavaScript apps. For more information about Entur's APIs, see https://developer.entur.org

Miss anything? Found a bug? File an [issue](https://github.com/entur/sdk/issues/new) or create a pull request!

* [Installation](#install)
* [Setup](#setup)
* [API](#api)
    * [findTrips](#findtrips)
    * [getTripPatterns](#gettrippatterns)
    * [getFeatures](#getfeatures)
    * [getStopPlaceDepartures](#getstopplacedepartures)
    * [getBikeRentalStation](#getbikerentalstation)
    * [getBikeRentalStations](#getbikerentalstations)
    * [getStopPlace](#getstopplace)
    * [getStopPlacesByPosition](#getstopplacesbyposition)
* [Utils](#utils)
    * [throttler](#throttler)
* [Types](#flow-and-typescript)

## Install
```bash
npm install @entur/sdk --save
```

## Setup
```javascript
import EnturService from '@entur/sdk'

const service = new EnturService({ clientName: 'awesomecompany-awesomeapp' })
```

### Configuration
| Name        | Type                  | Default     | Description                             |
|:------------|:----------------------|:------------|:----------------------------------------|
| clientName  | `string`              | `undefined` | The name of your application            |
| hosts       | `{object of hosts}`   | `{}`        | Override default endpoints              |


#### clientName (required)
We require that you pass a `clientName` that identifies your application. It should contain the name of your company or organization,
followed by a hyphen and your application's name. See https://developer.entur.org/pages-intro-authentication for more information.

#### hosts
The Entur SDK uses multiple endpoints for its services. Each endpoint can be overridden with hosts config (in case you use a proxy or a local instance of the endpoint). Available hosts are:

```javascript
{
    journeyplanner: '',
    geocoder: ''
}
```

## API

### findTrips

```javascript
(from: string, to: string, date?: Date | string | number) => Promise<Array<TripPattern>>
```

Finds up to 5 trip patterns from <from> to <to> at the time specified. This is a convenience method, which first tries to find locations for the given <from> and <to> strings before searching for trips between them. If you need more control, see the [`getTripPatterns`](#gettrippatterns) method.

If no locations are found for either <from> or <to>, or if <date> is invalid, an error will be thrown.

#### Parameters

##### from (`string`)
The place you want to search from. For instance `"Oslo"`

##### to (`string`)
The place you want to search from. For instance `"Bergen"`

#### date (`Date | string | number`) [Optional]
The wanted time of departure. Can be anything that is parseable by `new Date()`. If not provided, the search will be from "now".

### getTripPatterns

```javascript
(from: Location, to: Location, params?: GetTripPatternsParams, ignoreFields?: Array<string>) => Promise<Array<TripPattern>>
```

Types: [TripPattern](flow-types/TripPattern.js)

`getTripPatterns` is for searching for itineraries for a trip from some location to a destination at a given time. The method takes one argument `query`, which is an object with search parameters.

If you are going to do a huge amount of different searches at the same time, consider using our [`throttler`](#throttler) utility.

#### Parameters

##### from (`Location`)
The location to search for travels from.

##### to (`Location`)
The destination location to search for travels to.

##### params (`GetTripPatternsParams`) [Optional]
An object of search parameters.

| Key                     | Type               | Default   | Description |
|:------------------------|:-------------------|:----------|:------------|
| `searchDate`            | `Date`             | | when to calculate patterns |
| `arriveBy`              | `boolean`          | `false` | depart by `searchDate`, or arrive by `searchDate` |
| `modes`                 | [`Array of Modes`](#leg-mode) | `['foot', 'bus', 'tram', 'rail', 'metro', 'water', 'air']` | modes of transport to include in trip |
| `limit`                 | `number`           | `5`      | Limit result to this number of trip patterns |
| `wheelchairAccessible`  | `boolean`          | `false`  | include only stops which are wheelchair accessible |
| `walkSpeed`             | `number`           | `1.3`    | the walk speed to use in searches in meters per second |


##### ignoreFields  [Optional]
A list of keys to exclude from the resulting trip patterns.

Default:
```
[
    'notices',
    'situations',
    'journeyPattern',
    'fromEstimatedCall',
    'toEstimatedCall',
    'intermediateEstimatedCalls',
    'interchangeFrom',
    'interchangeTo',
    'pointsOnLink',
    'authority',
    'operator',
    'quay',
    'bookingArrangements',
    'rentedBike',
]
```

#### Example

```javascript
service.getTripPatterns(
    {
        name: 'Ryllikvegen, Lillehammer',
        coordinates: {
            latitude: 61.102848368937416,
            longitude: 10.51613308426234
        },
    },
    {
        place: 'NSR:StopPlace:337',
        name: 'Oslo S, Oslo'
    },
    {
        searchDate: new Date(),
    }
})
```

See [example/get-trip.js](./example/get-trip.js) for a more in depth example

### getFeatures

```javascript
(query: string, coords?: Coordinates, params?: GetFeaturesQuery) => Promise<Array<Feature>>
```

Types: [Feature](flow-types/Feature.js), [Coordinates](flow-types/Coordinates.js)

`getFeatures` is for searching for stop places, stations or addresses.

#### Parameters

##### query (`string`)
The search string that should resemble the name of the desired stop place or address. Examples: `"Oslo S"`, `"Schweigaards gate 23, Oslo"`, `"Voss stasjon"`.

##### coords (`Coordinates`) [Optional]
A set of coordinates to use when the weighting search results. Examples: `{ latitude: 59.909774, longitude: 10.763712 }`.

The results closest to the coordinates will be weighted above results with equally good string matches.
As an example, the street `Dronningens gate` exists both in Oslo and Trondheim. If you call `service.getFeatures('Dronningens gate', { latitude: 63.4305103, longitude: 10.3949874 })` (coordinates of Trondheim city center), the Dronningens gate in Trondheim will be preferred to the one in Oslo.

##### params (`GetFeaturesQuery`) [Optional]
An optional object of parameters to pass to the query.

| Key                  | Type     | Default             | Description |
|:---------------------|:---------|:--------------------|:------------|
| `layers`             | `string` | `"venue,address"`   | The types of places to search for in a comma-separated string. `venue` means stop places and stations, `address` means postal addresses that might not be connected to public transport.

### getDeparturesFromStopPlace

```javascript
(stopPlaceId: string, params?: GetDeparturesParams) => Promise<Array<EstimatedCall>>
```

`getDeparturesFromStopPlace` finds departures from one given stop place. Also see `getDeparturesFromStopPlaces` for fetching for multiple stops simultaneously.
The method will return a Promise which will resolve to an array of departures for that stop place.

#### Parameters

##### stopPlaceId (`string`)
The ID of the stop place you are interested in.

##### params (`Object`) [Optional]
An optional object of parameters to pass to the query.

| Key                      | Type           | Default      | Description |
|:-------------------------|:---------------|:-------------|:------------|
| `start`                  | `Date`         | `new Date()` | DateTime for when to fetch estimated calls from. |
| `timeRange`              | `number`       | `72000`      | The time range for departures to include in seconds. |
| `departures`             | `number`       | `5`          | The number of departures to return for each stop place. |
| `includeNonBoarding`     | `boolean`      | `false`      | Whether to include departures that do not accept boarding at given stop place. |
| `limit`                  | `number`       | `50`         | The maximum number of departures to fetch. |
| `whiteListedLines`       | `Array<string>` | `undefined` | A list of line IDs to include. All others will be excluded. |
| `whiteListedAuthorities` | `Array<string>` | `undefined` | A list of authority IDs to include. All others will be excluded. |

### getDeparturesFromStopPlaces

```javascript
(stopPlaceIds: Array<string>, params?: GetDeparturesParams) => Promise<Array<{ id: string, departures: Array<EstimatedCall> }>>
```

Types: [EstimatedCall](flow-types/EstimatedCall.js)

`getDeparturesFromStopPlaces` finds departures from one or more given stop places. Also see `getDeparturesFromStopPlace` for a simpler interface for fetching departures for a single stop.

The method will return an array of objects containing fields for the stop place's `id` and `departures`.

#### Parameters

##### stopPlaceIds (`Array<string>`)
The IDs of the stop places you are interested in.

##### params (`Object`) [Optional]

See the `params` parameter for `getDeparturesForStopPlace`.

### getBikeRentalStation

```javascript
(stationId: string) => Promise<BikeRentalStation>
```

Types: [BikeRentalStation](flow-types/BikeRentalStation.js)

`getBikeRentalStation` finds a single bike rental station by its ID.

#### Parameters

##### stationId (`string`)
The ID of the bike rental station you are interested in. The method will return a Promise which will resolve to an object of type [BikeRentalStation](flow-types/BikeRentalStation.js).

### getBikeRentalStations

```javascript
(coordinates: Coordinates, distance?: number) => Promise<Array<BikeRentalStation>>
```

Types: [BikeRentalStation](flow-types/BikeRentalStation.js)

`getBikeRentalStations` finds bike rental stations within an area surrounding a coordinate.

#### Parameters

##### coordinates (`{ latitude: number, longitude: number }`)
The coordinates of which to find bike rental stations around.

##### distance (`number`) [Optional]
Default: `500`

The "radius" in meters of the surrounding bounding box in which you want to find bike rental stations.
The width and height of the bounding box are therefore `2 * distance`, and the coordinates given are its centerpoint.

### getStopPlace

```javascript
(id: string) => Promise<StopPlace>
```

Types: [StopPlace](flow-types/StopPlace.js)

`getStopPlace` finds the stop place with the given ID.

### getStopPlacesByPosition

```javascript
(coordinates: Coordinates, distance?: number) => Promise<Array<StopPlace>>
```

Types: [StopPlace](flow-types/StopPlace.js)

`getStopPlacesByPosition` finds stop places within an area surrounding a coordinate.

#### Parameters

##### coordinates (`Coordinates`)
The coordinates of which to find bike rental stations around.

##### distance (`number`) [Optional]
Default: `500`

The "radius" in meters of the surrounding bounding box in which you want to find stop places.
The width and height of the bounding box are therefore `2 * distance`, and the coordinates given are its centerpoint.

## Utils

### `throttler`

If you are going to do a lot of requests at the same time, you are likely to exceed our rate limits.
To help you with this, we have a `throttler` utility that throttles the requests in order to respect the rate limits.

Example usage:

```
import EnturService, { throttler, convertFeatureToLocation } from '@entur/sdk'

const service = new EnturService({ clientName: 'myawesomecompany-myawesomeapp' })

async function getTripPatternsForVeryManyDifferentLocations() {
  const [fromLocation] = await service.getFeatures('Oslo S')
  const [toLocation] = await service.getFeatures('Drammen stasjon')
  const params = {
      searchDate: new Date(),
      from: convertFeatureToLocation(fromLocation),
      to: convertFeatureToLocation(toLocation),
  }

  // A huge array of arguments that we want to call a function with, one by one.
  const queries = Array(3000).fill(params)

  // We pass the function and the huge list of arguments to the throttler.
  // The resulting list will be in the same order as the arguments passed.
  const tripPatterns = await throttler(query => service.getTripPatterns(query), queries)
  console.log('Done!')
  return tripPatterns
}
```

## Flow and TypeScript

We provide library definitions for Flow and TypeScript. TypeScript should work out of the box. For Flow, make sure you include the following in your .flowconfig:

```
[libs]
node_modules/@entur/sdk/lib/libdef.flow.js
```
