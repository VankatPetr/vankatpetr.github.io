---
title: "Create Static Table Source Code II"
date: 2020-01-01
tags: [SAS, JSON engine]
header:
excerpt: "An example how to create a source code for a SAS static metadata table using SAS JSON engine."
---

In my previous blog I demonstrated how you can create a source code for a SAS static metadata table (sas7bdat file). In this blog I would like to demonstrate another way how you can achieve this using SAS JSON engine.
First, you convert your metadata table to a json file. The following macro **%meta_json** does that.
```sas
%macro meta_json(metapath, metatable);
	%let libpath=&metapath.;
	%let metatable=&metatable.;
	libname lib "&libpath.";

	proc json out="&libpath./&metatable..json" pretty;
		export lib.&metatable.;
	run;
%mend meta_json;

/*example of the meta_json macro */
*%meta_json(/home/petrvankat0/Metadata,class);
```
It will create a json file like this:

```json
{
  "SASJSONExport": "1.0 PRETTY",
  "SASTableData+CLASS": [
    {
      "Name": "Alfred",
      "Sex": "M",
      "Age": 14,
      "Height": 69,
      "Weight": 112.5
    },
    {
      "Name": "Alice",
      "Sex": "F",
      "Age": 13,
      "Height": 56.5,
      "Weight": 84
    },
    {
      "Name": "Barbara",
      "Sex": "F",
      "Age": 13,
      "Height": 65.3,
      "Weight": 98
    },
    {
      "Name": "Carol",
      "Sex": "F",
      "Age": 14,
      "Height": 62.8,
      "Weight": 102.5
    },
    {
      "Name": "Henry",
      "Sex": "M",
      "Age": 14,
      "Height": 63.5,
      "Weight": 102.5
    },
    {
      "Name": "James",
      "Sex": "M",
      "Age": 12,
      "Height": 57.3,
      "Weight": 83
    },
    {
      "Name": "Jane",
      "Sex": "F",
      "Age": 12,
      "Height": 59.8,
      "Weight": 84.5
    },
    {
      "Name": "Janet",
      "Sex": "F",
      "Age": 15,
      "Height": 62.5,
      "Weight": 112.5
    },
    {
      "Name": "Jeffrey",
      "Sex": "M",
      "Age": 13,
      "Height": 62.5,
      "Weight": 84
    },
    {
      "Name": "John",
      "Sex": "M",
      "Age": 12,
      "Height": 59,
      "Weight": 99.5
    },
    {
      "Name": "Joyce",
      "Sex": "F",
      "Age": 11,
      "Height": 51.3,
      "Weight": 50.5
    },
    {
      "Name": "Judy",
      "Sex": "F",
      "Age": 14,
      "Height": 64.3,
      "Weight": 90
    },
    {
      "Name": "Louise",
      "Sex": "F",
      "Age": 12,
      "Height": 56.3,
      "Weight": 77
    },
    {
      "Name": "Mary",
      "Sex": "F",
      "Age": 15,
      "Height": 66.5,
      "Weight": 112
    },
    {
      "Name": "Philip",
      "Sex": "M",
      "Age": 16,
      "Height": 72,
      "Weight": 150
    },
    {
      "Name": "Robert",
      "Sex": "M",
      "Age": 12,
      "Height": 64.8,
      "Weight": 128
    },
    {
      "Name": "Ronald",
      "Sex": "M",
      "Age": 15,
      "Height": 67,
      "Weight": 133
    },
    {
      "Name": "Thomas",
      "Sex": "M",
      "Age": 11,
      "Height": 57.5,
      "Weight": 85
    },
    {
      "Name": "William",
      "Sex": "M",
      "Age": 15,
      "Height": 66.5,
      "Weight": 112
    }
  ]
}
```

Then, you can utilize SAS JSON engine to read the json file and work with the data in the same way as you normally do with a dataset. For example, the following macro **%meta_json_read** enables you to read/print the json file using the proc print. In the same way you can use the json data in a data step etc.

```sas
%macro meta_json_read(metapath, metatable);
	%let libpath=&metapath.;
	%let metatable=&metatable.;

	filename in "&libpath./&metatable..json";
	libname in json;

	proc sql noprint;
		select memname
		into :metajson
		from
			dictionary.tables
		where
			libname = 'IN' and memname like 'SASTABLE%'
		;
	quit;

	proc print data=in.&metajson(drop=ordinal_root
	                                  ordinal_&metajson )
				     noobs;
	run;
%mend meta_json_read;


/*example of the meta_json_read macro */
*%meta_json_read(/home/petrvankat0/Metadata,class);
```
Output of the **%meta_json_read** macro:
<img src="{{ site.url }}{{ site.baseurl }}/images/StaticTableSourceCodeJsonMeta_Print.PNG" alt="Data Visualisation">


SAS code [link](https://github.com/VankatPetr/SAS/blob/master/StaticTableSourceCodeII/StaticTableSourceCodeJsonMeta.sas)
