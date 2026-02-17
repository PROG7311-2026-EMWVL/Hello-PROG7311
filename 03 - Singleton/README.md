# 🚀 Implementing a Singleton Configuration Manager in C#

## 📌 Scenario:
You are developing a Gadget Store Management System that requires a centralized configuration manager to store global settings. The configuration settings should be accessible throughout the application, but only one instance of the configuration manager should exist at any time.

## ✅ Requirements:
1. Create a class named `ConfigurationManager` that follows the Singleton Pattern
2. The class should store and retrieve the following configuration settings:
   - Store Name: `"Gadget Galaxy"`
   - Currency: `"ZAR"`
   - Tax Rate: `15%` (default)
3. Ensure that only one instance of `ConfigurationManager` is created and shared.
4. Implement a method `UpdateTaxRate(double newRate)` that updates the tax rate and ensures the change is reflected across the application.
5. In the `Main` method:
   - Retrieve the singleton instance of `ConfigurationManager`.
   - Display the initial configuration settings.
   - Retrieve another instance and update the tax rate to 18%.
   - Display the settings again to confirm that both instances reflect the updated value.
   - Verify and print whether both instances are the same.

   ## A possible source to consult for different types of singletons
   - https://csharpindepth.com/articles/singleton