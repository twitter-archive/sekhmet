# Sekhmet

---

**What is this repo or project?**

  - For Twitter, the Sekhmet project represents the foundational tools and building blocks for gaining insights and diagnosing system health in real-time. The first arc of that work is the *Monitor Evaluator*. We are initially sharing the spec for community input, but will be sharing the source code and examples in the not too distant future.

**How does it work?**

  - There is a foundational *time series* core library and API model schema. By creating this, we allow for generalized *time series* data to interact with our evaluation and notification systems. The framework can be extended to support other *time series* sources, while retaining a shared monitor definition for evaluation. The API definitions also allow for supplying time series data to be evaluated via the API - for use as testing monitor behavior or as an alternative to integrating new *time series sources*. Note: the initial project will ship with a Prometheus *time series source* example.

**Who will use this repo or project?**

  - Anyone who is interested in evaluating, gaining insights, or reacting to *time series* data - our primary use case is in the Observability space (metrics, logs, and traces) of distributed services, but can also be applied to other applications.

**What is the goal of this project?**

  - Make monitoring systems dynamic, facilitate a common monitoring and alerting service that isn't tied to a specific vendor/solution, and give people tools to better understand the behavior and context of their systems *AND* their alerting system. 

**Where does the name come from?**

  - In Egyptian mythology, Sekhmet (means "the powerful one") is a warrior goddess as well as goddess of healing. She is depicted as a lioness, the fiercest hunter known to the Egyptians. It was said that her breath formed the desert. She was seen as the protector of the pharaohs and led them in warfare. [wikipedia](https://en.wikipedia.org/wiki/Sekhmet)

## Sections

1. [Monitor Evaluator](#Monitor%20Evaluator)
2. [API](#API)
3. [Examples](#Examples)
4. [Entities & Concepts](#Entities)
5. [Contributing](#Contributing)
6. [Support](#Support)
7. [Authors](#Authors)
8. [License](#License)
9. [Security](#Security%20Issues)

## Monitor Evaluator

---

 The Monitor Evaluator is an HTTP API for executing ad-hoc validation of a ‘monitor’. This project extracts some of the concepts from MonAlert (the periodic alert evaluation system, a component of our Next Gen Alerting System) for defining and building monitors and alerts, but enables some new functionality:

  - Supply custom time-series data to test out alert definition behavior
  - Supply an anchor time to adjust the time period that evaluation should occur (the default is 'now')
  - The 'source' is the key result that is surfaced - results of Aggregate Evaluable's bubble up this information
  - Support for evaluation of sub-secondly data - down to millisecondly
  - No 'data-source' lock in. We can extend the project to plug-in different data sources to do evaluation against the shared monitor definition. 
  - Query Cost Validation: A monitor will do preflight costing and surface errors when the monitor is too expensive. Cost information is included in detailed execution results.
  - Query Execution Time. Detailed results show how long each query took vs cost. This will improve visibility in understanding trends to make the alerting/querying systems more stable.
  - Evaluation Warnings: In MonAlert, not receiving enough data or hitting query errors resulted in a monitor entering a system defined level. This occasionally results in false pages, due to monitors transitioning in and out of user defined states. These are no longer 'levels' and will be surfaced in results as 'warnings'.
  - Notifications: You can supply notification destination bindings for each level and receive a notification with the monitor results for the sources in that level.
  - Data Consistency Checks: Evaluation will periodically re-evaluate and compare results as a metric 
  - Multiple Data Sources: You can define queries that retrieve data from different time series sources. Be careful to ensure that the ‘source_id’ generation is normalized between time series sources, otherwise monitors may not behave as expected.

## Support

File an issue on GitHub

## Authors

* Ian Bennett <https://github.com/enbnt>

A full list of [contributors](https://github.com/twitter/sekhmet/graphs/contributors?type=a) can be found on GitHub.

Follow [@twitteross](https://twitter.com/twitteross) on Twitter for updates.

## License

Copyright 2013-2018 Twitter, Inc.

Licensed under the Apache License, Version 2.0: https://www.apache.org/licenses/LICENSE-2.0

## Security Issues?

Please report sensitive security issues via Twitter's bug-bounty program (https://hackerone.com/twitter) rather than GitHub.

## API

The evaluation API contains a single, stateless POST request

```
HTTP POST /monitor/evaluator
```

__PLEASE NOTE__ this API is in flux and subject to change based on internal requirements or community feedback. When the API is stable, the URL will include a version (i.e. /1/monitor/evaluator) that will only support non-breaking API changes until the next version. Until that point, expect breaking changes and/or changes to this spec.

#### Query Params

`include_monitor_def` {boolean} [**default**=false]: Response includes monitor definition

`detail` {count,source,all} [**default**=count]: Level of detail to include in response

`order_by` {severity,count} [**default**=severity]: Result ordered by

`sort_desc` {boolean} [**default**=true]: Result sorted as descending

*Ex:* `POST /monitor/evaluator?include_monitor_def=true&detail=source`

*Ex:* `POST /monitor/evaluator?detail=all&order_by=count`

*Ex:* `POST /monitor/evaluator?detail=source&sort_desc=false`

## Examples

---

### A Simple Monitor

`POST /monitor/evaluator`

__Request Payload__

```json
{
  "level_ordering": [
    "(╯°□°）╯︵ ┻━┻)"
  ],
  "monitor": {
    "name": "Prometheus Uptime Test",
    "description": "The Prometheus instance is down for too long",
    "levels": [
      {
        "level": "(╯°□°）╯︵ ┻━┻)",
        "evaluable": {
          "type": "simple",
          "query": {
            "query": "up"
          },
          "predicate": {
            "operator": "!=",
            "threshold": 1,
            "for": 5,
            "of": 5,
            "interval": {
              "unit": "m"
            }
          }
        }
      }
    ]
  },
  "time_series_source": "prometheus"
}
```

---

__Response__

```json
{
  "anchor_time_ms": 1527704633490,
  "levels": [
    {
      "level": "(╯°□°）╯︵ ┻━┻)",
      "count": 0
    },
    {
      "level": "ok",
      "count": 1
    }
  ],
  "detection_time_ms": 1527704633557,
  "elapsed_ms": 61
}

```

### A Monitor With Multiple Levels

`POST /monitor/evaluator?detail=source`

__Request Payload__

```json
{
  "level_ordering": [
    "warn",
    "critical"
  ],
  "monitor": {
    "name": "Prometheus Uptime Test",
    "description": "The Prometheus instance is down for too long",
    "levels": [
      {
        "level": "warn",
        "evaluable": {
          "type": "simple",
          "query": {
            "query": "up"
          },
          "predicate": {
            "operator": "!=",
            "threshold": 1,
            "for": 5,
            "of": 5,
            "interval": {
              "unit": "m"
            }
          }
        }
      },
      {
        "level": "critical",
        "evaluable": {
          "type": "simple",
          "query": {
            "query": "up"
          },
          "predicate": {
            "operator": "!=",
            "threshold": 1,
            "for": 20,
            "of": 20,
            "interval": {
              "unit": "m"
            }
          }
        }
      }
    ]
  },
  "time_series_source": "prometheus"
}
```

__Response__

```json
{
  "anchor_time_ms": 1527704831294,
  "levels": [
    {
      "level": "critical",
      "count": 0,
      "sources": []
    },
    {
      "level": "warn",
      "count": 0,
      "sources": []
    },
    {
      "level": "ok",
      "count": 1,
      "sources": [
        "prometheus.localhost:9090"
      ]
    }
  ],
  "detection_time_ms": 1527704831301,
  "elapsed_ms": 9
}
```

### A Monitor With Different Queries

`POST /monitor/evaluator?detail=all`

__Request Body__

```json
{
  "level_ordering": [
    "warn",
    "critical"
  ],
  "monitor": {
    "name": "Prometheus Query Range Test",
    "description": "The Prometheus 'query_range' latency is higher than expected",
    "levels": [
      {
        "level": "warn",
        "evaluable": {
          "type": "simple",
          "query": {
            "query": "http_request_duration_microseconds{handler=\"query_range\",quantile=\"0.9\"}"
          },
          "predicate": {
            "operator": ">",
            "threshold": 100,
            "for": 5,
            "of": 10,
            "interval": {
              "unit": "m"
            }
          }
        }
      },
      {
        "level": "critical",
        "evaluable": {
          "type": "simple",
          "query": {
            "query": "http_request_duration_microseconds{handler=\"query_range\",quantile=\"0.99\"}"
          },
          "predicate": {
            "operator": ">",
            "threshold": 500,
            "for": 15,
            "of": 20,
            "interval": {
              "unit": "m"
            }
          }
        }
      }
    ]
  },
  "time_series_source": "prometheus"
}
```

__Response__

```json
{
  "anchor_time_ms": 1527705028566,
  "levels": [
    {
      "level": "critical",
      "count": 0,
      "sources": [],
      "details": {
        "type": "simple",
        "trigger": false,
        "sources": [
          {
            "source": "prometheus.localhost:9090",
            "trigger": false,
            "ts": {
              "type": "sparse",
              "source_id": "prometheus.localhost:9090",
              "data": [
                [
                  1527704160000,
                  "NaN"
                ],
                [
                  1527704220000,
                  "NaN"
                ],
                [
                  1527704280000,
                  "NaN"
                ],
                [
                  1527704340000,
                  "NaN"
                ],
                [
                  1527704400000,
                  "NaN"
                ],
                [
                  1527704460000,
                  "NaN"
                ],
                [
                  1527704520000,
                  "NaN"
                ],
                [
                  1527704580000,
                  "NaN"
                ],
                [
                  1527704640000,
                  1056.2
                ],
                [
                  1527704700000,
                  1056.2
                ],
                [
                  1527704760000,
                  1056.2
                ],
                [
                  1527704820000,
                  1056.2
                ],
                [
                  1527704880000,
                  499.8
                ],
                [
                  1527704940000,
                  499.8
                ],
                [
                  1527705000000,
                  499.8
                ]
              ],
              "interval": {
                "unit": "minutes"
              },
              "context": {
                "quantile": "0.99",
                "instance": "localhost:9090",
                "job": "prometheus",
                "__name__": "http_request_duration_microseconds"
              }
            }
          }
        ],
        "cost": 0,
        "query_elapsed_ms": 4
      }
    },
    {
      "level": "warn",
      "count": 1,
      "sources": [
        "prometheus.localhost:9090"
      ],
      "details": {
        "type": "simple",
        "trigger": true,
        "triggered": [
          "prometheus.localhost:9090"
        ],
        "sources": [
          {
            "source": "prometheus.localhost:9090",
            "trigger": true,
            "ts": {
              "type": "sparse",
              "source_id": "prometheus.localhost:9090",
              "data": [
                [
                  1527704340000,
                  "NaN"
                ],
                [
                  1527704400000,
                  "NaN"
                ],
                [
                  1527704460000,
                  "NaN"
                ],
                [
                  1527704520000,
                  "NaN"
                ],
                [
                  1527704580000,
                  "NaN"
                ],
                [
                  1527704640000,
                  1056.2
                ],
                [
                  1527704700000,
                  1056.2
                ],
                [
                  1527704760000,
                  1056.2
                ],
                [
                  1527704820000,
                  1056.2
                ],
                [
                  1527704880000,
                  499.8
                ],
                [
                  1527704940000,
                  499.8
                ],
                [
                  1527705000000,
                  499.8
                ]
              ],
              "interval": {
                "unit": "minutes"
              },
              "context": {
                "quantile": "0.9",
                "instance": "localhost:9090",
                "job": "prometheus",
                "__name__": "http_request_duration_microseconds"
              }
            }
          }
        ],
        "cost": 0,
        "query_elapsed_ms": 7
      }
    },
    {
      "level": "ok",
      "count": 0,
      "sources": []
    }
  ],
  "detection_time_ms": 1527705028574,
  "elapsed_ms": 17
}
```

### A Monitor with Supplied Data

`POST /monitor/evaluator?detail=source`

__Request Body__

```json
{
  "anchor_time": 1527465342470, //set a time to evaluate against
  "level_ordering": [
    "warn",
    "critical"
  ],
  "monitor": {
    "name": "Prometheus Uptime Test",
    "description": "The Prometheus instance is down for too long",
    "levels": [
      {
        "level": "warn",
        "evaluable": {
          "type": "simple",
          "query": {
            "query": "up",
            "interval": {"unit": "m"},
            "source": "prometheus" // you can specify a source per query
          },
          "predicate": {
            "operator": "!=",
            "threshold": 1,
            "for": 5,
            "of": 5,
            "interval": {
              "unit": "m"
            }
          }
        }
      },
      {
        "level": "critical",
        "evaluable": {
          "type": "simple",
          "query": {
            "query": "up", //because these queries are the same, the system optimizes them
            "interval": {"unit": "m"},
            "source": "prometheus"
          },
          "predicate": {
            "operator": "!=",
            "threshold": 1,
            "for": 20,
            "of": 20,
            "interval": {
              "unit": "m"
            }
          }
        }
      }
    ]
  },
  "data": [ //supply data instead of querying against the real data source
    {
      "query": {
        "query": "up",
        "interval": {
          "unit": "m"
        },
        "source": "prometheus"
      },
      "data": [
        {
          "type": "dense",
          "source_id": "prometheus:9090.0", //simulate a larger cluster - instance #0
          "start_time": 1527464100000,
          "values": [1,1,1,1,1,1,1,1,0,1,1,1,1,1,1,0,0,0,0,0],
          "interval": {
            "unit": "m"
          }
        },
        {
          "type": "dense",
          "source_id": "prometheus:9090.1", //by specifying - instance #1
          "start_time": 1527464100000,
          "values": [1,1,1,1,1,1,1,1,0,1,1,1,1,1,1,0,1,1,0,0],
          "interval": {
            "unit": "m"
          }
        },
        {
          "type": "dense",
          "source_id": "prometheus:9090.2", //multiple sources - instance #2
          "start_time": 1527464100000,
          "values": [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
          "interval": {
            "unit": "m"
          }
        },
        {
          "type": "dense",
          "source_id": "prometheus:9090.3", // instance #3
          "start_time": 1527464100000,
          "values": [1,1,1,1,1,1,1,1,0,1,1,1,1,1,1,0,0,0,0,0], //provide data that you know will enter a level - this should be 'critical'
          "interval": {
            "unit": "m"
          }
        }
      ],
      "start_time": 1527464100000,
      "end_time": 1527465300000
    }
  ]
}
```

__Response__

```json
{
  "anchor_time_ms": 1527465342470,
  "levels": [
    {
      "level": "critical",
      "count": 1,
      "sources": [
        "prometheus:9090.2"
      ]
    },
    {
      "level": "warn",
      "count": 2,
      "sources": [
        "prometheus:9090.0",
        "prometheus:9090.3"
      ]
    },
    {
      "level": "ok",
      "count": 1,
      "sources": [
        "prometheus:9090.1"
      ]
    }
  ],
  "detection_time_ms": 1527705196686,
  "elapsed_ms": 4
}
```

### A Monitor with Supplied Data that sends test Notifications

`POST /monitor/evaluator?detail=source`

__Request Body__

```json
{
  "anchor_time": 1527465342470,
  "level_ordering": [
    "warn",
    "critical"
  ],
  "monitor": {
    "name": "Prometheus Uptime Test",
    "description": "The Prometheus instance is down for too long",
    "levels": [
      {
        "level": "warn",
        "evaluable": {
          "type": "simple",
          "query": {
            "query": "up",
            "interval": {"unit": "m"},
            "source": "prometheus"
          },
          "predicate": {
            "operator": "!=",
            "threshold": 1,
            "for": 5,
            "of": 5,
            "interval": {
              "unit": "m"
            }
          }
        }
      },
      {
        "level": "critical",
        "evaluable": {
          "type": "simple",
          "query": {
            "query": "up",
            "interval": {"unit": "m"},
            "source": "prometheus"
          },
          "predicate": {
            "operator": "!=",
            "threshold": 1,
            "for": 20,
            "of": 20,
            "interval": {
              "unit": "m"
            }
          }
        }
      }
    ]
  },
  "data": [
    {
      "query": {
        "query": "up",
        "interval": {
          "unit": "m"
        },
        "source": "prometheus"
      },
      "data": [
        {
          "type": "dense",
          "source_id": "prometheus:9090.0",
          "start_time": 1527464100000,
          "values": [1,1,1,1,1,1,1,1,0,1,1,1,1,1,1,0,0,0,0,0],
          "interval": {
            "unit": "m"
          }
        },
        {
          "type": "dense",
          "source_id": "prometheus:9090.1",
          "start_time": 1527464100000,
          "values": [1,1,1,1,1,1,1,1,0,1,1,1,1,1,1,0,1,1,0,0],
          "interval": {
            "unit": "m"
          }
        },
        {
          "type": "dense",
          "source_id": "prometheus:9090.2",
          "start_time": 1527464100000,
          "values": [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
          "interval": {
            "unit": "m"
          }
        },
        {
          "type": "dense",
          "source_id": "prometheus:9090.3",
          "start_time": 1527464100000,
          "values": [0,1,0,1,0,1,0,1,0,1,1,1,1,1,1,0,0,0,0,0],
          "interval": {
            "unit": "m"
          }
        }
      ],
      "start_time": 1527464100000,
      "end_time": 1527465300000
    }
  ], //same as before, but now we add notifications!
  "notifications": [
    {
      "level": "warn",
      "destinations": [ //define multiple destination types for a specific level
        {
          "type": "email",
          "to": [
            "{your_email_address}"
          ]
        },
        {
          "type": "webhook",
          "url": "http://localhost:8080/webhook"
        }
      ]
    },
    {
      "level": "critical",
      "destinations": [
        {
          "type": "email",
          "to": [
            "{your_team}"
          ],
          "cc": [
            "{cc1}, {cc2}"
          ],
          "bcc": [
            "{bcc}"
          ]
        },
        {
          "type": "pagerduty",
          "pagerduty_key": "{yourPDKey}"
        }
      ]
    }
  ]
}
```

__Response__

```json
{
  "anchor_time_ms": 1527465342470,
  "levels": [
    {
      "level": "critical",
      "count": 1,
      "sources": [
        "prometheus:9090.2"
      ]
    },
    {
      "level": "warn",
      "count": 2,
      "sources": [
        "prometheus:9090.0",
        "prometheus:9090.3"
      ]
    },
    {
      "level": "ok",
      "count": 1,
      "sources": [
        "prometheus:9090.1"
      ]
    }
  ],
  "detection_time_ms": 1527705344717,
  "elapsed_ms": 1636,
  "notification_errors": [], //surface whether any errors were encountered when sending notifications
  "consistency_test_passed": true
}
```

__Notification Content__

![Warning Email](https://github.com/twitter/sekhmet/blob/master/supplied_data_warn_email.png)

![Critical Email](https://github.com/twitter/sekhmet/blob/master/supplied_data_critical_email.png)

### A Monitor with Real Notifications

`POST /monitor/evaluator?detail=source`

__Request Body__

```json
{
  "level_ordering": [
    "warn",
    "critical"
  ],
  "monitor": {
    "name": "Prometheus Query Range Test",
    "description": "The Prometheus 'query_range' latency is higher than expected",
    "levels": [
      {
        "level": "warn",
        "evaluable": {
          "type": "simple",
          "query": {
            "query": "http_request_duration_microseconds{handler=\"query_range\",quantile=\"0.9\"}"
          },
          "predicate": {
            "operator": ">",
            "threshold": 100,
            "for": 5,
            "of": 10,
            "interval": {
              "unit": "m"
            }
          }
        }
      },
      {
        "level": "critical",
        "evaluable": {
          "type": "simple",
          "query": {
            "query": "http_request_duration_microseconds{handler=\"query_range\",quantile=\"0.99\"}"
          },
          "predicate": {
            "operator": ">",
            "threshold": 500,
            "for": 15,
            "of": 20,
            "interval": {
              "unit": "m"
            }
          }
        }
      }
    ]
  },
  "time_series_source": "prometheus",
  "notifications": [
    {
      "level": "critical",  //only notify if we enter 'critical' level!
      "destinations": [
        {
          "type": "email",
          "to": [
            "{email_address}"
          ]
        }
      ]
    }
  ]
}

__Response__

```json
{
  "anchor_time_ms": 1527465342470,
  "levels": [
    {
      "level": "critical",
      "count": 1,
      "sources": [
        "prometheus:9090.2"
      ]
    },
    {
      "level": "warn",
      "count": 2,
      "sources": [
        "prometheus:9090.0",
        "prometheus:9090.3"
      ]
    },
    {
      "level": "ok",
      "count": 1,
      "sources": [
        "prometheus:9090.1"
      ]
    }
  ],
  "detection_time_ms": 1527705769139,
  "elapsed_ms": 2097,
  "notification_errors": [],
  "consistency_test_passed": true
}
```

__Notification Content__

![Critical Email](https://github.com/twitter/sekhmet/blob/master/real_data_critical_email.png)

![Critical Email](https://github.com/twitter/sekhmet/blob/master/real_data_critical_email2.png)


### A Monitor containing Aggregate Evaluation

`POST /monitor/evaluator?detail=source`

__Request Body__


```json
{
  "level_ordering": [
    "(╯°□°）╯︵ ┻━┻)"
  ],
  "monitor": {
    "name": "GC impacting latency",
    "description": "The prometheus instance is hitting excessive GC, which is impacting end user latency",
    "levels": [
      {
        "level": "(╯°□°）╯︵ ┻━┻)",
        "evaluable": {
          "type": "aggregate",
          "aggregator": "and_on_source", //and will only trigger if the predicate holds true for all on same source
          "children": [
            {
              "type": "simple",
              "query": {
                "query": "http_request_duration_microseconds{handler=\"query_range\",quantile=\"0.99\"}"
              },
              "predicate": {
                "operator": ">",
                "threshold": 100,
                "for": 5,
                "of": 5,
                "interval": {
                  "value": 15,
                  "unit": "s"
                }
              }
            },
            {
              "type": "simple",
              "query": {
                "query": "go_gc_duration_seconds{quantile=\"1\"}"
              },
              "predicate": {
                "operator": ">",
                "threshold": 100,
                "for": 2,
                "of": 5,
                "interval": {
                  "value": 15, //our interval is 15 seconds between data points!
                  "unit": "s"
                }
              }
            }
          ]
        }
      }
    ]
  },
  "time_series_source": "prometheus"
}
```

__Response__

```json
{
  "anchor_time_ms": 1527705900017,
  "levels": [
    {
      "level": "(╯°□°）╯︵ ┻━┻)",
      "count": 0,
      "sources": []
    },
    {
      "level": "ok",
      "count": 1,
      "sources": [
        "prometheus.localhost:9090"
      ]
    }
  ],
  "detection_time_ms": 1527705900040,
  "elapsed_ms": 77
}
```

## Entities

Here is a detailed look at the entities that make up our API's JSON schema.

### Time Series

A Time Series is a sequence or series of (time, value) pairs. Our common time series format allows for multiple representations of this data. This is to help keep the API human readable, while also allowing for potential optimizations of response size to reduce latency of evaluation. All TimeSeries are expected to supply a 'source_id' - the source identifier for where the metric was emitted - and allow for optional custom context data, for surfacing Time Series Database specific information.

This common Time Series data format is used for representing results, as well as for passing in data for evaluation via the API.

#### Sparse

A 'Sparse' Time Series represents a time series where the majority of the values are expected to NOT be present for a specific time interval. This format is probably the most commonly used representation for Time Series Databases, so may be the most familiar. This format requires that all (time, value) pairs be explicitly stated.

Data Points are represented as an array of [time, value], where time is in epoch milliseconds and the value is expected to be a floating point value.

```json
{
  "type": "sparse",
  "source_id": "prometheus.localhost:9090",
  "data": [
    [
      1527704640000,
      1056.2
    ],
    [
      1527704700000,
      1056.2
    ],
    [
      1527704760000,
      1056.2
    ],
    [
      1527704820000,
      1056.2
    ],
    [
      1527704880000,
      499.8
    ],
    [
      1527704940000,
      499.8
    ],
    [
      1527705000000,
      499.8
    ]
  ],
  "interval": {
    "unit": "minutes"
  },
  "context": {
    "quantile": "0.9",
    "instance": "localhost:9090",
    "job": "prometheus",
    "__name__": "http_request_duration_microseconds"
  }
}

```

#### Dense

A 'Dense' Time Series represents a time series where the majority of values are expected to be present for a specific time interval. This allows for the timestamp to be implicitly stated in the data, reducing verbosity of the API. This should be the preferred format for humans to send data to the API.

Note: The 'NaN' value represents an undefined/missing data point for a given time interval, that the system will ignore/skip.

```json
{
  "type": "dense",
  "source_id": "prometheus:9090.3",
  "start_time": 1527464100000,
  "values": [0,1,"NaN",1,0,1,0,1,0,1,1,1,1,1,1,0,0,0,0,0],
  "interval": {
    "unit": "m"
  },
  "context": {
    "instance": "localhost:9090",
    "job": "prometheus",
    "__name__": "up"
  }
}

```

#### Time Interval

```json
{
  "unit": "m", //one of 'ns', 'us', 'ms', 's', 'm', 'h', 'd'
  "value": 1 //optional: default value of 1
}
```

#### Query

```json
{
  "query": "query string for time series source",
  "interval": "<Time Interval>", //optional - default's to predicate's interval for a Simple Evaluable, otherwise required
  "source": "prometheus" //optional - time series source to specify per query. default's to evaluation request's supplied time_series_source, otherwise required.
}
```

#### Query Result

```json
{
  "query": "<Query>", //see Query schema above
  "data": ["<TimeSeries"], //see TimeSeries schema above
  "start_time": 0000, //epoch milliseconds
  "end_time": 0000 //epoch milliseconds
}
```

### Monitor

```json
"monitor": {
    "name": "Prometheus Uptime Test",
    "description": "The Prometheus instance is down for too long",
    "levels": [
      {
        "level": "warn",
        "evaluable": "<Evaluable>" //see Evaluable schema
      },
      {
      	"level": "critical",
      	"evaluable": "<Evaluable>" //see Evaluable schema
      }
    ]
  }
```

#### Evaluable

Evaluable is an abstract type, meaning something that can be evaluated. There is currently support for two types, Simple and Aggregate.

##### Simple

A 'Simple' Evaluable is explicitly tied to a query and defines a static threshold

```json
{
  "type": "simple",
  "query": "up", //query string to be used
  "predicate": {  //this is the equivalent of "<= 99.9 for 3 of 10m"
    "operator": "<=", //one of '<', '>', '<=', '>=', '=', '!='
    "threshold": 99.9,
    "for": 3, //optional: defaults to 1
    "of": 10, //optional: defaults to 'for''s value
    "interval": {"unit": "m"} //see interval schema for more
  }
}
```

##### Aggregate

An 'Aggregate' Evaluable is made up of child Evaluable types, where the 'aggregator' function is applied to the results.

```json
{
  "type": "aggregate",
  "aggregator": "and_on_source",  //one of {'and', 'or', 'and_on_source'}
  "children": ["<Evaluable>"] //abstract evaluable type allows for nesting
}
```

### Notifications

Notifications allow for sending context and data from an evaluation to a set of destinations - per level. Below are some examples of configuring different destination types:

#### Email

Send an email notification

```json
{
  "type": "email",
  "to": [ //must supply at least 1 'to' address
    "to_address"
  ],
  "cc": [ //optional
    "cc1", "cc2"
  ],
  "bcc": [ //optional
    "bcc"
  ]
}
```

#### Pagerduty

Send a notification to Pagerduty's API at the specified integration key. More details on integrations to come.

```json
{
  "type": "pagerduty",
  "pagerduty_key": "{yourPDKey}"
}
```

#### Webhook

Send an Evaluation Result payload to a URL

```json
{
  "type": "webhook",
  "url": "http://localhost:8080/webhook"
}
```
