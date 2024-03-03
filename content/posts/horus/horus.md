---
title: "Horus: a new observability system for distributed microservices"
date: "2021-12-19"
description: "How can we make better open source observability systems?"
summary: "Horus plugs in a gap in the open source ecosystem with a full-stack observability system that's easy to integrate"
tags: ["observability", "microservices", "open source software"]
categories: ["software engineering"]
series: ["software engineering"]
ShowToc: false
TocOpen: false
---

 Open source observability software has a problem: gluing things together is time intensive and confusing.

Despite the wonderful advance of [OpenTelemetry](https://opentelemetry.io/), which is making telemetry data generation increasingly simple and lowering the switching costs between observability systems, we still need a plug-and-play solution for what to *do* with that data.

That's what I set out to do with some colleagues when we built [Horus](https://tryhorus.com/).

Horus is a full-stack observability system. It's made to make observability easy for distributed services by allowing you to:

1. Instrument your data with the `horus-agent` package and few lines of code ([link](https://www.npmjs.com/package/horus-agent))
2. Spin up `Horus` in Docker to process, store, and visualize your data
3. Help you investigate by connecting your **metrics to related traces**

If this sounds interesting, you might enjoy reasing our [case study](https://tryhorus.com/case-study) which explores the problems, existing solutions, and the trade-offs we made when building Horus.
