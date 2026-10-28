# How the Formatted Log Output Actually Works


The low-level function can keep using the old approach, writing strings directly. It gets called by the other log-output functions.  

Then write a proxy function that takes an XMLNODE parameter. This function accepts calls from all the other upper-level log-writing functions; the upper-level functions unify their log format, put the information into an XMLNODE, and then the proxy calls the low-level function to write the log.  

As for the upper-level functions, the most common kinds of log output are: plain text output (LOGERROR), assertion ASSERT, and exception CATCH. Make these three into three separate functions or macros. In practice, LOGERROR creates an XMLNODE, writes the log type, time, string and other information into the XMLNODE's attributes, then calls the proxy function.  
Then there's the logging for each feature block. Item-loss logging, for example, needs its own upper-level log function that formats the information into an XMLNODE and hands it to the proxy function to write.  

Then there's the server-side log output container, which can filter and display by log type and colour-code them — exception logs can be marked red as a warning, and so on.  

Finally, a word about the log viewer. The reason for making logs XML in the first place is that it makes them easy to use with a log viewer. Note that every time you add a new log type, besides writing the upper-level function that submits the log, you also have to write the parsing code for it in the log viewer.

