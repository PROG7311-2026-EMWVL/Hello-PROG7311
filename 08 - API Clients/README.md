# 📝 Implementing the EskomSePush API Client

## 📌 Scenario:  
You are developing an .NET Core MVC API client to show loadshedding times. 

## Learning APIs with the ESP API 2.0

In this activity, we will:
* Learn about APIs and API clients
* Begin to use Postman
* Make our first API request in Postman
* Make our first API request in dotnet
* Explore how to convert JSON to C# objects

### Research!
* Find out what APIs and API clients are...
* I would begin by Googling and choosing some results
  * https://aws.amazon.com/what-is/api/ - its AWS!
  * https://www.postman.com/api-platform/api-client/ - its Postman!

### You need to build a client that incorporates the following functionality:
1. Check the api allowance (use /api_allowance)
1. Allow the user to search for their area (use /areas_search)
1. Allow the user to view the loadshedding details for their area (use /area)

### Steps:
1. We will be using this API: https://documenter.getpostman.com/view/1296288/UzQuNk3E
2. Sign up for a key here: https://eskomsepush.gumroad.com/l/api (DO NOT PAY!! - choose monthly plan at $0!)
3. Add the token as an environment variable as we did previously.
4. Give it a go and test the different endpoints.
5. Take one of the endpoints data and explore the JSON using: https://jsonformatter.curiousconcept.com/
6. We need to get this from a JSON format to C# objects (deserialization). To do this, we need to get the class structure by inputting the JSON here: https://json2csharp.com/. You can use these classes in your app.
7. Then, you need to make requests to the API using a relevant library from a controller.
8. Thereafter, you need to actually deserialize using a relevant library in the controller method.
9. Then output the data to a view using a model class as a template.

Code is in this repo for a basic app

### Next step
For practice... Build an API client that interests you using one of these free APIs: https://rapidapi.com/collection/list-of-free-apis

### You can consult these resources:
* https://www.csharp.com/article/consuming-asp-net-web-api-rest-service-in-asp-net-mvc-using-http-client/
* https://learn.microsoft.com/en-us/aspnet/web-api/overview/advanced/calling-a-web-api-from-a-net-client
* https://www.youtube.com/watch?v=LZJvdFDCKxM
