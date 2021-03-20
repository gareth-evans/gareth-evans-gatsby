---
path: reactive-extensions-inotifypropertychanged
date: 2016-10-07T23:00:00.000Z
title: Reactive Programming using Reactive Extensions and INotifyPropertyChanged
description: Using Rx to create a stream of INotifyPropertyChanged events
tags:
  - C#
  - ReactiveExtensions
  - ""
---
I recently worked on a complex trade entry form in WPF that involved complex relationships between different fields on the form. When certain values were set on the form then other areas of the forms needed to be enabled, disabled or constrained in other ways.

Having used Reactive Extensions (Rx) extensively in previous projects I naturally leaned towards a reactive solution.

The core piece of this approach was an extension method for `INotifyPropertyChanged`.

```csharp
public static IObservable GetPropertyChangedStreamFor<TSource, TValue>(
    this TSource source, 
    Expression<Func<TSource, TValue>> propertyExpression)
        where TSource : INotifyPropertyChanged
{
    var propertyChangeStream = Observable
        .FromEventPattern<PropertyChangedEventHandler, PropertyChangedEventArgs>(
            h => source.PropertyChanged += h, 
            h => source.PropertyChanged -= h); 
    var propertyName = ExtractPropertyNameFromLambda(propertyExpression);
    var propertyGetter = propertyExpression.Compile();
		
    return propertyChangeStream
        .Where(ev => ev.EventArgs.PropertyName == propertyName)
        .Select(_ => propertyGetter(source));
}
```

The source parameter is the object that implements INotifyPropertyChanged and propertyExpression is a lambda that calls the property which you want to monitor. Lines 6-9 create a stream of PropertyChanged events. Line 11 compiles the expression so that it can be executed to retrieve the value of the property. Line 14 filters the property changed stream to only include events for the property referenced in the property expression. Line 15 executes the compiled function passing in the object to which the property belongs, which in turn will return the current value of the property.

The following example demonstrates the usage of the method:

```csharp
var person = new Person();
var firstNameStream = person.GetPropertyChangedStreamFor(x => x.FirstName);
firstNameStream.Subscribe(value => Console.WriteLine(value));
 
person.FirstName = "Gareth";
 
// Console output:
// Gareth
```

This is a more elegant solution than monitoring the PropertyChanged event explicitly as you would have to get the property name from the event args then use reflection to get the current value of that property. You would also have to manage the subscriptions to property change events, which gets really gnarly, very quickly. However, where this approach really comes into it’s own, is the composition of property change event streams.

On our person class we have two other properties, LastName and FullName. We want the FullName to be FirstName + LastName; historically how I’ve done this is to reassign the FullName property in both the setters of FirstName and LastName so that when either changes the FullName remains in sync.

Rx gives us the ability to combine streams of events to create a new composite event, allowing us to build complex behaviours more easily. The behaviour we want is when either FirstName or LastName changes we combine the latest of each of those property values to create the new full name. In the constructor of the Person class we combine streams for FirstName and LastName and assign the result to FullName.

```csharp
public Person()
{
    var firstNameStream = this.GetPropertyChangedStreamFor(x => x.FirstName);
    var lastNameStream = this.GetPropertyChangedStreamFor(x => x.LastName);
 
    firstNameStream
        .CombineLatest(lastNameStream, (firstName, lastName) => $"{firstName} {lastName}") 
	.Subscribe(fullName => this.FullName = fullName);
}
```

I’ve created a working example of this in the following LINQPad file [](https://gist.github.com/gareth-evans/0a3df69cfdd12674509dc9f411b1c740)[](https://gist.github.com/gareth-evans/0a3df69cfdd12674509dc9f411b1c740)[here](https://gist.github.com/gareth-evans/0a3df69cfdd12674509dc9f411b1c740)