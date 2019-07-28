---
title: "Dynamic SQL Using Python"
date: 2019-07-27
tags: [Python, MS SQL]
header:
excerpt: "Python, MS SQL"
---

Python version 3.6 introduced a formatted string literal or f-string which is a string literal that is prefixed with 'f' or 'F'. These strings may contain replacement fields, which are expressions delimited by curly braces {}. While other string literals always have a constant value, formatted strings are really expressions evaluated at run time.

That can be used to build dynamic SQL query:
{% gist 5a2d6ea582b2072b6926dc3271e2c189 %}
