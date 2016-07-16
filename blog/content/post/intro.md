+++
date        = "2013-06-21T11:27:27-04:00"
title       = "Nitro: A quick and simple profiler for Go"
description = "Nitro is a simple profiler for your Golang applications"
tags        = [ "Development", "Go", "profiling" ]
topics      = [ "Development", "Go" ]
slug        = "nitro"
project_url = "https://github.com/spf13/nitro"
+++
# Nitro

Quick and easy performance analyzer library for [Go](http://golang.org/).

## Overview

Nitro is a quick and easy performance analyzer library for Go.
It is useful for comparing A/B against different drafts of functions
or different functions.

## Implementing Nitro

Using Nitro is simple. First, use `go get` to install the latest version
of the library.

    $ go get github.com/spf13/nitro

Next, include nitro in your application.
