---
title: "Create Static Table Source Code"
date: 2019-12-30
tags: [SAS, Linux shell, PROC SQL]
header:
excerpt: "An example how to create a source code for a SAS static metadata table."
---
Do you have a static metadata table (sas7bdat file) and you need to generate a source code for this table including all values? You may ask why this would be needed? Many reasons, for example you would like to track the metadata table and its changes in Git. If you check in the whole sas7bdat file, you will not be able to fully utilize Git. To see and track metadata changes, you need both DDL (creating table) and DML (inserting data into table) of the metadata table. The below SAS code generates PROC SQL code which can replicate an existing metadata table and insert all existing metadata. Such a code can be easily tracked in Git. If you need to change your metadata, you can just adjust the insert statements and then execute your code in you PROD environment. It is assumed SAS on Unix/Linux environment.

The code is not complicated. First you need to prepare a macro **%get_ddl** which can generate the PROC SQL create statement. As I’m lazy, I borrowed this simple macro from [Allan Bowe](https://stackoverflow.com/users/66696/allan-bowe).

Then you prepare **%DS2SourceCode** macro which generates the whole source code. It creates 2 temporary files metatable_create.sas and metatable_data.sas. A few Linux shell commands then create the whole SAS code.

Before the final version of the SAS code is produced, it is tested if the code can exactly replicate the existing metadata table. This is done by the last macro **%meta**.

```sas
/*---------------------------------------------------------------------------*/
/*--Prepare macro for generating proc sql create statement                 --*/
/*---------------------------------------------------------------------------*/

%macro get_ddl(ds=,outfile=);
   filename tmp temp;
   proc printto log=tmp;quit;
   proc sql; describe table &ds;
   proc printto log=log;quit;
   data _null_;
      infile tmp;
      file &outfile;
      input;
      if _infile_=:'NOTE: SQL table ' then start+1;
      else if _infile_=:'NOTE: PROCEDURE SQL used' then stop;
      else if index(_infile_,'            The SAS System       ') then delete;
      else if start=1 then put _infile_;
      putlog _infile_;
   run;
   filename tmp;
%mend;

/*---------------------------------------------------------------------------*/
/*--Prepare macro for generating metadata source code                      --*/
/*---------------------------------------------------------------------------*/
%macro DS2SourceCode(metapath, metatable, TestScript=Yes);
	/*Define output library - the src code will recreate the meata table in
	this library*/
	%if &TestScript=YES %then %do;
		%let workpath=%sysfunc(pathname(work));
		libname lib "&workpath.";
		%end;
	%else %do;
		libname lib "&metapath.";
		%end;

	/*Define metadata library*/
	libname metalib "&metapath";

	/*Copy the meta table int the work library*/
	proc copy in=metalib out=work;
		select &metatable;
	run;

	/*Create proc sql create table statement*/
	%get_ddl(ds=lib.&metatable., outfile="&metapath./&metatable._create.sas");

	/*Create proc sql insert statement*/
	data _null_;
		file "&metapath./&metatable._data.sas" dlm=',';
		set metalib.&metatable.;
		format _character_ $quote200.;
		put "Insert into lib.&metatable. values(" (_all_) (~) ');';
	run;

	/*Create source code which can replicate the meta table in the work library*/
	x muser=$(stat -c '%U' &metapath./&metatable..sas7bdat);
	x mgroup=$(stat -c '%G' &metapath./&metatable..sas7bdat);
	x echo "libname lib '%sysfunc(pathname(lib))';" > &metapath./&metatable..sas;
	x echo "proc sql noprint;" >> &metapath./&metatable..sas;
	x cat &metapath./&metatable._create.sas >> &metapath./&metatable..sas;
	x cat &metapath./&metatable._data.sas >> &metapath./&metatable..sas;
	x echo "quit;" >> &metapath./&metatable..sas;
	x chown ${muser}:${mgroup} &metapath./&metatable..sas;

	/*Remove unnecesary files*/
	x rm &metapath./&metatable._create.sas;
	x rm &metapath./&metatable._data.sas;

	/*Remove temp tables*/
	proc delete data = work.&metatable.;
	run;
%mend DS2SourceCode;

/*---------------------------------------------------------------------------*/
/*--Test the metadata source code and output final version                 --*/
/*---------------------------------------------------------------------------*/
%macro meta(metapath, metatable);
	/*Create src script using the work library as output destination for the
  meta table*/
	%DS2SourceCode(&metapath, &metatable, TestScript=YES);

	/*Replicate the meta table in the work library using the newly created src
  script*/
	options nosource nonotes;
	%include "&metapath./&metatable..sas";
	options source notes;

	/*Compare the meta table from the metadata library and the newly created
  meta table from the work library*/
	proc compare base=metalib.&metatable.
				 compare=work.&metatable.;
	run;

	/*Generate final source code*/
	%let metarc=&sysinfo;
	%put RC: &metarc;
	%if &metarc eq 0 %then %do;
		%DS2SourceCode(&metapath, &metatable); /*i.e. TestScript=NO*/
		%end;
	%else %do;
		%put Source code does not generate the same sas dataset;
		x rm &metapath./&metatable..sas;
		%end;

%mend meta;

/*run macro example*/
*%meta(Home/usr/metadata, holiday);
```
SAS code [link](https://github.com/VankatPetr/SAS/blob/master/StaticTableSourceCode/StaticTableSourceCode.sas)

Example of a generated source code:
```sas
libname lib 'Home/usr/metadata';
create table LIB.HOLIDAY( label='Holiday data for en_US and en_CA locale' bufsize=65536 )
  (
   name char(32),
   desc char(64),
   category char(32),
   begin num,
   end num,
   month num,
   day num,
   rule num
  );

Insert into lib.holiday values("BOXING","Boxing Day","en_CA",.,.,12,26,0,);
Insert into lib.holiday values("CANADA","Canadian Independence Day","en_CA",.,.,7,1,0,);
Insert into lib.holiday values("CANADAOBSERVED","Canadian Independence Day observed","en_CA",.,.,7,1,8,);
Insert into lib.holiday values("CHRISTMAS","Christmas","en_US",.,.,12,25,0,);
Insert into lib.holiday values("CHRISTMAS","Christmas","en_CA",.,.,12,25,0,);
Insert into lib.holiday values("COLUMBUS","Columbus Day","en_US",.,.,10,2,2,);
Insert into lib.holiday values("EASTER","Easter Sunday","en_US",.,.,0,0,11,);
Insert into lib.holiday values("FATHERS","Father's Day","en_US",.,.,6,3,1,);
Insert into lib.holiday values("HALLOWEEN","Halloween","en_US",.,.,10,31,0,);
Insert into lib.holiday values("LABOR","Labor Day","en_US",.,.,9,1,2,);
Insert into lib.holiday values("LABOR","Labor Day","en_CA",.,.,9,1,2,);
Insert into lib.holiday values("MLK","Martin Luther King, Jr.'s birthday","en_US",1986,.,1,3,2,);
Insert into lib.holiday values("MEMORIAL","Memorial Day","en_US",1971,.,5,-1,2,);
Insert into lib.holiday values("MOTHERS","Mother's Day","en_US",.,.,5,2,1,);
Insert into lib.holiday values("NEWYEAR","New Year's Day","en_US",.,.,1,1,0,);
Insert into lib.holiday values("NEWYEAR","New Year's Day","en_CA",.,.,1,1,0,);
Insert into lib.holiday values("THANKSGIVING","U.S. Thanksgiving Day","en_US",.,.,11,4,5,);
Insert into lib.holiday values("THANKSGIVINGCANADA","Canadian Thanksgiving Day","en_CA",.,.,10,2,2,);
Insert into lib.holiday values("USINDEPENDENCE","U.S. Independence Day","en_US",.,.,7,4,0,);
Insert into lib.holiday values("USPRESIDENTS","Abraham Lincoln's and George Washington's birthdays","en_US",1971,.,2,3,2,);
Insert into lib.holiday values("VALENTINES","Valentine's Day","en_US",.,.,2,14,0,);
Insert into lib.holiday values("VETERANS","Veterans Day","en_US",1938,1971,11,11,0,);
Insert into lib.holiday values("VETERANS","Veterans Day","en_US",1971,1978,10,4,1,);
Insert into lib.holiday values("VETERANS","Veterans Day","en_US",1978,.,11,11,0,);
Insert into lib.holiday values("VETERANSUSG","Veterans Day - U.S. government-observed","en_US",.,.,11,11,9,);
Insert into lib.holiday values("VETERANSUSPS","Veterans Day - U.S. post office-observed","en_US",.,.,11,11,8,);
Insert into lib.holiday values("VICTORIA","Victoria Day","en_CA",.,.,5,24,10,);
quit;
```
