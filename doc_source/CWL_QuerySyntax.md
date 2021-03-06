# CloudWatch Logs Insights Query Syntax<a name="CWL_QuerySyntax"></a>

CloudWatch Logs Insights supports a query language you can use to perform queries on your log groups\. Each query can include one or more query commands separated by Unix\-style pipe characters \(`|`\)\.

Six query commands are supported, along with many supporting functions and operations, including regular expressions, arithmetic operations, comparison operations, numeric functions, datetime functions, string functions, and generic functions\.

Comments are also supported\. Lines in a query that start with the `#` character are ignored\.

For information about the fields that CloudWatch Logs discovers automatically, see [Supported Logs and Discovered Fields](CWL_AnalyzeLogData-discoverable-fields.md)\.

## CloudWatch Logs Insights Query Commands<a name="CWL_QuerySyntax-commnds"></a>

The following table lists the six supported query commands along with basic examples\. For more powerful sample queries, see [Sample Queries](CWL_QuerySyntax-examples.md)\.


| Command | Description | Examples | 
| --- | --- | --- | 
|  `display` |  Specifies which fields to display in the query results\. If you specify this command more than once in your query, only the fields you specify in the last occurrence are used\.  |  <pre>FIELDS @message<br />| PARSE @message "[*] *" as loggingType, loggingMessage<br />| FILTER loggingType = "ERROR"<br />| DISPLAY loggingMessage<br /></pre> This example uses the field `@message` and creates the ephemeral fields `loggingType` and `loggingMessage` for use in the query\. It filters the events to only those with `ERROR` as the value of `loggingType`, but then displays only the `loggingMessage` field in the results\.  | 
|  `fields` |  Retrieves the specified fields from log events\. You can use functions and operations within a fields command\.  |  <pre>fields `foo-bar`, action, abs(f3-f4)</pre> This example retrieves the fields `foo-bar`, `action`, and the absolute value of the difference between `f3` and `f4` for all log events in the log group\. Any log field named in a query that has characters other than the @ sign, the period \(\.\), and alphanumeric characters must be surrounded by backtick \(```\) characters\. For example, the `foo-bar` field name must be enclosed in backtick characters because it includes a non\-alphanumeric character\.   | 
|  `filter` |  Filters the results of a query based on one or more conditions\. You can use comparison operators \(=, \!=, <, <=, >, >=\), Boolean operators \(`and`, `or`, and `not`\) and regular expressions\. You can use `in` to test for set membership\. Put an array with the elements to check for immediately after `in`\. You can use `not` with `in`\.  | <pre>fields f1, f2, f3 | filter (duration>2000)</pre> retrieves the fields `f1`, `f2`, and `f3` for all log events with a value over 2000 in the `duration` field\. <pre>filter<br />                                        (duration>2000)</pre> is also a valid query, but the results don't show separate fields\. Instead, the results show the `@timestamp` and all log data in the `@message` field for all log events where duration is more than 2000\. <pre>fields f1, f2 | filter (f1=10 or f3>25)</pre> retrieves the fields `f1` and `f2` for all log events where `f1` is 10 or `f3` is more than 25\. <pre>fields f1 | filter statusCode like /2\d\d/</pre> returns log events where the field `statusCode` has a value between 200 and 299\. <pre>fields @timestamp, @message <br />| filter @message in [300,400,500]</pre> ` ` returns log events that include "300", "400", or "500" in the message\. <pre>fields @timestamp, @message<br />| filter @message not in ["foo","bar",1]</pre> returns log events that do not include "foo", "bar", or "1" in the message\. | 
|  `stats` |  Calculates aggregate statistics based on the values of log fields\. Several statistical operators are supported, including `sum()`, `avg()`, `count()`, `,min()`, and `max()`\. When you use `stats`, you can also use `by` to specify one or more criteria to use to group data when calculating the statistics\.  |  <pre>stats avg (f1) by f2</pre> calculates the average value of `f1` for each unique value of `f2`\.  | 
|  `sort` |  Sorts the retrieved log events\. Both ascending \(`asc`\) and descending \(`desc`\) order are supported\.  |  <pre>fields f1, f2, f3 | sort f1 desc</pre>  retrieves the fields `f1`, `f2`, and `f3` and sorts the returned events in descending order based on the value of `f1`\.  | 
|  `limit` |  Specifies the number of log events returned by the query\. You can use this to limit the results to a small number to see a small set of relevant results\. You can also use `limit` with a number between 1000 and 10,000 to increase the number of query result rows displayed in the console to an amount greater than the default of 1000 rows\. If you don't specify a limit, the query defaults to displaying a maximum of 1000 rows\. |  <pre>fields f1, f2 | sort @timestamp desc | limit 25</pre> retrieves the fields `f1` and `f2`, sorts the events in descending order based on the value of `@timestamp`, and returns the first 25 events by sort order\. In this case, the sort order is by timestamp starting with the most recent, so the most recent 25 events are returned\. Some sample queries provided with CloudWatch Logs Insights use `head` or `tail` commands instead of `limit`\. These commands are being deprecated and have been replaced with `limit`\. Use `limit` instead of `head` or `tail` in all queries that you write\.  | 
|  `parse` |  Extracts data from a log field, creating one or more ephemeral fields that you can process further in the query\. `parse` accepts both glob expressions and regular expressions\. For glob expressions, provide the `parse` command with a constant string \(characters enclosed in either single or double quotation marks\) where each variable piece of text is replaced with an asterisk \(\*\)\. These are extracted into ephemeral fields and given an alias after the `as` keyword, in positional order\. Enclose regular expressions in forward slashes \(/\)\. Within the expression, each part of the matched string that is to be extracted is enclosed in a named capturing group\. An example of a named capturing group is `(?<name>.*)`, where `name` is the name and `.*` is the pattern\.   |  Using this single log line as an example: <pre>25 May 2019 10:24:39,474 [ERROR] {foo=2, bar=data} The error was: DataIntegrityException</pre> The following two `parse` expressions each do the following: the ephemeral fields `level`, `config`, and `exception` are extracted\. `level` has a value of `ERROR`, `config` has a value of `{foo=2, bar=data}`, and `exception` has a value of `DataIntegrityException`\. The first expression uses a glob expression, and the second uses a regular expression\. <pre>parse @message "[*] * The error was: *" as level, config, exception</pre> <pre>parse @message /\[(?<level>\S+)\]\s+(?<config>\{.*\})\s+The error was: (?<exception>\S+)/</pre> The following example uses a regular expression to extract the ephemeral fields `@user2`, `@method2`, and `@latency2` from the log field `@message` and returns the average latency for each unique combination of `@method2` and `@user2`\. <pre>parse @message /user=(?<user2>.*?), method:(?<method2>.*?), latency := (?<latency2>.*?)/ <br />| stats avg(@latency2) by @method2, @user2</pre>  | 

## Regular Expressions in the Filter Command<a name="CWL_QuerySyntax-regex"></a>

You can use `like` or `=~` \(equal sign followed by a tilde\) in the `filter` query command to filter by substrings or regular expressions\. Enclose your match string with double or single quotation marks to perform substring matching\. To perform regular expression matching, enclose it with forward slashes\. The query returns only log events that match the criteria that you set\.

**Examples**

The following three examples return all events in which `f1` contains the word `Exception`\. The first two examples use regular expressions\. The third example uses a substring match\. All three examples are case sensitive\.

```
fields f1, f2, f3 | filter f1 like /Exception/
```

```
fields f1, f2, f3 | filter f1 =~ /Exception/
```

```
fields f1, f2, f3 | filter f1 like "Exception"
```

The following example uses a regular expression\. It returns all events in which `f1` contains the word `Exception`\. The query isn't case sensitive\.

```
fields f1, f2, f3 | filter f1 like /(?i)Exception/ 
```

The following example uses a regular expression\. It returns all events in which `f1` is exactly the word `Exception`\. The query isn't case sensitive\.

```
fields f1, f2, f3 | filter f1 =~ /^(?i)Exception$/
```

## Using Aliases in Queries<a name="CWL_QuerySyntax-alias"></a>

You can use `as` to create one or more aliases in a query\. Aliases are supported in the `fields`, `stats`, and `sort` commands\. 

You can create aliases for log fields and for the results of operations and functions\. 

**Examples**

The following examples show the use of aliases in query commands\.

```
fields abs(myField) as AbsoluteValuemyField, myField2
```

Returns the absolute value of `myField` as `AbsoluteValuemyField` and also returns the field `myField2`\.

```
stats avg(f1) as myAvgF1 | sort myAvgF1 desc
```

Calculates the average of the values of the `f1` as `myAvgF1` and returns them in descending order by that value\.

## Using Comments in Queries<a name="CWL_QuerySyntax-comments"></a>

You can comment out lines in a query by using the `#` character\. Lines that start with the `#` character are ignored\. This can be useful to document your query or to temporarily ignore part of a complex query for one call, without deleting that line\.

In the following example, the second line of the query is ignored\.

```
fields @timestamp, @message
# | filter @message like /delay/
| limit 20
```

## Supported Operations and Functions<a name="CWL_QuerySyntax-operations-functions"></a>

The query language supports many types of operations and functions, as shown in the following tables\.

**Comparison Operations**

You can use comparison operations in the `filter` command and as arguments for other functions\. Comparison operations accept all data types as arguments and return a Boolean result\.

```
= != < <= > >=
```

**Boolean Operators**

You can use the Boolean operators `and`, `or`, and `not`\. You can use these Boolean operators only in functions that return a Boolean value\.

**Arithmetic Operations**

You can use arithmetic operations in the `filter` and `fields` commands and as arguments for other functions\. Arithmetic operations accept numeric data types as arguments and return numeric results\.


| Operation | Description | 
| --- | --- | 
|  `a + b` |  Addition  | 
|  `a - b` |  Subtraction  | 
|  `a * b` |  Multiplication  | 
|  `a / b` |  Division  | 
|  `a ^ b` |  Exponentiation\. `2 ^ 3` returns `8`  | 
|  `a % b` |  Remainder or modulus\. `10 % 3` returns `1`  | 

**Numeric Operations**

You can use numeric operations in the `filter` and `fields` commands and as arguments for other functions\. Numeric operations accept numeric data types as arguments and return numeric results\.


| Operation | Description | 
| --- | --- | 
|  `abs(a)` |  Absolute value\.  | 
|  `ceil(a)` |  Round to ceiling \(the smallest integer that is greater than the value of `a`\)\.  | 
|  `floor(a)` |  Round to floor \(the largest integer that is smaller than the value of `a`\)\.  | 
|  `greatest(a,b,... z)` |  Returns the largest value\.  | 
|  `least(a, b, ... z)` |  Returns the smallest value\.  | 
|  `log(a)` |  Natural log\.  | 
|  `sqrt(a)` |  Square root\.  | 

**General Functions**

You can use general functions in the `filter` and `fields` commands and as arguments for other functions\. 


| Function | Arguments | Result Type | Description | 
| --- | --- | --- | --- | 
|  `ispresent(fieldname)` |  Log field |  Boolean |  Returns `true` if the field exists\.  | 
|  `coalesce(fieldname1, fieldname2, ... fieldnamex)` |  Log fields |  Log field |  Returns the first non\-null value from the list\.  | 

**String Functions**

You can use string functions in the `filter` and `fields` commands and as arguments for other functions\. 


| Function | Arguments | Result Type | Description | 
| --- | --- | --- | --- | 
|  `isempty(fieldname)` |  String |  Boolean |  Returns `true` if the field is missing or is an empty string\.  | 
|  `isblank(fieldname)` |  String |  Boolean |  Returns `true` if the field is missing, an empty string, or contains only white space\.  | 
|  `concat(string1, string2, ... stringz)` |  Strings |  String |  Concatenates the strings\.  | 
|  `ltrim(string) or ltrim(string1, string2)` |  String |  String |  Removes white space from the left of the string\. If the function has a second string argument, it removes the characters of `string2` from the left of `string1`\. For example, `ltrim("xyZfooxyZ","xyZ")` returns `"fooxyZ"`\.  | 
|  `rtrim(string) or rtrim(string1, string2)` |  String |  String |  Removes white space from the right of the string\. If the function has a second string argument, it removes the characters of `string2` from the right of `string1`\. For example, `rtrim("xyZfooxyZ","xyZ")` returns `"xyZfoo"`\.  | 
|  `trim(string) or trim(string1, string2)` |  String |  String |  Removes white space from both ends of the string\. If the function has a second string argument, it removes the characters of `string2` from both sides of `string1`\. For example, `trim("xyZfooxyZ","xyZ")` returns `"foo"`\.  | 
|  `strlen(string)` |  String |  Number |  Returns the length of the string in Unicode code points\.  | 
|  `toupper(string)` |  String |  String |  Converts the string to uppercase\.  | 
|  `tolower(string)` |  String |  String |  Converts the string to lowercase\.  | 
|  `substr(string1, x), or substr(string1, x, y)` |  String, number or string, number, number |  String |  Returns a substring from the index specified by the number argument to the end of the string\. If the function has a second number argument, it contains the length of the substring to be retrieved\. For example, `substr("xyZfooxyZ",3, 3)` returns `"foo"`\.  | 
|  `replace(string1, string2, string3)` |  String, string, string |  String |  Replaces all instances of `string2` in `string1` with `string3`\. For example, `replace("foo","o","0")` returns `"f00"`\.  | 
|  `strcontains(string1, string2)` |  String |  Number |  Returns 1 if `string1` contains `string2` and 0 otherwise\.  | 

**Datetime Functions**

You can use datetime functions in the `filter` and `fields` commands and as arguments for other functions\. You can use these functions to create time buckets for queries with aggregate functions\.

As part of datetime functions, you can use time periods that consist of a number and then either `m` for minutes or `h` for hours\. For example, `10m` is 10 minutes and `1h` is 1 hour\.


| Function | Arguments | Result Type | Description | 
| --- | --- | --- | --- | 
|  `bin(period)` |  Period |  Timestamp |  Rounds the value of `@timestamp` to the given period and then truncates\.  | 
|  `datefloor(a, period)` |  Timestamp, period |  Timestamp |  Truncates the timestamp to the given period\. For example, `datefloor(@timestamp, 1h)` truncates all values of `@timestamp` to the bottom of the hour\.  | 
|  `dateceil(a, period)` |  Timestamp, period |  Timestamp |  Rounds up the timestamp to the given period and then truncates\. For example, `dateceil(@timestamp, 1h)` truncates all values of `@timestamp` to the top of the hour\.  | 
|  `fromMillis(fieldname)` |  Number |  Timestamp |  Interprets the input field as the number of milliseconds since the Unix epoch and converts it to a timestamp\.  | 
|  `toMillis(fieldname)` |  Timestamp |  Number |  Converts the timestamp found in the named field into a number representing the milliseconds since the Unix epoch\.  | 

**IP Address Functions**

You can use IP address string functions in the `filter` and `fields` commands and as arguments for other functions\. 


| Function | Arguments | Result Type | Description | 
| --- | --- | --- | --- | 
|  `isValidIp(fieldname)` |  String |  Boolean |  Returns `true` if the field is a valid IPv4 or IPv6 address\.  | 
|  `isValidIpV4(fieldname)` |  String |  Boolean |  Returns `true` if the field is a valid IPv4 address\.  | 
|  `isValidIpV6(fieldname)` |  String |  Boolean |  Returns `true` if the field is a valid IPv6 address\.  | 
|  `isIpInSubnet(fieldname, string)` |  String, string |  Boolean |  Returns `true` if the field is a valid IPv4 or IPv6 address within the specified v4 or v6 subnet\. When you specify the subnet, use CIDR notation such as `192.0.2.0/24` or `2001:db8::/32`\.  | 
|  `isIpv4InSubnet(fieldname, string)` |  String, string |  Boolean |  Returns `true` if the field is a valid IPv4 address within the specified v4 subnet\. When you specify the subnet, use CIDR notation such as `192.0.2.0/24`\.  | 
|  `isIpv6InSubnet(fieldname, string)` |  String, string |  Boolean |  Returns `true` if the field is a valid IPv6 address within the specified v6 subnet\. When you specify the subnet, use CIDR notation such as `2001:db8::/32`\.  | 

**Aggregation Functions in the Stats Command**<a name="CWL_Insights_Aggregation_Functions"></a>

You can use aggregation functions in the `stats` command and as arguments for other functions\. 


| Function | Arguments | Result Type | Description | 
| --- | --- | --- | --- | 
|  `avg(NumericFieldname)` |  Numeric log field |  Number |  The average of the values in the specified field\.  | 
|  `count(fieldname) or count(*)` |  Log field |  Number |  Counts the log records\. `count(*)` counts all records in the log group, while `count (fieldname)` counts all records that include the specified field name\.  | 
|  `count_distinct(fieldname)` |  Log field |  Number |  Returns the number of unique values for the field\. If the field has very high cardinality \(contains many unique values\), the value returned by `count_distinct` is just an approximation\.  | 
|  `max(fieldname)` |  Log field |  Log field value |  The maximum of the values for this log field in the queried logs\.  | 
|  `min(fieldname)` |  Log field |  Log field value |  The minimum of the values for this log field in the queried logs\.  | 
|  `pct(fieldname, value)` |  Log field value, value |  Log field value |  A percentile indicates the relative standing of a value in a dataset\. For example, `pct(@duration, 95)` returns the `@duration` value at which 95 percent of the values of `@duration` are lower than this value, and 5 percent are higher than this value\.  | 
|  `stddev(NumericFieldname)` |  Numeric log field |  Number |  The standard deviation of the values in the specified field\.  | 
|  `sum(NumericFieldname)` |  Numeric log field |  Number |  The sum of the values in the specified field\.  | 

**Non\-Aggregation Functions in the Stats Command**<a name="CWL_Insights_Non-Aggregation_Functions"></a>

You can use non\-aggregation functions in the `stats` command and as arguments for other functions\. 


| Function | Arguments | Result type | Description | 
| --- | --- | --- | --- | 
|  `earliest(fieldname)` |  Log field |  Log field |  Returns the value of `fieldName` from the log event that has the earliest timestamp in the queried logs\.  | 
|  `latest(fieldname)` |  Log field |  Log field |  Returns the value of `fieldName` from the log event that has the latest timestamp in the queried logs\.  | 
|  `sortsFirst(fieldname)` |  Log field |  Log field |  Returns the value of `fieldName` that sorts first in the queried logs\.  | 
|  `sortsLast(fieldname)` |  Log field |  Log field |  Returns the value of `fieldName` that sorts last in the queried logs\.  | 