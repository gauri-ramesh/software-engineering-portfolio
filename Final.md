Gauri Ramesh <br>
Software Engineering Final Exam Part 1

## Code Sample #1: CSV Generation
To find this file in the code base, navigate to this [link](https://github.com/skmshaffer/RAIK383H-SoftwareEngineeringProject/blob/ffd6954de93d4a38dc496cc26d048c64288a7f84/NelnetProject/Utilities/CsvGenerator.cs).

CsvGenerator.cs
```CSharp
    public static byte[] GenerateCsv<T>(IList<T> itemsToWriteList, ClassMap map = null)
    {
        using (var memoryStream = new MemoryStream())
        {
            TextWriter writer = new StreamWriter(memoryStream);
            var csv = new CsvWriter(writer);
            var records = itemsToWriteList;

            if (map != null)
            {
                csv.Configuration.RegisterClassMap(map);
            }

            csv.WriteRecords(records);
            writer.Flush();

            return memoryStream.ToArray();
        }
    }
```

Example of ClassMap used in `GenerateCsv` method:

```CSharp
public class TotalProfitReportItem
    {
        public string StudentFirstName { get; set; }
        public string StudentLastName { get; set; }
        public string FeeType { get; set; }
        public decimal Amount { get; set; }
    }

    public sealed class TotalProfitClassMap : ClassMap<TotalProfitReportItem>
    {
        public TotalProfitClassMap()
        {
            Map(m => m.StudentFirstName).Name("First Name");
            Map(m => m.StudentLastName).Name("Last Name");
            Map(m => m.FeeType).Name("Fee Description");
            Map(m => m.Amount).Name("Amount");
        }
    }
```

ResponseFactory.cs

```CSharp
public class ResponseFactory
{
    public static HttpResponseMessage ConstructCsvResponse(byte[] csvContentBytes)
    {
        var result = new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = new ByteArrayContent(csvContentBytes)
        };
        result.Content.Headers.ContentType = new MediaTypeHeaderValue("application/octet-stream");
        result.Content.Headers.ContentLength = csvContentBytes.Length;
        return result;
    }
}
```

### About This Code Chunk
This series of code chunks provides a full picture of the CSV Generation stack. The CSV Generation stack is used in several places throughout our application, from the Revenue Reports, to families viewing their students and invoices. <br/><br/>
 Since we interpreted the requirements as having to build a hands-off system, we chose to send the monthly reports as CSVs in an email to the administrator. Because of this, I chose to generate CSV reports on the backend to prevent duplicate code between report viewing and sending.  This operates with the help of CsvHelper, a NuGet package that helps generate CSVs from models. <br/><br/>
 `TotalProfitReportItem` is an example of a model class that describes the columns of one of our revenue reports. `TotalProfitReportItem` demonstrates low coupling by separating itself from any of the other reports. By creating a model for each report type, it becomes easy to change what information is displayed in the report and create new reports. ClassMaps are a feature in the CsvHelper library that allow you to customize how information is displayed in the CSV. For this, I used the ClassMap to rename the columsn. Additionally, this class demonstrates high cohesion by defining the CSV ClassMaps in the same place as the model definition. <br/><br/>
 `GenerateCsv` is where the CSV actually gets generated in the form of a byte array to prepare for sending it as an HTTP response. The goal of this method was to make it as abstract as possible since several parts of the application use this feature. This method belongs in the utility project to allow for reuse in multiple components. it prefers interfaces over their implementations by using an `IList`, so that if we choose to use something like a `LinkedList` in the future, we are able to without much refactoring. It also allows for an IList of any type `<T>` to be generated into a CSV. Finally, it makes use of C#'s default parameters so that even developers not using a model with a ClassMap can use this method.<br/><br/>
 Within the function, this code utilizes `using` statements to control resource usage. Additionally, `writer.Flush()` is called to prevent the `CsvWriter` from becoming overloaded. <br/><br/>
 The final chunk in this code sample is the `ConstructCsvResponse` method. This method custom creates the header and response of the CSV byte array using UTF-8 and specifying the length of the response. It displays good software engineering principles by making the representation of the data consistent both in the back-end and the front-end.



## Code Sample #2 : Finding Closest Date Algorithm in JavaScript

To find this file in the codebase, navigate to this [link](https://github.com/skmshaffer/RAIK383H-SoftwareEngineeringProject/blob/master/NelnetProject/Web/Nelnet_UI/source/Utils/date-utils.js).
```JavaScript
const parseDateTime = (invoiceData) => {
    let dateParseArray = [];
    invoiceData.forEach(element => {
        let date = new Date((Date.parse(element.InvoiceDate.substring(0, 10))));
        dateParseArray.push(date);
    });

    let today = new Date();
    dateParseArray = dateParseArray.sort((a, b) => {
        let distanceA = Math.abs(today - a);
        let distanceB = Math.abs(today - b);
        return distanceA - distanceB;
    }).filter((d) => {
        return d - today > 0;
    });

    if (dateParseArray[0] === undefined) {
        return "";
    }

    const nextScheduledMonth = dateParseArray[0].getUTCMonth() + 1;
    const nextScheduledDay = dateParseArray[0].getUTCDate();
    const nextScheduledYear = dateParseArray[0].getUTCFullYear();
    const dateString = `${nextScheduledMonth}/${nextScheduledDay}/${nextScheduledYear}`;

    return dateString;
};

export { parseDateTime };

```

Code Chunk in Use:

![ReportSummary](screens/regsummary.PNG)

### About this Code Chunk
* Uses JavaScript ES6 features (export)
* Uses a sorting algorithm to find the nearest date to today that is in the future.
* Adheres to single responsibility principle because all date-related functions are encapsulated in this utility class.

This sample of code takes data gathered by hitting the Invoice endpoint and calculates the date of the next invoice in it. It is used in two places: first, in the registration summary page, as shown below. It is also shown on the Update Card Information page on the payment user accounts. <br/><br/>
Some of the salient features of this code chunk include the use of features in JavaScript EcmaScript 6 (ES6). This includes the use of arrow functions, templated strings, and use of keywords `let` and `const`. This matches industry standard specifications of how to write JavaScript. Additionally, I configured ESLint, a JavaScript static code analysis tool, to enforce these styles throughout our JavaScript files. <br/><br/>
The sample uses a sorting algorithm to find the nearest date to today in the future, so it demonstrates understanding of last semester's Algorithms/Introduction to Software Engineering course. Additionally, this code is encapsulated in its own utility class to minimize code duplication and maximize code reuse. It anticipates change and demonstrates modularity - if we need more date utility functions, there is a place it can go without breaking other parts of the system.

## Code Sample #3: Report Generation
Of all of the reports I generated, I chose to showcase the Total Profit Report Engine.

`TotalProfitReportEngine.cs`

```CSharp
    public class TotalProfitReportEngine : ITotalProfitReportEngine
    {
        private readonly IInvoiceAccessor _invoiceAccessor;
        private readonly IInvoiceItemAccessor _invoiceItemAccessor;
        private readonly IStudentAccessor _studentAccessor;

        public TotalProfitReportEngine(IInvoiceAccessor ia, IInvoiceItemAccessor it, IStudentAccessor sa)
        {
            _invoiceAccessor = ia;
            _invoiceItemAccessor = it;
            _studentAccessor = sa;
        }


        public byte[] CreateReport(ReportPreferences preferences)
        {
            var recordsToWrite = CreateReportItemsFromDatabase(preferences);
            return CsvGenerator.GenerateCsv(recordsToWrite, new TotalProfitClassMap());
        }

        public IList<TotalProfitReportItem> CreateReportItemsFromDatabase(ReportPreferences preferences)
        {
            var invoiceItems = _invoiceItemAccessor.Get();
            IList<TotalProfitReportItem> reportItems = new List<TotalProfitReportItem>();

            var startDate = DateTime.Parse(preferences.StartTime);
            var endDate = DateTime.Parse(preferences.EndTime);

            foreach (var invoiceItem in invoiceItems)
            {
                LoadInvoiceItem(invoiceItem);
                if (invoiceItem.Type == InvoiceItemType.BaseTuition || invoiceItem.Type == InvoiceItemType.LateFee)
                {
                    if (startDate <= invoiceItem.Invoice.InvoiceDate && endDate >= invoiceItem.Invoice.InvoiceDate)
                    {
                        var type = invoiceItem.Type;
                        type = StringUtils.ConvertConstantToTitleCase(type);

                        reportItems.Add(new TotalProfitReportItem
                        {
                            StudentFirstName = invoiceItem.Invoice.Student.FirstName,
                            StudentLastName = invoiceItem.Invoice.Student.LastName,
                            FeeType = type,
                            Amount = Math.Round(invoiceItem.Amount, 2)
                        });
                    }
                }
            }

            return reportItems;
        }

        public ReportTotal GetTotal(ReportPreferences preferences)
        {
            var totalProfitEarned = CreateReportItemsFromDatabase(preferences).Sum(profitItem => profitItem.Amount);
            return new ReportTotal
            {
                Amount = Math.Round(totalProfitEarned, 2),
                Description = "Total Profit Earned In This Period"
            };
        }
    }
```
Code Chunk in Use:
![ReportSummary](screens/reportexample.PNG)

About This Code Chunk: <br/>
<br/>
This code chunk demonstrates good use of IDesign Architecture and dependency injection. As seen in the constructor of the TotalProfitReportEngine, because this is an engine, it is only communicating with the Accessors (the layer below it). Additionally, this code follows the dependency inversion principle by using constructor injection for all of the Accessors it is using. One large roadblock I ran into while writing all of the report engines was standardizing the way different languages represent dates. C# and JavaScript each use different ways of representing dates as strings. I fixed this by creating a C#-formatted date string on the JavaScript side, and allowing the report engines to parse it into DateTime objects. Because that is a piece of business logic, I put this code in the Engine layer. This sample demonstrates high cohesion because all methods relating to the `TotalProfitReportEngine` are encapsulated in one class, allowing for maximum code reuse. Efficiency and clean code was taken into account while writing this method - I took care to only create report column objects for those invoices that fall in the desired date range. This class implements the `ITotalProfitReportEngine` interface, which implements the `IReportEngine` interface. The `IReportEngine` interface contains methods for all of the bare-bones necessities of a revenue report - `CreateReportItemsFromDatabase`, the method that takes persisted data and constructs rows based on user preferences. There is the `ReportTotal`, which calculates any kind of aggregate amount the user wants, and finally, `CreateReport`, which creates the CSV to send up to the web layer. Finally, this Engine utilizes LINQ to help make the C# code clean and readable.
## Code Sample #4 : Standardizing Front-End Input Validation Error Messages

```javascript
const assignErrorClass = (viewModelAttribute, inputField, validator = true, message =
"The following field is required") => {
    if (viewModelAttribute === undefined) {
        badInput(inputField, message);
        $(inputField).addClass("is-invalid");
        return false;
    } else if (!validator) {
        badInput(inputField, message);
        $(inputField).addClass("is-invalid");
        return false;
    } else {
        goodInput(inputField);
        return true;
    }
};

const goodInput = (element) => {
    $(element).removeClass("is-invalid");
    $(`${element}-invalid`).css("visibility", "hidden");
};

const badInput = (id, message) => {
    $(id).addClass("is-invalid");
    $(`${id}-invalid`).css("visibility", "visible");
    $(`${id}-invalid`).text(message);
};
```

### About the Code Chunk:
This code standardizes front-end error error messages during input validation by providing a utility function that assigns Bootstrap error classes to Knockout observables based on an optional validation function and a custom error message. This sample of code, similar to the Closest Date Algorithm, makes use of ES6+ features. In the `goodInput` and `badInput` functions I use JQuery to dynamically add and remove error CSS classes. This adheres to software engineering principles by preventing code duplication throughout the file, since every input needs it during registration. Additionally, this adheres to the principles of developing good user interfaces by giving the user feedback as they interact with the application that is consistent both visually and structurally. The error message are displayed in a pink/red tone to indicate danger or error to the user. This function works in any input validation scenario because the validator and message parameters have default values. This means that even if you have input that requires no validation besides making sure it isn't empty, it can still be used.

## Code Sample #5 : Text Message Notification Accessor

```CSharp
using System.Diagnostics;
using Core.Accessors;
using Core.Models;
using Twilio;
using Twilio.Exceptions;
using Twilio.Rest.Api.V2010.Account;

namespace Accessors
{
    public class TextMessageAccessor : ITextMessageAccessor
    {
        public bool Send(PhoneNumber to, string body)
        {
            var accountSid = "AC9e977e44167ba6b169d6884c96a9903c";
            var authToken = "b96004eaed62d0eb9ba571a95048776d";

            TwilioClient.Init(accountSid, authToken);
            try
            {
                var message = MessageResource.Create(
                    to: new Twilio.Types.PhoneNumber(to.Number),
                    from: "+15312016725",
                    body: body);
                return true;
            }
            catch (ApiException e)
            {
                Debug.WriteLine(e.Message);
                Debug.WriteLine($"Twilio Error {e.Code} - {e.MoreInfo}");
                return false;
            }
        }
    }
}
```
### About this Code Chunk
This code uses the Twilio API to send automatic text message notifications to users when a payment is coming up. It belongs in the Accessor layer, and it only interacts with the Core layer directly below it. It uses credentials given by my account on Twilio to send a text message to the phone number gathered during registration. <br/><br/> First, this method implements `ITextMessageAccessor`, and uses a Unity DI container to achieve constructor injection. This code demonstrates adaptation to change. Our initial schema did not account for text message notifications, but it was designed such that this type of notification would be possible. We standardized the representation of `PhoneNumber` so it could be used by text message notification APIs by accessing a property directly without writing error-prone string manipulation code. <br/><br/>The structure of the workflow made it easy to integrate a third-party API with our home-grown code. It demonstrates good use of the library by researching the custom exceptions that belong to the API and catching these exceptions where the library is used, the most logical place, since this feature is not client-facing in the web app. I chose to not propagate these exceptions to the Engine level so that I could use Twilio's custom exceptions without importing the library again in another spot. `ApiException` is a custom Twilio exception that provides better error documentation than one a developer using it would have implemented, so it prevents reinventing the wheel. <br/><br/>
I avoided reinventing the wheel and stayed consistent with our existing notification workflow by using Twilio's C# NuGet package to call `messageResource.Create()` instead of formatting my own HTTP request to Twilio's endpoint in the Web layer, which the JavaScript documentation suggests. The final reason I preferred to write this on the C# side relates to volatility-based decomposition - client-facing code tends to change more frequently and more quickly than back-end code, and because this code is not volatile, it should be encapsulated with other non-volatile code. Finally, this code demonstrates consistency with the remainder of the code base. The `ITextMessageAccessor` and the `ISmtpEmailServerAccessor` both contain a `Send` method, and the `TextNotificationProcessing Engine` and the `EmailNotificationProcessingEngine` have similar workflows that follow from this acccessor. If a developer at Nelnet were to take on our project after handoff, it would be easy for them to navigate the structure of both notification workflows.



