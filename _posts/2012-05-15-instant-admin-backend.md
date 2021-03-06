---
layout: post
title: Bottle, Elixir, Bootstrap and Datatables - Instant Admin Backend
categories:
---

# {{ page.title }}

_UPDATE: I've recently stopped using Elixir and have moved to using straight SQLAlchemy._

Recently I have been working on a web-based administrative backend, and have found doing so unusually easy, mostly because of the combination of [Bottle](http://bottlepy.org/), a python micro-framework, [Elixir](http://elixir.ematia.de/), a wrapper over top of [SQLAlchemy](http://www.sqlalchemy.org/) (which is itself an SQL toolkit and [ORM](http://en.wikipedia.org/wiki/Object-relational_mapping)), Twitter's [bootstrap](http://twitter.github.com/bootstrap/), scaffolding for websites, and finally [Datatables](http://datatables.net/), which enables advanced interactions with HTML tables.

The only difficulty I had, and it was slight, was integrating bootstrap and datatables together, but this [post](http://datatables.net/blog/Twitter_Bootstrap_2) helped out quite a bit.

I would certainly suggest to anyone looking to create an administrative backend, or any web application really, to look into these four technologies, as it would take very little time to create a [minimum viable product](http://en.wikipedia.org/wiki/Minimum_viable_product), or a demo, using a combination of Bottle, Elixir, Bootstrap, and Datatables.

In future posts I will present an example application.
