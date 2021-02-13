# Executing Scripts

The script provides the ability to write custom functions in other programming languages and execute them within Stream applications. The custom functions using scripts can be defined via the function definitions and accessed in queries similar to any other inbuilt functions.

Scripts help to define custom functions in other programming languages such as javascript. This can eliminate the need for writing extensions to fulfill the functionalities that are not provided in Stream Applications or by its extension.

## Syntax

The syntax for defining the script is as follows.

```js
define function <function name>[<language name>] return <return type> {
    <function logic>
};
```
    
The defined function can be used in the queries similar to inbuilt functions as follows.

```json
<function name>( (<function parameter>(, <function parameter>)*)? )
```
    
Here, the <code>&lt;function parameter&gt;</code>'s are passed into the <code>&lt;function logic&gt;</code> of the definition as an <code>Object[]</code> with the name <code>data</code>.

The functions defined via the function definitions have higher precedence compared to inbuilt functions and the functions provided via extensions.

The following parameters are used to configure the function definition:

<table>
<thead>
<tr class="header">
<th>Parameter</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><code>&lt;function name&gt;</code></td>
<td>The name of the function created. (It is recommended to define a function name in <code>camelCase</code>.)</td>
</tr>
<tr class="even">
<td><code>&lt;language name&gt;</code></td>
<td>Name of the programming language used to define the script, such as <code>javascript</code>, <code>r</code>, or <code>scala</code></td>
</tr>
<tr class="odd">
<td><code>&lt;return type&gt;</code></td>
<td>The return type of the function. This can be <code>int, long, float, double, string, bool</code> or <code>object</code>. Here, the function implementer is responsible for returning the output according on the defined return type to ensure proper functionality.</td>
</tr>
<tr class="even">
<td><code>&lt;language name&gt;</code></td>
<td>The execution logic that is written in the language specified under the <code>&lt;language name&gt;</code>, where it consumes the <code>&lt;function parameter&gt;</code>'s through the <code>data</code> variable and returns the output in the type specified via the <code>&lt;return type&gt;</code> parameter.
</td>
</tr>
</tbody>
</table>

## Transform data using Custom Functions

To write custom function calls, follow the procedure below:

1. Open the GUI. Click on `Stream Apps` tab.

1. Click on **New** to start defining a new stream application.

1. Enter a **Name** as `TemperatureProcessing` or feel free to chose any other name for the stream application.

1. Enter a **Description**.
    
1. In the `TemperatureProcessing` stream application, define a source stream as follows.

    ```
    define stream TempStream (deviceID long, roomNo int, temp double);
    ```

1. Add sink stream to send results of script function

    ```sql
    @sink(type= 'c8streams', stream='DeviceTempStream', @map(type='json'))
    define stream DeviceTempStream (id string, temp double);
    ```

1. In this example, you can write a function that can be used to concatenate the room number and device ID as follows.
    
    ```js
    define function concatFn[javascript] return string {
        var str1 = data[0];
        var str2 = data[1];
        var str3 = data[2];
        var responce = str1 + str2 + str3;
        return responce;
    };
    ```

1. Add stream query to apply the script you wrote to the relevant attributes of the input stream definition.

    ```sql
    select concatFn(roomNo,'-',deviceID) as id, temp
    from TempStream
    insert into DeviceTempStream;
    ```
    
1. Save the stream application. The completed stream application is as follows.
    
    ```js
    @App:name("TemperatureProcessing")
    @App:description("Calculate an average temperature of the room")
    
    define stream TempStream (deviceID long, roomNo int, temp double);
    
    @sink(type= 'c8streams', stream='DeviceTempStream', @map(type='json'))
    define stream DeviceTempStream (id string, temp double);
    
    define function concatFn[javascript] return string {
            var str1 = data[0];
            var str2 = data[1];
            var str3 = data[2];
            var responce = str1 + str2 + str3;
            return responce;
    };
    
    select concatFn(roomNo,'-',deviceID) as id, temp
    from TempStream
    insert into DeviceTempStream;
    ```
   
## Transform complex json data using Custom Functions

Parsing complex JSON data would be good application to write custom functions. Consider that nested json data is received over an input stream. Defining a message schema while defining a stream as explained in [Consuming Data - Introduction](./consuming-data.md#introduction) can be cumbersome or error prone.

In the below example we will see how complex data can be parsed using custom javascript function.

To write custom function calls, follow the procedure below:

1. Open the GUI. Click on `Stream Apps` tab.

1. Click on **New** to start defining a new stream application.

1. Enter a **Name** as `ProcessEmployeeData` or feel free to chose any other name for the stream application.

1. Enter a **Description**.

1. Define an input c8db collection

    ```
    @source(type = 'c8db', collection = "CompanyXInputStream", collection.type="doc" , replication.type="global", @map(type='json'))
    define stream CompanyXInputStream (seqNo string, name string, address string);
    ```
   
1. Define an output stream   

    ```
    @sink(type = 'c8streams', stream = "CompanyXProfessionalInfo", replication.type="local")
    define table CompanyXProfessionalInfo (name string, workAddress string);
    ```   

1. Consider that `CompanyXInputStream` will receive employee data in below format.

    ```json
    {
      "seqNo": "1200001",
      "name": "Raleigh McGilvra",
      "address": {
        "permanent": {
          "street": "236  Pratt Avenue",
          "city": "Orchards",
          "state": "Washington",
          "country": "USA",
          "zip": "98662"
        },
        "work": {
          "street": "1746  Rosebud Avenue",
          "city": "Little Rock",
          "state": "Arkansas",
          "country": "USA",
          "zip": "72212"
        }
      }
    }
    ```

1. Consider that we want to convert `address.work` in the well formatted string.

1. Define a javascript function to process `address` field.

    ```js
    define function getWorkAddress[javascript] return string {
        work_address = JSON.parse(data[0]).work

        // Concatenate all the address fields as a single string
        formatted_address =  work_address.street + ", " + work_address.city + ", " + work_address.state + ", " + work_address.country + ", " + work_address.zip;
        return formatted_address
    };
    ```
   
1. Write a Stream Query to transfom data using `getWorkAddress` function.

    ```
    -- Data Processing
    @info(name='Query')
    select name, getWorkAddress(address) as workAddress
    from CompanyXInputStream
    insert into CompanyXProfessionalInfo;
    ```

1. Save the stream application. The completed stream application is as follows.

    ```js
    @App:name('ProcessEmployeeData')
    
    @source(type = 'c8db', collection = "CompanyXInputStream", collection.type="doc" , replication.type="global", @map(type='json'))
    define stream CompanyXInputStream (seqNo string, name string, address string);
    
    @sink(type = 'c8streams', stream = "CompanyXProfessionalInfo", replication.type="local")
    define table CompanyXProfessionalInfo (name string, workAddress string);
    
    define function getWorkAddress[javascript] return string {
        work_address = JSON.parse(data[0]).work
        formatted_address =  work_address.street + ", " + work_address.city + ", " + work_address.state + ", " + work_address.country + ", " + work_address.zip;
        return formatted_address
    };
    
    -- Data Processing
    @info(name='Query')
    select name, getWorkAddress(address) as workAddress
    from CompanyXInputStream
    insert into CompanyXProfessionalInfo;
    ```