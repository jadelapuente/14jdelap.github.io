---
title: "Why I work in tech."
date: "2024-12-07"
description: "Understand how Go embeds and calls C within a Go package by inspecting the symbol tables of the executable files with Cgo."
summary: "Understand how Go embeds and calls C within a Go package by inspecting the symbol tables of the executable files with Cgo."
tags: ["go", "c", "compilation symbols"]
categories: ["go", "c", "software engineering"]
# series: ["AI"]
ShowToc: true
TocOpen: false
draft: true
---

I regularly surprise people when they find out I don't have the typical background of a software engineer: I did a BSc in Government at the London School of Economics. Then there's always the question of why I work in tech — what motivated me to switch to something that is so different.

My one-liner is that tech is the oil industry of today: the industry that most pushes human progress. I want to explain what I mean with that and why I find it exciting and important.

- Life has historically been poor, difficult, and painful
- We've inherited a unique period of history where the world *grew* rather than stagnated
- The world isn't predetermined to always continue growing, it's something that depends on us individually and more broadly as a society
- Software has been one of the biggest engines of growth and development since the 70s, much more than other industries
- I believe that software is a critical *enabler* to not only its but also other industries' growth (e.g. mRNA vaccines)

## Nasty, brutish and short

```md
No arts; no letters; no society; and which is worst of all, continual fear and danger of violent death; and the life of man, solitary, poor, nasty, brutish, and short.
– Thomas Hobbes, *Leviathan*
```

Hobbes warned of life in the "state of nature" — without government and civil society — as "nasty, brutish, and short". The irony is that compared to today, virtually any other time in human history was precisely "nasty, brutish, and short" for everyone.

Just look at how much GDP growth concerns politicians:

- This week Xi Jingping [signalled in his New Years message](https://www.ft.com/content/f3f4327d-7733-4e84-b88d-9fa34ac8e163) that growth is China's "top priority" after it grew 4.8% rather than the projected 5.0%
- Europe is also concerned about its growth — just look at the [Draghi report](https://commission.europa.eu/topics/strengthening-european-competitiveness/eu-competitiveness-looking-ahead_en) — especially given the widening [GDP gap between the EU and the US' GDP](https://www.lemonde.fr/en/opinion/article/2023/09/04/the-gdp-gap-between-europe-and-the-united-states-is-now-80_6123491_23.html)
- Britain's Prime Minister [promising to "rebuild Britain"](https://www.bbc.com/news/articles/clll8d2vd8yo) by boosting economic growth

I get them, they're very, very important concerns! But they're also, relatively speaking, recent ones because the world didn't really grow until 200 years ago. Robert J. Gordon says in his seminal *Rise and Fall of American Growth*

> The most fundamental question of modern economic history is why after two millennia of no growth at all in per capita real output from Roman times to 1750, economic growth moved out of its hibernation and began to wake up.

This isn't to say that there wasn't wealth, but that *creating* wealth at a societal level was extremely difficult.

![Total output of the world economy from Our World in Data. These historical estimates of GDP are adjusted for inflation. Three sources were combined to create this time series: the Maddison Database before 1820, the Maddison Project Database between 1820–1989, and the World Bank from 1990 onward.](/images/global-gdp.png)

![Annual working hours per worker, which has halfed in 150 years in many countries.](/images/annual-working-hours.png)
