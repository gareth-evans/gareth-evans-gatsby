---
title: Collapsing Controls Bound to Commands in WPF
path: collapsing-controls-bound-to-commands-in-wpf
date: 2010-10-24T16:24:32
description: I'm quite interested in user experience design, recently I decided that the default disabling of controls that are bound to commands in WPF is not always conducive to a good user experience.
tags:
 - C#
 - WPF
 - XAML
redirects:
  - /2010/10/24/Collapsing-Controls-Bound-to-Commands-in-WPF/
---

For example if a modal dialog to create a new contact has a required fields, it makes sense that the button to save the new contact is visible but disabled until the required fields have been completed. In fact it would look pretty weird to have the button hidden until the contact can be saved.

What about if a command is dependant on a role based security requirement. If a user does not have permission to create a new contact not only are you wasting screen real estate and cluttering the UI with commands that cannot be executed, you are also leaving a visual reminder that the user may not but trusted to run this command, which could be detrimental to the overall user experience.

## A simple solution
As I mentioned when a command in WPF cannot execute the control is disabled. So a simple solution to hide the control, is to bind the Visibility property to the IsEnabled property. To do this we first need to create a value converter to turn the boolean IsEnabled value to one of the Visibility enumerations.

## The Converter
```csharp
[MarkupExtensionReturnType(typeof(IValueConverter))]
public class BoolToVisibilityConverter 
    : MarkupExtension,  IValueConverter
{
    private static BoolToVisibilityConverter instance;

    public BoolToVisibilityConverter() { }

    public object Convert(
        object value, 
        Type targetType, 
        object parameter, 
        System.Globalization.CultureInfo culture)
    {
        if (value.GetType() != typeof(bool))
            throw new ArgumentException("value must of type bool");

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

    public override object ProvideValue(IServiceProvider serviceProvider)
    {
        if (instance == null)
            instance = new BoolToVisibilityConverter();

        return instance;
    }
}
```

So this is a fairly straight forward converter. It takes a value, makes sure its a boolean and then converts true to visible and false to collapsed. One thing to note here is Ive inherited from MarkupExtension as this simplfies the use of converters in XAML. You can see a good article on this written by Rakesh Gunijan.

## The XAML
```xml
<Button
    Command="{Binding SomeCommand}"
    Content="Some Command"
    Visibility="{Binding RelativeSource={RelativeSource Self}, 
        Path=IsEnabled, 
        Converter={local:BoolToVisibilityConverter}}"/>
```

The important part of the of the XAML above is the binding on the Visibility attribute. The binding specfies firstly the source to be the IsEnabled property of the button, and secondly the value be converted using the BooleanToVisibilityValueConverter.