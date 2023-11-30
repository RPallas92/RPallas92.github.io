---
title: "Gas stations finder in Golang"
layout: post
date: 2023-11-30 08:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
- go
- algorithms
- data structures
- faang
- golang
- kd-tree
- gas stations
category: blog
author: Ricardo Pallas
projects: true
description: Gas Station Finder, crafted with Go, simplifies gas station searches with smart features and efficient architecture.
---

Hi all,

I wanted to share a project I created a couple of years ago, during winter holidays, with the purpose of learning Go.

In December 2022, gas prices were more expensive than ever in Spain, and I drove a lot. That's why I written the [Gas Station Finder](https://github.com/RPallas92/GasPrices) in Go; I wanted to find all gas stations between 2 cities and sort them by price. This way, I could fill the tank at a cheaper gas station without having to deviate from the route.

![Sample UI for Gas Stations service](https://raw.githubusercontent.com/RPallas92/GasPrices/main/gas_prices_ui.png)

Let me break down how it all works.

The project relies on the OpenRouteService API to find the route between two coordinates. It also uses its reverse geocoding service to convert city names into coordinates. This is done to expose an API that given 2 city names, it returns both the route and all near gas stations with its prices.

The REST API is built with the Gin web framework. I liked it as it was straightforward and simple to use.

The app is split into different components: one for figuring out where you are, one for planning routes, another for getting directions, one for getting gas station prices, and another to retrieve gas stations nearby.

In order to improve performance, I did to things:
- Use a KDTree to store Gas stations. For a given route, it called the KDTree to find nearby gas stations really fast (O(log n)).
- Keep all prices in memory and refresh them in the background every 4 hours. Since the prices don´t change that often, caching them was a good idea to improve performance, because retrieving the prices from the Spanish goverment website takes quite a bit.

As an improvement, instead of having all the gas stations prices in memory, we can use an embedded databse like [GrausDB](https://github.com/RPallas92/GrausDB). It can also be used to store routes between cities instead of calling OpenRouteService to calculate them each time. These routes are not going to change often, therefore we can keep refresh them every 2 weeks, for example.


I enjoyed coding it since it only took a few hours, and it proved to be a useful app for myself. Also, I learned a little bit of Go!

You can find the code [on GitHub: GasPrices](https://github.com/RPallas92/GasPrices).

Thanks for reading!
Ricardo.