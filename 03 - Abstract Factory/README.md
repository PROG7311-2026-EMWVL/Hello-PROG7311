# 🛒 Implementing an Abstract Factory for Galaxy Gadget Store  

## 📌 Scenario:  
You are developing an **online gadget store**, **Galaxy Gadget Store**, that sells various gadgets such as **smartwatches, VR headsets, and wireless earbuds**. The store wants to support **multiple brands** and ensure that new brands can be **easily added without modifying the existing code**.  

To achieve this, you must implement the **Abstract Factory Pattern**, which will allow the store to dynamically create gadgets based on the selected brand.  

## ✅ Requirements:  
1. **Create an interface `IGadget`** that represents a generic gadget with a method `GetDetails()`.  
2. **Implement three concrete gadgets:**  
   - ⌚ **SmartWatch**  
   - 🎮 **VRHeadset**  
   - 🎧 **WirelessEarbuds**  
3. **Define an abstract factory `IGadgetFactory`** that declares methods to create these gadgets.  
4. **Implement concrete factories `SamsungFactory` and `AppleFactory`**, each responsible for creating brand-specific gadgets.  
5. **Develop a `GalaxyGadgetStore` class** that:  
   - Accepts a **gadget factory** dynamically.  
   - Retrieves and displays the details of all gadgets for the selected brand.  
6. **In the `Main` method:**  
   - 🔍 Prompt the user to choose a **brand** (`Samsung` or `Apple`).  
   - 🏭 Instantiate the appropriate **gadget factory**.  
   - 📋 Display the available gadgets **for the selected brand**.  