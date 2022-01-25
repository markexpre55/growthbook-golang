# GrowthBook Go SDK

[GrowthBook](https://www.growthbook.io) is a modular Feature Flagging and Experimentation platform.

This is a Go package that lets you evaluate feature flags and run
experiments (A/B tests) within a Go application.

- **No external dependencies**
- **Lightweight and fast**
- Local targeting and evaluation, **no HTTP requests**
- **Use your existing event tracking** (GA, Segment, Mixpanel, custom)
- Run mutually exclusive experiments with **namespaces**


## Installation

```
go get github.com/growthbook/growthbook-golang
```


## Quick Usage

The main public API for the SDK is provided through the `Context` and
`GrowthBook` types. The `Context` type provides a means to pass
settings to the main `GrowthBook` type, while the `GrowthBook` type
provides a `Feature` method for accessing feature values, and a `Run`
method for running inline experiments.

```go
// Parse feature map from JSON data.
features := growthbook.ParseFeatureMap(jsonBody)

// Create context and main GrowthBook object.
context := growthbook.NewContext().WithFeatures(features)
gb := growthbook.New(context)

// Perform feature test.
if gb.Feature("my-feature").On {
  // ...
}

color := gb.Feature("signup-button-color").GetValueWithDefault("blue")
fmt.Println(color)

experiment :=
  growthbook.NewExperiment("my-experiment").
    WithVariations("A", "B")

result := gb.Run(experiment)

fmt.Println(result.Value)

experiment2 :=
  growthbook.NewExperiment("complex-experiment").
    WithVariations(
      map[string]string{"color": "blue", "size": "small"},
      map[string]string{"color": "green", "size": "large"},
    ).
    WithWeights(0.8, 0.2).
    WithCoverage(0.5)

result2 := gb.Run(experiment2)
fmt.Println(result2.Value.(map[string]string)["color"],
  result2.Value.(map[string]string)["size"])
```

## The GrowthBook Context

A new `Context` is created using the `NewContext` function and its
fields can be set using the `WithEnabled`, `WithAttributes`,
`WithURL`, `WithFeatures`, `WithForcedVariations`, `WithQAMode` and
`WithTrackingCallback` methods. These `With...` methods return a
`Context` pointer to enable call chaining. Context details can
alternatively be parsed from JSON data (see [JSON data
representations](#json-data-representations)). The fields in a
`Context` include information about the user for whom feature results
will be evaluated (the `Attributes`), the features that are defined,
plus some additional values to control forcing of feature results
under some circumstances.

Given a `Context` value, a new `GrowthBook` value can be created using
the `New` function. The `GrowthBook` type has some getter and setter
methods (setters are methods with names of the form `With...`) for
fields of the associated `Context`. As well as providing access to the
underlying Context and exposing the main `Feature` and `Run` methods,
the `GrowthBook` type also keeps track of the results of experiments
that are performed, in order to implement tracking and experiment
subscription callbacks.

For example, assuming that the `growthbook` package is imported with
name "`growthbook`", the following code will create a `Context` and
`GrowthBook` value using features parsed from JSON data and some fixed
attributes:

```go
// Parse feature map from JSON.
features := growthbook.ParseFeatureMap(featureJSON)

// Create context and main GrowthBook object.
context := growthbook.NewContext().
  WithFeatures(features).
  WithAttributes(growthbook.Attributes{
    "country": "US",
    "browser": "firefox",
  })
gb := growthbook.New(context)
```

### Features

The `WithFeatures` method of `Context` takes a `FeatureMap` value,
which is defined as `map[string]*Feature`, and which can be created
from JSON data using the `ParseFeatureMap` function (see [JSON data
representations](#json-data-representations)). You can pass a feature
map generated this way to the `WithFeatures` method of `Context` or
`GrowthBook`:

```go
featureMap := ParseFeatureMap([]byte(
  `{
     "features": {
       "feature-1": {...},
       "feature-2": {...},
       "another-feature": {...},
     }
   }`))

gb := NewContext().WithFeatures(featureMap)
```

If you need to load feature definitions from a remote source like an
API or database, you can update the context at any time with
`WithFeatures`.

If you use the GrowthBook App to manage your features, you don't need
to build this JSON file yourself -- it will auto-generate one for you
and make it available via an API endpoint.

If you prefer to build this file by hand or you want to know how it
works under the hood, check out the detailed [Feature
Definitions](#feature-definitions) section below.

### Attributes

You can specify attributes about the current user and request. These
are used for two things:

 * Feature targeting (e.g. paid users get one value, free users get
   another);
   
 * Assigning persistent variations in A/B tests (e.g. user id "123"
   always gets variation B).

Attributes can be any JSON data type -- boolean, integer, string,
array, or object and are represented by the `Attributes` type, which
is an alias for the generic `map[string]interface{}` type that Go uses
for JSON objects. If you know them up front, you can pass them into
`Context` or `GrowthBook` using `WithAttributes`:

```go
gb := growthbook.New(context).
  WithAttributes(Attributes{
    "id":       "123",
    "loggedIn": true,
    "deviceId": "abc123def456",
    "company":  "acme",
    "paid":     false,
    "url":      "/pricing",
    "browser":  "chrome",
    "mobile":   false,
    "country":  "US",
  })
```

You can also set or update attributes asynchronously at any time with
the `WithAttributes` method. This will completely overwrite the
attributes object with whatever you pass in. If you want to merge
attributes instead, you can get the existing ones with `Attributes`:

```go
attrs := gb.Attributes()
attrs["url"] = "/checkout"
gb.WithAttributes(attrs)
```

Be aware that changing attributes may change the assigned feature
values. This can be disorienting to users if not handled carefully. A
common approach is to only refresh attributes on navigation, when the
window is focused, and/or after a user performs a major action like
logging in.

### Tracking Callback

Any time an experiment is run to determine the value of a feature, we
can run a callback function so you can record the assigned value in
your event tracking or analytics system of choice.

```go
context.WithTrackingCallback(func(experiment *growthbook.Experiment,
  result *growthbook.ExperimentResult) {
    // Example using Segment.io
    client.Enqueue(analytics.Track{
      UserId: context.Attributes()["id"],
      Event: "Experiment Viewed",
      Properties: analytics.NewProperties().
        Set("experimentId", experiment.Key).
        Set("variationId", result.VariationID)
    })
  }
)
```


## Using Features

Given a `GrowthBook` object, feature results can be retrieved using
the `Feature` method. This takes the name of a feature as a parameter,
and uses the stored feature definitions and attributes to evaluate
whether the feature is active and what the feature value is. For
example:

```go
if gb.Feature("test-feature").On {
	...
}
```

Defaulting of the values of feature results is assisted by the
`GetValueWithDefault` method on the `FeatureResult` type. For example,
this code evaluates the result of a feature and returns the feature
value, defaulting to "blue" if the feature has no value:

```go
color := gb.Feature("signup-button-color").GetValueWithDefault("blue")
```

## Feature Definitions

### Basic Feature

#### Default Values

### Override Rules

#### Rule Conditions

#### Force Rules

##### Gradual Rollouts

#### Experiment Rules

##### Weights

##### Tracking Key

##### Hash Attribute

##### Coverage

##### Namespaces



## Inline Experiments

Experiments can be defined and run using the `Experiment` type and the
`Run` method of the `GrowthBook` type. Experiment definitions can be
created directly as values of the `Experiment` type, or parsed from
JSON definitions using the `ParseExperiment` function. Passing an
`Experiment` value to the `Run` method of the `GrowthBook` type will
run the experiment, returing an `ExperimentResult` value that contains
the resulting feature value. This allows users to run arbitrary
experiments without providing feature definitions up-front.

### Inline Experiment Return Value


## Error handling

The GrowthBook public API does not return errors under any normal
circumstances. The intention is for developers to be able to use the
SDK in both development and production smoothly. To this end, error
reporting is provided by a configurable logging interface.

For development use, the `DevLogger` type provides a suitable
implementation of the logging interface: it prints all logged messages
to standard output, and exits on errors.

For production use, a logger that directs log messages to a suitable
centralised logging facility and ignores all errors would be suitable.
The logger can of course also signal error and warning conditions to
other parts of the program in which it is used.

To be specific about this:

 * None of the functions that create or update `Context`, `GrowthBook`
   or `Experiment` values return errors.

 * The main `GrowthBook.Feature` and `GrowthBook.Run` methods never
   return errors.

 * None of the functions that create values from JSON data return
   errors.

For most common use cases, this means that the GrowthBook SDK can be
used transparently, without needing to care about error handling. Your
server code will never crash because of problems in the GrowthBook
SDK. The only effect of error conditions in the inputs to the SDK may
be that feature values and results of experiments are not what you
expect.


## JSON data representations

For interoperability of the GrowthBook Go SDK with versions of the SDK
in other languages, the core "input" values of the SDK (in particular,
`Context` and `Experiment` values and maps of feature definitions) can
be created by parsing JSON data. A common use case is to download
feature definitions from a central location as JSON, to parse them
into a feature map that can be applied to a GrowthBook `Context`, then
using this context to create a `GrowthBook` value that can be used for
feature tests.

A contrived example of how this might work is:

```go
// Download JSON feature file and read file body.
resp, err := http.Get("https://s3.amazonaws.com/myBucket/features.json")
if err != nil {
	log.Fatal(err)
}
defer resp.Body.Close()
body, err := ioutil.ReadAll(resp.Body)
if err != nil {
	log.Fatal(err)
}

// Parse feature map from JSON.
features, err := ParseFeatureMap(body)
if err != nil {
	log.Fatal(err)
}

// Create context and main GrowthBook object.
context := NewContext().WithFeatures(features)
growthbook := New(context)

// Perform feature test.
if growthbook.Feature("my-feature").On {
	// ...
}
```

The functions that implement this JSON processing functionality have
names like `ParseContext`, `BuildContext`, and so on. Each `Parse...`
function process raw JSON data (as a `[]byte` value), while the
`Build...` functions process JSON objects unmarshalled to Go values of
type `map[string]interface{}`. This provides flexibility in ingestion
of JSON data.


## Tracking and subscriptions

The `Context` value supports a "tracking callback", which is a
function that is called any time an experiment is run to determine the
value of a feature, so that users can record the assigned value in an
external event tracking or analytics system.

In addition to the tracking callback, the `GrowthBook` type also
supports more general "subscriptions", which are callback functions
that are called any time `Run` is called, irrespective of whether or
not a user is included in an experiment. The subscription system
ensures that subscription callbacks are only called when the result of
an experiment changes, or a new experiment is run.
