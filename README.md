## Basic AVR for .NET Windows template

This basic Windows template provides 

* Start-up program behavior
* A single database connection
* Global exception handling with error logging

### Start-up program behavior

A start-up program class that provides AVR Classic-like start-up program behavior. The `Main()` routine in the `Program` class is the program entry point. Command line arguments are available in `Main()` and can be used for things such as dynanimcally changing which of your program's forms display first.

You can add variables to this `Program` class to make them globally available to all forms and other classes in your project. For example, as provided, the `Program` class offers one global variable named `pgmDB`. The code below shows another global variable added named `CustomerNumber`. Be sure to mark any global variable as `Shared(*Yes)` and `Access(*Public)`.  

```
BegClass Program

    DclDB pgmDB DBName("*PUBLIC/Cypress") Shared(*Yes) Access(*Public)
    DclFld CustomerNumber Type(*Integer4) Shared(*Yes) Access(*Public)

    BegSr Main Shared(*Yes) Access(*Public) ... 
```

Later, in a form or another class, to use `CustomerNumber`, you only need to get or set it with its fully-qualified name like this: 

```
Chain Customer Key(Program.CustomerNumber)        
```

While this example shows `CustomerName` as a simple get/set field, you could also surface read-only properties or other types of data from the `Program` class--just be sure to mark the member `Shared(**Yes)` and `Access(**Public)`.   

> Beware! The road to programming hell is paved with global variables. Ideally, you would never add global variables as mentioned above. Global variables tightly couple routines together that should generally not know anything about each other. This makes your program much harder to maintain and debug. So, add a few global variables if your needs demand them, but don't get carried away. 

By the way, the "the road to programming hell..." quote from above is from Steve McConnell's [Code Complete, 2nd edition](https://www.amazon.com/Code-Complete-Practical-Handbook-Construction/dp/0735619670/ref=sr_1_1?ie=UTF8&qid=1473953357&sr=8-1&keywords=code+complete). This book is highly recommended and guaranteed to make you a better programmer.  

### A single database connection

The `Program` class provides a globally-available database connection. Using this DB connection ensures your Windows program will only start one database server job. This DB connection is available through a shared DclDB member in the `Program` class. 

*Why is this global database connection important?* Without it, each form that connects to a database server starts its own job on that server. With many forms, this means any one Windows program would cause *many* database server jobs. By routing all IO through a centrally declared database connection, you'll ensure that any one Windows program starts only one database server job. 

> If you look in the `Program` class, you'll notice its database connection is marked `Shared(\*Yes)`. Under no circumstances should you ever use `Shared(*Yes)` with either database 
connections or database files in an AVR for .NET Web app! The shared DB technique
shown here is for Windows applications only.

The code below is a minimal example showing how to use the global database connection in your forms.

~~~
Using System
Using System.Collections
Using System.ComponentModel
Using System.Data
Using System.Drawing
Using System.Text
Using System.Windows.Forms

BegClass Form1 Extends(System.Windows.Forms.Form) Access(*Public) Partial(*Yes)

    BegConstructor Access(*Public)
        InitializeComponent()
    EndConstructor

    DclDB pgmDB DBName( "*PUBLIC/Cypress" )

    DclDiskFile  Customer +
          Type( *Input  ) +
          Org( *Indexed ) +
          Prefix( Customer_ ) +
          File( "Examples/CMastNewL1" ) +
          DB( pgmDB ) +
          ImpOpen( *No )

        DclFld f2 Type(Form2) 

    BegSr Form1_Load Access(*Private) Event(*this.Load)
        DclSrParm sender *Object
        DclSrParm e System.EventArgs

        // Assign global DB object to local DB object.
        *This.pgmDB = Program.pgmDB
        // Open files as needed (after the DB assignment)
        Open Customer 
    EndSr

    BegSr Form1_FormClosing Access(*Private) Event(*this.FormClosing)
        DclSrParm sender Type(*Object)
        DclSrParm e Type(System.Windows.Forms.FormClosingEventArgs)
        
        // Make it easy on yourself here! *ALL is your friend. 
        Close *All 
    EndSr    
EndClass
~~~

Declare a local (to the form) DclDB connection and as many files as your form needs as you normally would. This local DB connection will be used for compile-time purposes. At runtime, in the Form's load event, use this line:

        *This.pgmDB = Program.DB 

