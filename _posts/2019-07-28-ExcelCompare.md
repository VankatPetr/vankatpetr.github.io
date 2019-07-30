---
title: "Compare 2 Excel Files and Create an Excel Diff using Python"
date: 2019-07-28
tags: [Python, Pandas, Excel]
header:
excerpt: "A simple code for creating a diff of 2 excel files."
---

I recently needed to create  a diff of 2 Excel files. Because I was not aware of any tool which could do that, I scripted the following Python code which does the job. It outputs everything to an Excel file with a separate tab for differences, new rows and dropped rows.

Notes:
* Excel files need to have the same columns (order, names)
* The following parameters need to be set up (at the beginning of the code):
  - Path to the 'old' Excel file (containing the original rows)
  - Path to the 'new' Excel file (containing the changed rows)
  - List of key columns - this can be one or more columns (list), the column(s) has to be a primary key
  - Sheet name (the sheet containing the data) - the code assumes the same sheet name in both files (but it can be easily changed)

Here is the full code (example):
{% gist daeccc1d238f342b6029c3b92f20862a %}
