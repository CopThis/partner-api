# Example Partner API

## Staging and Production Environments

Merchbar believes strongly in testing our integrations rigorously before enabling them on production data, so we ask that all of our partners set up a staging environment in addition to their production API servers.

The staging environment should service all of the same API requests as the production environment, with realistic-looking responses for each request that are consistent across calls. In cases where there would be real-world effects for API calls (e.g. shipping actual merchandise to a customer) those actions should not happen in the staging environment. In cases where the effects of real world actions would be necessary to completely simulate an API response (e.g. supplying the tracking number for an order) Merchbar recommends the staging environment supply a realistic-looking, but faked, response.


## Security and Authentication

All requests to the API will be made over HTTPS, and any non-TLS connections should be immediately rejected. Cipher suite selection is up to individual partners, but we second Mozilla's [Modern Compatibility](https://wiki.mozilla.org/Security/Server_Side_TLS#Modern_compatibility) recommendation.

Merchbar expects to authenticate with the API server using [HTTP Basic Authentication](https://en.wikipedia.org/wiki/Basic_access_authentication). Specifically, all requests will be made with a header as follows:

```
Authorization: Basic cHFOMllNWVdZMmg4OWJSamkwalFIRzZhVFZKSDJrOg==
```

Where `cHFOMllNWVdZMmg4OWJSamkwalFIRzZhVFZKSDJrOg==` is the base64 representation of username/password pair tied to Merchbar's authorization on the API server. For simplicity's sake, we recommend using a randomly generated token as the username and a blank string as the password, which by [the spec](https://tools.ietf.org/html/rfc1945#section-11.1) are concatenated with a colon before being base64 encoded (this example was `pqN2YMYWY2h89bRji0jQHG6aTVJH2k:`).

Merchbar will only use one token (or username/password pair) at a time, but it should be possible through some means to invalidate previous credentials and create new ones in order to secure against possible (though very unlikely) leaks of authentication credentials.


## JSON Response Envelope

All successful responses should have an HTTP status code `200` and should be formatted as JSON objects with a single key, `data`, that contains the response data.  For `POST` requests and single entity GET requests (i.e. `GET /v1/orders/123`), the `data` attribute should be an object, while collection GETs (i.e. `GET /v1/stores/`) should return an array of objects.

If an endpoint supports pagination, an additional `pagination` key should be returned with an object containing the page size `per_page`, page number `current_page` (1-indexed), and the number of pages accessible `total_pages`.

```json
{
    "data": { },
    "pagination": {
        "current_page": 1,
        "per_page": 20,
        "total_pages": 15
    }
}
```


## Errors

If an error occurs, a response should be returned with a `4xx` or `5xx` HTTP status code (note: any `200` will be considered successful). In this case, instead of a `data` key, the top-level response object should include an `error` key for an error response.

The value of `error` may be a simple string, such as `{"error": "Forbidden"}`, or it may be an object that contains details relevant to the error. The implementation of errors codes and messages is left to the partner, but consistent use of HTTP error codes with detailed error messages is of great assistance to any user of the API.

## Request formats

In general, Merchbar recommends that resource identifiers be passed as parameters to API endpoints as part of the URI. If the resulting action is simply to return information about a resource, the request should operate via the `GET` HTTP method, while `POST` and `PATCH` should be used when data is being mutated. For `GET` requests, any parameters must be passed as URL query parameters. For `POST` and `PATCH` requests, the HTTP body should contain a single JSON `object` with all of the information needed to perform the action. `PATCH` differs from `POST` in that partial representations of objects are acceptable in a `PATCH` request (with omitted fields left unchanged).


## JSON Representation

Most data types are easily represented with the standard JSON data types, `object`, `array`, `number`, `string`, `boolean` and `null`. However, there are a few types that are commonly used but have no standard representation:

 - **Floating point numbers** are part of the JSON spec, but we recommend against using them for values that need any true precision. For its own API, Merchbar tries not to use them at all.
 - **Prices** in particular need precision, so these values should be marshalled to JSON as strings without a dollar sign (or any other currency marker) present, e.g. `19.99`.
 - **Dates and times** are sensitive to timezones, so these values should be marshalled to JSON as strings with the time expressed in UTC, using [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) recommendations, e.g. `"2016-08-01"` for dates, and `"2016-08-01T12:34:56Z"` for combination date/times.
 - **Identifiers** for objects (like stores, merch items, and orders) can be either represented as a `number` (integers) or `string`. Merchbar stores these values as strings on our end for maximum flexibility.

## API Versioning

It's common to want to make changes to APIs as new functionality is added or certain business realities change that require a corresponding change in code. However, after an API has been published, all efforts possible should be made to ensure that backwards compatibility is not lost. Optimally, no API clients should need to ever change their currently-working integration based on a change made to the API.

In the case where a change needs to be made that cannot be backwards compatible, a new version of the API should be created for the new functionality. We also ask that, when creating a new version of the API, Merchbar be notified as to the deprecation timeline for the old API.

To facilitate versioned API endpoints, we recommend including the version number in the URL path of every request, and we do so in all of our examples below.



# API Endpoints

The following is a list of the recommended methods for a partner API. Within each endpoint, we show an example of the expected output containing the fields we consider to be required. However, it's fine to send along other metadata as well, as needed or desired.

## GET `/v1/stores`

Returns a list of stores. Typically, one store is assigned for each artist. We support hiding stores from our users, so if you want to prevent us from offering any of the merchandise in a given store, make sure to return `"visible": false`. This endpoint should be paginated unless the number of stores is small.

**Parameters:**

name | type | description
--- | --- | ---
`per_page` | `number` | the number of stores to include per page (optional)
`page` | `number` | the current page number (1-indexed, optional)

**Example Response:**

```json
{
    "data": [
        {
            "id": 1,
            "title": "Artist Name",
            "visible": true
        },
        {
            "id": 2,
            "title": "Another Artist",
            "visible": false
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 10,
        "total_pages": 1
    }
}
```

## GET `/v1/stores/{id}`
Returns extended store metadata. In order to keep our store status up to date, Merchbar should be able to poll this API endpoint for every store in your system at least once a day.

**Example Response:**

```json
{
    "data": {
        "id": 1,
        "title": "Artist Name",
        "description": "The official store of Artist Name!",
        "photo": "https://dt72k0ec4onep.cloudfront.net/JSNEMSDE92381733.JPG",
        "visible": true
    }
}
```

## GET `/v1/stores/{id}/merch`

Returns a list of merchandise within a specific store. In order to keep our item quantities up to date, Merchbar should be able to poll this API endpoint for every store in your system at least once a day. This endpoint should be paginated.

A note on merch and variants: for a given product line, there may be multiple ways to split the various SKUs. There are no hard and fast rules, but here are some guidelines to follow that will ensure the best experience for customers on Merchbar:

 - Each merch item should be a single design in a single color. Split items with multiple colors into multiple merch items for this feed.
 - If there are different fits, e.g. men's vs. women's, these should be split into multiple merch items as well.
 - Variants may be priced differently from each other, which can be used to cover higher costs for more rare sizes, or even to sell multi-packs of items for a discount.
 - If in doubt, a good rule of thumb is to group together items as variants if one picture can adequately represent them all.

Merchbar supports basic HTML in the merchandise `description` field. The `photos` field should hold an array of product image URLs sorted by preferred order. The `status` field should be one of `"available"`, `"backorder"`, or `"preorder"` with an `available_date` value returned for the latter two fields (if known). If the item should not be shown at all, be sure to send `"visible": false`. Merch without any variants should omit the `variants` field completely. The `sale_price` parameter allows you to indicate to us that an item typically sells for a given `price` but is currently on sale for less.

**Parameters:**

name | type | description
--- | --- | ---
`per_page` | `number` | the number of stores to include per page (optional)
`page` | `number` | the current page number (1-indexed, optional)

**Example Response:**

```json
{
    "data": [
        {
            "id": "SKU123",
            "title": "Merch Item #1",
            "description": "A beautiful piece of merchandise for you, with variants!",
            "color": "black",
            "price": "15.99",
            "currency": "USD",
            "quantity": 100,
            "sale_price": null,
            "visible": true,
            "status": "available",
            "available_date": null,
            "photos": [
                "https://dt72k0ec4onep.cloudfront.net/JYCH091425558193.JPG",
                "https://dt72k0ec4onep.cloudfront.net/AQTSQDA425523214.JPG",
                "https://dt72k0ec4onep.cloudfront.net/HOWJSDE773452342.JPG"
            ],
            "variants": [
                {
                    "id": "SKU123XS",
                    "title": "Extra Small",
                    "quantity": 20,
                    "price": "15.99",
                    "currency": "USD",
                    "sale_price": null
                },
                {
                    "id": "SKU123S",
                    "title": "Small",
                    "quantity": 20,
                    "price": "15.99",
                    "currency": "USD",
                    "sale_price": null
                },
                {
                    "id": "SKU123M",
                    "title": "Medium",
                    "quantity": 20,
                    "price": "15.99",
                    "currency": "USD",
                    "sale_price": null
                },
                {
                    "id": "SKU123L",
                    "title": "Large",
                    "quantity": 20,
                    "price": "15.99",
                    "currency": "USD",
                    "sale_price": null
                },
                {
                    "id": "SKU1232X",
                    "title": "Extra Large",
                    "quantity": 20,
                    "price": "15.99",
                    "currency": "USD",
                    "sale_price": null
                }
            ]
        },
        {
            "id": "SKU456",
            "title": "Merch Item #2",
            "description": "A beautiful piece of merchandise! Pre-order to get yours first.",
            "color": "white",
            "price": "25.00",
            "currency": "USD",
            "quantity": 20,
            "sale_price": "22.00",
            "visible": true,
            "status": "preorder",
            "available_date": "2016-12-25",
            "photos": ["https://dt72k0ec4onep.cloudfront.net/JYCH091425558193.JPG"]
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 50,
        "total_pages": 1
    }
}
```

## POST `/v1/order_previews`

Returns an order preview, which in most cases is just a reiteration of the input line items along with a total shipping cost estimate. If an item is no longer available, make sure to return `"available": false` for that line item, but return a shipping estimate as if that item was not included in the original line items at all (or, as if its quantity was set to `0`).

Note: Merchbar offers international shipping with a number of our partners. If you do not support international shipping, let us know, and also make sure to return an error response here if a country other than `"US"` is specified.

**Example Request Body:**

```json
{
    "address": {
        "first_name": "Loyal",
        "last_name": "Customer",
        "line1": "123 Main St.",
        "line2": null,
        "city": "San Francisco",
        "state": "CA",
        "postal_code": "94102",
        "country": "US"
    },
    "line_items": [
        {
            "id": "SKU456",
            "quantity": 2
        },
        {
            "id": "SKU123",
            "variant_id": "SKU1232X",
            "quantity": 1
        }
    ]
}
```

**Example Response:**

```json
{
    "data": {
        "address": {
            "first_name": "Loyal",
            "last_name": "Customer",
            "line1": "123 Main St.",
            "line2": null,
            "city": "San Francisco",
            "state": "CA",
            "postal_code": "94102",
            "country": "US"
        },
        "line_items": [
            {
                "id": "SKU456",
                "quantity": 2,
                "available": true
            },
            {
                "id": "SKU123",
                "variant_id": "SKU1232X",
                "quantity": 1,
                "available": true
            }
        ],
        "shipping_options": [
            {
                "id": "USPSPRI",
                "title": "USPS Priority Mail",
                "ship_date": "2016-08-01",
                "ship_time": "2-3 business days",
                "price": "19.98",
                "currency": "USD"
            },
            {
                "id": "USPSFC",
                "title": "USPS First-Class Mail",
                "ship_date": "2016-08-01",
                "ship_time": "5-6 business days",
                "price": "6.98",
                "currency": "USD"
            },
            {
                "id": "USPSEXP",
                "title": "USPS Priority Mail Express",
                "ship_date": "2016-08-01",
                "ship_time": "Overnight",
                "price": "36.22",
                "currency": "USD"
            }
        ]
    }
}
```

## POST `/v1/orders`

Creates a new order. If any requested item is no longer available, the entire order should fail.

The `status` field on the returned order can be set to any string. Partners should choose values here based on their unique fulfillment process. Please supply Merchbar a list of possible values this field can take.

Note: Merchbar offers international shipping with a number of our partners. If you do not support international shipping, let us know, but also make sure to return an error response here if a country other than `"US"` is specified.

**Example Request Body:**

```json
{
    "address": {
        "first_name": "Loyal",
        "last_name": "Customer",
        "line1": "123 Main St.",
        "line2": null,
        "city": "San Francisco",
        "state": "CA",
        "postal_code": "94102",
        "country": "US"
    },
    "line_items": [
        {
            "id": "SKU456",
            "quantity": 2
        },
        {
            "id": "SKU123",
            "variant_id": "SKU1232X",
            "quantity": 1
        }
    ],
    "selected_shipping_option_id": "USPSFC"
}
```

**Example Response:**

```json
{
    "data": {
        "id": "CONF128472-294",
        "created": "2016-08-01T13:12:11Z",
        "status": "RECEIVED",
        "tracking_number": null,
        "address": {
            "first_name": "Loyal",
            "last_name": "Customer",
            "line1": "123 Main St.",
            "line2": null,
            "city": "San Francisco",
            "state": "CA",
            "postal_code": "94102",
            "country": "US"
        },
        "line_items": [
            {
                "id": "SKU456",
                "quantity": 2,
                "available": true
            },
            {
                "id": "SKU123",
                "variant_id": "SKU1232X",
                "quantity": 1,
                "available": true
            }
        ],
        "shipping": {
            "id": "USPSFC",
            "title": "USPS First-Class Mail",
            "ship_date": "2016-08-01",
            "ship_time": "5-6 business days",
            "price": "6.98",
            "currency": "USD"
        }
    }
}
```

## GET `/v1/orders/{order_id}`

Returns the order status

**Parameters:** None

**Example Response:**

```json
{
    "data": {
        "id": "CONF128472-294",
        "created": "2016-08-01T13:12:11Z",
        "status": "SHIPPED",
        "tracking_number": "9361289878166149570033",
        "address": {
            "first_name": "Loyal",
            "last_name": "Customer",
            "line1": "123 Main St.",
            "line2": null,
            "city": "San Francisco",
            "state": "CA",
            "postal_code": "94102",
            "country": "US"
        },
        "line_items": [
            {
                "id": "SKU456",
                "quantity": 2,
                "available": true
            },
            {
                "id": "SKU123",
                "variant_id": "SKU1232X",
                "quantity": 1,
                "available": true
            }
        ],
        "shipping": {
            "id": "USPSFC",
            "title": "USPS First-Class Mail",
            "ship_date": "2016-08-01",
            "ship_time": "5-6 business days",
            "price": "6.98",
            "currency": "USD"
        }
    }
}
```


## PATCH `/v1/orders/{order_id}`

Updates the shipping address for a given order. If changing the address is not allowed, or if changing the address would change the shipping cost, an error should be returned instead. Merchbar will not send any other fields beyond `address` within this PATCH request.

Note that this is a **PATCH** request.

**Example Request Body:**

```json
{
    "address": {
        "first_name": "Loyal",
        "last_name": "Customer",
        "line1": "456 Side St.",
        "line2": null,
        "city": "San Francisco",
        "state": "CA",
        "postal_code": "94103",
        "country": "US"
    }
}
```

**Example Response:**

```json
{
    "data": {
        "id": "CONF128472-294",
        "created": "2016-08-01T13:12:11Z",
        "status": "RECEIVED",
        "tracking_number": null,
        "address": {
            "first_name": "Loyal",
            "last_name": "Customer",
            "line1": "456 Side St.",
            "line2": null,
            "city": "San Francisco",
            "state": "CA",
            "postal_code": "94103",
            "country": "US"
        },
        "line_items": [
            {
                "id": "SKU456",
                "quantity": 2,
                "available": true
            },
            {
                "id": "SKU123",
                "variant_id": "SKU1232X",
                "quantity": 1,
                "available": true
            }
        ],
        "shipping": {
            "id": "USPSFC",
            "title": "USPS First-Class Mail",
            "ship_date": "2016-08-01",
            "ship_time": "5-6 business days",
            "price": "6.98",
            "currency": "USD"
        }
    }
}
```

## POST `/v1/orders/{order_id}/cancel`

Cancels a given order. Merchbar will ignore the response body for this request, so long as it is not an error, but following REST principles the order object should be returned, potentially without fields that are no longer relevant after the order is cancelled.

**Request Body:** None

**Example Response:**

```json
{
    "data": {
        "id": "CONF128472-294",
        "created": "2016-08-01T13:12:11Z",
        "status": "CANCELLED",
        "line_items": [
            {
                "id": "SKU456",
                "quantity": 2,
                "available": true
            },
            {
                "id": "SKU123",
                "variant_id": "SKU1232X",
                "quantity": 1,
                "available": true
            }
        ]
    }
}
```
