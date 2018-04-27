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

        private void LoadInvoiceItem(InvoiceItem invoiceItem)
        {
            invoiceItem.Invoice = _invoiceAccessor.Get(invoiceItem.Invoice.Id);
            var student = invoiceItem.Invoice.Student;

            if (student != null)
            {
                invoiceItem.Invoice.Student = _studentAccessor.Get(student.Id);
            }
        }
    }
```
Code Chunk in Use:
![ReportSummary](screens/reportexample.PNG)

## Code Sample #4 : Standardizing Front-End Error Validation 

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
* TODO
* TODO
* TODO

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
* TODO
* TODO
* TODO




