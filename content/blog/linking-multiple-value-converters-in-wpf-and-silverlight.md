---
title: Linking Multiple Value Converters in WPF and Silverlight
path: linking-multiple-value-converters-in-wpf-and-silverlight
date: 2010-07-28T18:44:14
description: I did this work ages ago and have been meaning to blog about it for some time. The problem I had, was that I wanted to bind the visibility property of an element to a boolean value.
tags:
- C#
- Silverlight
- WPF
- XAML
redirects:
 - /2010/07/28/Linking-Multiple-Value-Converters-in-WPF-and-Silverlight/
---
I started off with a simple visibility converter similar to those that can be found in numerous places on the web, i.e True is converted to visible and False is converted to collapsed. I called the VisibilityConverter; the code for this is shown below:

```csharp
public class VisibilityConverter : IValueConverter
{
    public object Convert(
        object value, 
        Type targetType, 
        object parameter,
        System.Globalization.CultureInfo culture)
    {
        if (!(value is bool))
            throw new ArgumentException("Argument 'value' must be of type bool");
 
        bool isVisible = (bool)value;
        return isVisible ? Visibility.Visible : Visibility.Collapsed;
    }
 
    public object ConvertBack(
        object value, 
        Type targetType,
        object parameter,
        System.Globalization.CultureInfo culture)
    {
        throw new NotSupportedException();
    }
}
```

What happens here is, in the Convert method the value parameter is the value that is passed in from the xaml binding. This value is cast to type bool and then the comparison operator is used to return either visible or collapsed depending on the value passed in from the binding.

Later on in the project I needed to reverse the relationship between the boolean value and the visibility, i.e. True was collapsed and False was visible. I could have created a copy of the original converter changed the relationship in code. However, I decided to go with a more reusuable mechanism, which reused the existing visibility converter, code reuse is always good right?

Firstly I needed a way to negate the boolean value of the binding. So for this, yes youve guessed it, I created another value converter creatively called NegateConverter.

```csharp
public class NegateConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, System.Globalization.CultureInfo culture)
    {
        return !(bool)value;
    }
 
    public object ConvertBack(object value, Type targetType, object parameter, System.Globalization.CultureInfo culture)
    {
        throw new NotSupportedException();
    }
}
```

OK so thats pretty straight forward it takes the boolean value flips it and then returns it.

So now I need a way to use both these converters together in the same binding, enter the SequentialValueConverter.

```csharp
public class SequentialValueConverter : List, IValueConverter
{
    public object Convert(
        object value,
        Type targetType,
        object parameter,
        System.Globalization.CultureInfo culture)
    {
        object returnValue = value;
 
        foreach (IValueConverter converter in this)
        {
            returnValue = converter.Convert(
                returnValue, 
                targetType,
                parameter,
                culture);
        }
 
        return returnValue;
    }
 
    public object ConvertBack(
        object value, 
        Type targetType, 
        object parameter, 
        System.Globalization.CultureInfo culture)
    {
        throw new NotSupportedException();
    }
}
```

The key thing to notice about this class is it still implements IValueConveter like the previous converters but it also inherits from the generic class List, so its a value converter that contains value converters. The Convert method takes the value parameter and stores it in the returnValue variable. Then it iterates through each value converter, calls the Convert method and stores the result in the returnValue variable. This value is then passed as the value for the next value converter.

So heres how to use it in the XAML.

```xml
<converters:sequentialvalueconverter x:key="negatedVisibilityConverter">
    <converters:negateconverter />
    <converters:visibilityconverter />
</converters:sequentialvalueconverter>
```

Notice that the NegateConverter and VisibilityConverter are children of the SequentialValueConverter.

Once you have declared the SequentialValueConverter in XAML you can use it like you would any other value converter.

I hope you find this useful. Thanks for reading.