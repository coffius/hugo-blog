+++
date = "2017-09-02T18:00:00+03:00"
draft = true
title = "Testing Cassandra"
slug = "testing-cassandra"
+++

Ways of testing Cassandra:

* Mocking session with [scalamock](http://scalamock.org/)
* Using an embedded server with [cassandra-unit](https://github.com/jsevellec/cassandra-unit)
* Using a docker container with [docker-it-scala](https://github.com/whisklabs/docker-it-scala)

<!--more-->

## Mocking session
It is possible to mock a session if you use [Datastax Driver](https://github.com/datastax/java-driver)

_TODO:_ Code example
_TODO:_ Check if it is possible to do the same for phantom
_TODO_

## Embedded Cassandra server

Static daemon - one instance for all tests
Problems with parallel tests - timeouts
Problems with

_TODO_

## Integration with Docker

Code example
Require docker support at CI server

_TODO_

## Links

* Sources - _TODO_
* [`scalamock`](http://scalamock.org/)
* [`cassandra-unit`](https://github.com/jsevellec/cassandra-unit)
* [`docker-it-scala`](https://github.com/whisklabs/docker-it-scala)

_TODO_
