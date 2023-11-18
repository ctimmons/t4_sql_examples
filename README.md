t4_sql_examples
===============

(NOTE: This respository is archived.  The latest versions of C# have robust string interpolation capabilities, which makes T4 templates mostly obsolete.  Looked at another way, it's now easier to write a C# app that generates code than it is to write the same functionality in a set of T4 templates.)

For a [variety of reasons](http://stackoverflow.com/questions/760834/how-can-i-design-a-java-web-application-without-an-orm-and-without-embedded-sql), I don't like [ORMs](http://en.wikipedia.org/wiki/Object-relational_mapping).  In the absence of an ORM, the usual alternative is manually writing hundreds - or even thousands - of lines of boilerplate code to pass data back and forth between the app(s) and the database.  Yuck.

Fortunately there's an easier way.  A code generator can quickly write all of that boilerplate at the touch of a button.  The best part is that you, the programmer, have full access to both the templates and the generated source code, so you can change what gets generated at will.  This is unlike an ORM, which is going to hide what it's doing under the covers, and make it hard - if not impossible - to do anything outside of that particular ORM's model (they're all a little different, and none has truly universal capabilities).

I'm not aware of any commercial or open source code generators that do this for .Net apps, so I wrote my own.  A .Net library that exposes the metadata for a given MS SQL server is located in the Utilities.Sql project in the [cs_Utilities](https://github.com/ctimmons/cs_utilities) repository.

This Visual Studio solution contains five projects demonstrating how use that library and T4 templates to generate various kinds of database access code - TSQL, C#, F# and Visual Basic.

Dependencies
------------

All of the projects depend on the Utilities.Sql project from the [cs_Utilities](https://github.com/ctimmons/cs_utilities) solution.

The generated code may have dependencies on Utilities.Core, Utilities.Sql (both are in cs_Utilities), and/or Microsoft.SqlServer.Types, in addition to any assemblies you choose to generate code references to.

(To get the Microsoft.SqlServer.Types assembly, go to [SQL Server 2012 SP1 Feature Pack](http://www.microsoft.com/en-us/download/details.aspx?id=35580) and expand the "Install Instructions" section.  Scroll down to the section labeled "Microsoft® System CLR Types for Microsoft® SQL Server® 2012" and download and run the appropriate 32 or 64 bit installer.

There are feature packs available for other versions of SQL Server.  Go to [Microsoft Search](http://search.microsoft.com/) and search for "sql server feature pack".)

Generated TSQL code may also reference a TSQL function called "dbo.util_Get_SqlVariant_As_NVarCharMax".  This is a small function I wrote that returns the string representation of any SQL_VARIANT value.  It can be found in the [tsql_utilities](https://github.com/ctimmons/tsql_utilities) repository.  This is currently hardcoded in the Utilities.Sql code, but in the near future I might change it to be a configuration option.


The Projects
------------

### T4_Utilities

An empty C# project whose only purpose is to serve as a home for T4 utility templates.

These templates are referenced in the other projects' .tt files via the T4 include directive, e.g. <#@ include file="..\T4_Utilities\SQL Generator Utilities.ttinclude" #>.

### SQL

Another empty C# project whose only purpose is to serve as a home for T4 templates that generate T-SQL code.

### CSharp and Visual_Basic

Basically identical to each other.  The project's Main.tt template generates code for the data entity classes and data access methods, producing either C# or VB code.

### FSharp

The Main.tt template and included templates are the same structure as the CSharp and Visual_Basic projects.

However, F# projects in Visual Studio do *not* support T4 templates.  To get around this, the standalone TextTransform.exe template generator needs to be run separately on the Main.tt template.  For .Net 4.5, this utility is found in "C:\Program Files (x86)\Common Files\microsoft shared\TextTemplating\12.0\".  For previous versions of .Net, TextTransform.exe may be under the \10.0\ or \11.0\ subfolders.

In the project's properties page, I've put a call to TextTransform in the project's pre-build step (open the FSharp project's properties and go to the "Build Events" tab).  Just rebuild the project to run the T4 templates.

### Processing Logic

Except for the T4_Utilities project, the processing logic is the same:

- The Main.tt file sets up the code generation environment by loading the necessary assemblies, importing namespaces, and including any other T4 templates (*.ttinclude files) it needs.
- An SqlConnection is created and opened.
- A Configuration instance is created and its properties set to the desired values.
- An output directory is set.
- A Server instance is created.
- The template functions that actually generate the code are called.


Generating the Code
-------------------

First, download and build the [cs_Utilities](https://github.com/ctimmons/cs_utilities) Visual Studio solution from GitHub.

Then make these changes to the projects in this solution to generate the code on your system:

In the Main.tt file in each project (except for the T4_Utilities project - it doesn't have a Main.tt file):

1. Alter the <#@ assembly name=...#> directive that loads Utilities.Sql.dll to point to the correct location on your system.

2. Alter the connection string to match your MS SQL server name and security credentials.

3. Optionally, change the outputDirectory variable to point to where you want the templates to put the generated files.  Right now all of the projects Main.tt templates write their output files to a folder under the project's root folder.

In the T4_Utilities project, there's a file named SQL Generator Utilities.ttinclude that contains a _prefixes dictionary.  This dictionary is used by the GetStoredProcedureName method (right after the dictionary) to generate a uniform name for a stored procedure.  Change the keys and values in the dictionary to reflect the database names on your server (the keys) and the abbreviation for each (the values).

In the FSharp project, open the project's properties and go to the "Build Events" tab.  As noted above, the TextTransform.exe that comes with .Net should be in a subfolder of "C:\Program Files (x86)\Common Files\microsoft shared\TextTemplating\", either the 10.0\ or 11.0\ subfolder.  Make sure the call in the Pre-Build step is pointing to the correct folder.

Except for the FSharp project, altering and then saving Main.tt should make Visual Studio run the template and generate the code.  The template can also be run by right clicking on the Main.tt file in Visual Studio's Project Explorer and choosing "Run Custom Tool".

For the FSharp project, building the project will cause the call to TextTransform.exe in the Pre-Build step to be executed.


The Generated Code
------------------

The code generated by the CSharp, FSharp and Visual_Basic projects all generate two source files: one containing classes that serve as simple database entities (one per table), and a file containing access methods that perform the select, insert, update, and delete operations using those entities.

The SQL project's templates iterate over all of the tables in the selected database(s) on the specified server.  For each table, the templates generate a separate stored procedures for each of the SELECT, INSERT, UPDATE, MERGE, and DELETE operations on a table.  This means, for example, if you've selected 4 databases, and each database has 10 tables, then the SQL project will generate 200 files (40 tables times 5 operations per table).


Adding the Generated Code to the Project
----------------------------------------

For the CSharp, FSharp and Visual_Basic projects, the generated code can be added to its respective project (or any other C#, F#, or VB project) and compiled.

The stored procedures produced by the SQL project can be loaded into SQL Server Management Studio and executed.


What's Next
-----------

The T4 templates included in this repository only scratch the surface of what's possible.  Unlike ORMs, you're not limited to what the ORM author thought was the "right" way to do things.  The sky really is the limit when it comes to code generation.

If you have any ideas for template examples, or new functionality in the [Utilities.Sql](https://github.com/ctimmons/cs_utilities) library, please create an enhancement issue in the relevant GitHub repository.

