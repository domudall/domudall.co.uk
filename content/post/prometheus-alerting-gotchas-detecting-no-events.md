+++
comments = true
date = "2016-10-24T18:00:00+01:00"
draft = false
menu = ""
share = true
slug = "prometheus-alerting-gotchas-detecting-no-events"
tags = ["golang", "monitoring", "gcp", "prometheus"]
title = "Prometheus Alerting Gotchas - Detecting No Events"
image = "https://prometheus.io/assets/jumbotron-background-cb04e0f2d6d.png"

+++

[Prometheus](https://prometheus.io) has become our de facto monitoring tool at Fresh8 for a number of reasons; it's [built in golang and open source](https://github.com/prometheus/prometheus), it's built by [SoundCloud](https://developers.soundcloud.com/blog/prometheus-monitoring-at-soundcloud), and it has a concept of labels which, in my mind, makes it far superior to statsd.

There are some areas, however, that seem a bit more tricky to work with. Prometheus allows you to set up a number of [alerts](https://prometheus.io/docs/alerting/overview/) using [rulesets](https://prometheus.io/docs/alerting/rules/), and has a separate [Alert Manager](https://prometheus.io/docs/alerting/alertmanager/) to monitor these alerts separate to Prometheus itself. This communicates with 3rd party tools such as Slack and PagerDuty so that the alerting is not coupled with the monitoring itself. The documentation on the setup of this entire system isn't as thorough/easy to follow as we initially thought it would be (future blog post idea - tick).

Now that we do have it set up, I'd like to cover a fairly simple alerting scenario that can catch out those unfamiliar with the querying within Prometheus. The scenario we're alerting on is when an API receives no events over a 5 minute period; this for us implies that the service is dead or incorrectly set up, as even in our quietest period we will get around 100 requests per minute. This service is also autoscaled based on throughput, therefore instances in this case are be transient. Our initial alert rule for this looked as follows:

```
ALERT ApiNoEvents
IF sum(rate(api_request_count{error=""}[5m])) by (instance) < 1
FOR 5m
LABELS { severity = "critical" }
ANNOTATIONS {
 summary = "No events into API over threshold period: {{ $labels.instance }}",
 description = "{{ $labels.instance }} API has received no events over 5mins"
}
```

The main part to be aware of is the rule query:

```
sum(rate(api_request_count{error=""}[5m])) by (instance) < 1
```

This looks at the rate of the total number of requests entering the service (that are not erroring) over a period of 5 minutes. If an instance of the service stops receiving requests, the data value will drop to 0, and trigger an alert. All good, however, a problem exists when all the services currently operating die.

Due to the fact we're using polling a metric endpoint for services, rather than pushing metrics from the services, when all the services are terminated, no metrics are obtained. This then means the `api_request_count` metric no longer exists, and therefore will not provide a numeric value based on a query, meaning the above is useless. To rectify this, two queries must be used as part of the alert; a check for the value being above 1 over the 5 minute period, and a check to make sure the metric is not absent during the 5 minute period, which is achieved using the [absent function](https://prometheus.io/docs/querying/functions/#absent()). This is what the full rule looks like:

```
sum(rate(api_request_count{error=""}[5m])) by (instance) < 1 OR absent(sum(rate(api_request_count[5m]))) == 1
```

This will check that every instance is receiving data (checking the load balancing is working/requests are hitting the service itself), as well as making sure at least one instance of the service is reporting metrics. Of course, for more verbosity we could split this into two separate alerts.

We're going to be setting up a lot more alerts over the coming weeks, so any more little gotchas or alerting tips, I'll post them up here.
