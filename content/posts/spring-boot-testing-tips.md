+++
title = "Spring Boot Testing Tips"
date = "2021-09-15"
tags = [
    "spring-boot",
    "testing"
]
draft = true
+++

Some random dos and don'ts for writing tests with Spring Boot. The most recent stable Spring Boot version at the time of publishing was 2.5.4.

# Confine Test Configuration
The configuration and setup of the tests should be as close to them as possible. This reduces the possibility of one test interfering with another and makes it much easier for the reader of the test to get a sense of its purpose.
The following are a selection of means to achieve that goal:

## @TestConfiguration

## Properties