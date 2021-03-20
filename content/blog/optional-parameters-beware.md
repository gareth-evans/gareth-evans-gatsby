---
path: operational-parameters-beware
date: 2010-10-28T13:35:23.443Z
title: Optional Parameters Beware
description: Have you ever wondered how optional parameters in C# 4 work? No?
  Well I have. After a bit of playing around I discovered a potential for bugs
  when using them.
redirects:
 - /2010/10/28/Optional-Parameters-Beware/
---
You have a simple application that creates contacts. The Contact class is shared between other applications and as a result is part of an external assembly. The class has a single constructor which takes the contact name and address details as parameters. The last parameter Country has a default value of “UK”.

```csharp
public class Contact
{
    public string Name { get; set; }
    public string AddressLine1 { get; set; }
    public string AddressLine2 { get; set; }
    public string Town { get; set; }
    public string City { get; set; }
    public string Country { get; set; }
    public Contact(
        string name,
        string addressLine1,
        string addressLine2,
        string town,
        string city,
        string country = "UK")
    {
            Name = name;
            AddressLine1 = addressLine1;
            AddressLine2 = addressLine2;
            Town = town;
            City = city;
            Country = country;
    }
}
```

The use of the default value affords you the luxury assuming that unless otherwise specified the contact is in the UK. So the following code would print out “Gareth lives in UK”, not good English, but there you go.

```csharp
static void Main(string[] args)
{
    Contact contact = new Contact(
        "Gareth", 
        "My House", 
        "My Road", 
        "My Town", 
        "My City");
    Console.WriteLine("{0} lives in {1}", contact.Name, contact.Country);
    Console.ReadLine();
}
```

If this application had to be deployed to another country, the default country UK would not be appropriate. To avoid rebuilding the whole application (and any other applications that use this assembly) you could change and rebuild the external assembly use some assembly redirection and youve changed the default country without recompiling the original application. Right? Wrong!!

Take a look at the image below showing the disassembled application.

Reflector Disassembly of Contacts Application

Notice that the constructor for the Contact class has the value “UK” hard coded. This means that regardless of any changes made to the external assembly UK will always be passed as the country from the contacts application, until the application is rebuild against the modified external assembly.