to assign the local database connection to the `Program` class's global  database connection. You must take care to ensure *every* form assigns its local DB to the global one. Otherwise, you'll start more jobs than necessary on your database server.  

The database connection was opened in the `Program` class. Be sure to do this assignment before you open any files. After the assignment, open as many files as necessary in your form.

In the `FormClosing` event, use 

        Close *All 

to close all files when the form closes. The easiest way to ensure all files are closed is using `\*All` with `Close` instead of trying to remember what files got opened. \*All is safer and easier. Be sure to *not* close the DclDB connection in your forms. It is closed automatically when the program ends. 

> Do not close the global DB connection (`Program.pgmDB`) in any forms. The global connection is closed automatically when the program ends. 

### Global exception handling

Even the best program can incur an unexpected error. Without global exception handling, when an unexpected error occurs your users see a dialog similar to this: 

![](http://asna.com/media/images/exception_error-1.png)

Not only does this look unprofessional, but because your program isn't on the lookout for such unhandled exceptions, you don't have the opportunity to log the error. Without some rational error capturing, your best clues as to what happened as left to the user's memory. 

With global exception handling in place, when an unhandled exception occurs, a form similiar to this is displayed: 

![](http://asna.com/media/images/pretty_exception_error-2.png)

This form is included in the template so you can further customize it to your specifications.

A simple error logger is also included (which was adapted from [Microsoft ASP.NET error logging code](https://msdn.microsoft.com/en-us/library/bb397417.aspx)). With this error information recorded, you are no longer at the mercy of users' flawed memories! As coded, this logger writes error information to a file named `errorlog.txt` located in the same directory as the program's executable. When an unhandled exception occurs, the following text (which has been abbreviated here slightly for this article) is written to the log file: 

```
Info: ********** 9/15/2016 9:51:41 AM **********
Text: An unhandled UIThreadException occurred.
Exception: ********** 9/15/2016 9:51:41 AM **********
Exception Type: ASNA.VisualRPG.Runtime.ConnectionNotOpenException
Exception: File Customer cannot be opened because a connection to database *PUBLIC/Cypress has not been established
Stack Trace: 
   at ASNA.VisualRPG.Runtime.DBFile.open(Database database, AccessMode accessMode, Boolean isCacheWrite, Boolean isCommit, ServerCursors serverCursor)
   at StandardWindowsTemplate.Form2.Form2_Load(Object sender, EventArgs e) in C:\Users\roger\Documents\Programming\AVR\Other\StandardWindowsTemplate\StandardWindowsTemplate\Form2.vr:line 44
   at System.Windows.Forms.Form.OnLoad(EventArgs e)
```

The amount of detail recorded in the log file is dictated by the presence of [the program's PDB file.](http://devcenter.wintellect.com/jrobbins/pdb-files-what-every-developer-must-know) If the program's PDB file (Program Database File) is present alongside the EXE the logging info includes the class name and the line number of code in that class where the error occured. Without the PDB, you still get good error information, but you don't get the line number where the error occurred.  

The Logger class is easy to use and could also be used from other places in your program where you need to log something. It currently offers three shared members. The `Info()` method the one you'd most likely use--it lets you write an arbitrary string to the log file. The `Exception` is used when you want to log an exception to log file. Usually you'll leave that to the global error handling. These methods are shared so you don't need to instance the `Logger` class to use them.

```
Logger.Info('An unhandled domain exception occurred.') 
Logger.Exception(ex)
```        

The third `Logger` method is `NotifySystemOps`. That method doens't currently do anything but rather is provided as a jumping off place for you to add custom code to notify a system operator when an unhandled exception occurs. 

Logging errors during an unhandled exception is a bit of a gamble. For example, if the unhandled exception occurred because the user's disk is file, the code to log that error to a text file is gonna fail! That's not likely happen very often, an errant DB connection is far more likely to cause an unhandled exception. That's why errors are written to a text file instead to a database file. We'd like to avoid causing an unhandled exception logging an unhandled exception!     

> Beware that when deployed, you may encounter write errors trying to write to the folder where the EXE resides. If that's the case, consider logging the errors to users's `Documents` folder or some other location where you know write authorities exist.

Like the error form itself, the logging code is included in the template so that you can further customize it if necessary. 

To test unhandled exception handling in the template, click the `Throw unhandled exception` link label on `Form2.`  