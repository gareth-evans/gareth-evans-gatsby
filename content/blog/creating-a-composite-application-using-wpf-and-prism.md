---
title: Creating a Composite Application using WPF and Prism
path: creating-a-composite-application-using-wpf-and-prism
date: 2010-07-30T17:34:42
description: I've been doing some work with Prism lately, for those of you who dont know what Prism is you can check out the codeplex page here. The priciples in Prism are quite different to programming patterns Ive used before. So Ive written this sample app to share my findings.
tags:
- C#
- WPF
- XAML
- Prism
---

## Implementation

The first thing you need to do is download Prism from it codeplex site here. Once youve downloaded and extracted it open and build the CompositeApplicationLibrary_Desktop solution in the CAL directory.

Once thats built, create a new WPF application, Im going to call mine GDU, but you can call yours what ever you like. Now you need to reference those Prism assemblies. You can reference them from where they currently located, however that may get you into trouble with your colleagues if you are using source control, as they may not  have those assemblies in the same location on their machine if at all.

A much more elegant solution is to create a folder in the project called Composite Libraries, then right click on the folder and Add > Existing Item, navigate to the path [Prism Directory]CALDesktopComposite.UnityExtensionsbinDebug and add the follwoing files:

* Microsoft.Practices.Composite.dll
* Microsoft.Practices.Composite.Presentation.dll
* Microsoft.Practices.Composite.UnityExtensions.dll
* Microsoft.Practices.Unity.dll

After these file ahve been added, add references to these DLLs rather than to those in the original location, this means that when you check in your code those files will be included in the checkin so will be available to the rest of your team.

The default configuration for a WPF app is to have the StartupUri attribute set to the main window which is loaded when the application is executed. The model for prism applications is quite different. So got into App.xaml and remove the StartupUri attribute, then delete MainWindow.xaml from your project.

Now we need to create an interface called IShellView with a single method ShowView as shown below:
```csharp
public interface IShellView
{
    void ShowView();
}
```

After that create a new window and call it ShellView, then implement the IShellView interface in the ShellView code behind with the ShowView method calling the windows show method.

Next you need to create a class that will provide an instance of ShellView. For this created a class called ShellViewPresenter. This takes a IShellView as the only parameter in its constructor and then exposes that through a public property called  View. So far there hasnt bee much explaination of why were doing this, please bear with me, an explanation is coming after this next class, the Bootstrapper class.

```csharp
internal class Bootstrapper : UnityBootstrapper
{
    protected override void ConfigureContainer()
    {
        this.Container.RegisterType<IShellView, ShellView>();
        base.ConfigureContainer();
    }

    protected override IModuleCatalog GetModuleCatalog()
    {
        var catalog = new ModuleCatalog();
        //add modules here, we'll do this later
        return catalog;
    }

    protected override System.Windows.DependencyObject CreateShell()
    {
        ShellViewPresenter provider = this.Container.Resolve<ShellViewPresenter>();
        IShellView view = provider.View;
        view.ShowView();

        return view as DependencyObject;
    }
}
```

OK this is where all that hard work comes together, well almost anyway. You will notice that the first line in the ConfigureContainer method calls the RegisterType with two generic parameters, IShellView and ShellView. What this means is when the container tries to resolve a reference to IShellView it will create and instance of ShellView. The CreateShell uses that functionality.

The first line of CreateShell creates an instance of ShellViewPresenter using the Container.Resolve method. Internal what happens here is the method tries to create an instance of ShellViewPresenter and finds that it needs an object that implements IShellView, because ShellView is registered against IShellView an instance of ShellView is created and passed to the constructor of ShellViewPresenter and then that instance is passed back to CreateShell method. The ShowView method is then called to show the window.

At this point your could build and run. When you run the app ShellView will be shown as the applications main window, whoopy doo I hear you say, I had that before you started messing with my head. Yes you did, but now you have the basic foundation of a composite application.

There a just of couple things we need to do before we start creating modules. We need to create a region in the ShellView window where modules can be displayed. Firstly reference the Microsoft.Composite.Presentation namespace as illustrated in the fourth line in the code below. Secondly add the RegionName attached property to the Grid element, Ive called it MainRegion but you can call it what you like.

```xml
<Window x:Class="GDU.ShellView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:regions="clr-namespace:Microsoft.Practices.Composite.Presentation.Regions;
                            assembly=Microsoft.Practices.Composite.Presentation"
        Title="Shell" Height="300" Width="300">
    <Grid regions:RegionManager.RegionName="MainRegion">
 
    </Grid>
</Window>
```

## Creating a Reusable Module Framework

To continue with the MVVM pattern, we now want to create a standard C# Class Library; call it Core. Delete the Class1.cs file, then add the follwoing references to the project:

* PresentationCore
* PresentationFramework
* System.Xaml
* WindowsBase

To simlify the future development of modules well create two classes, ViewBase and ViewModelBase. ViewBase is the simplest it inherits from ContentControl  and has a single dependency property called HeaderProperty of type object, with a CLR property called Header.

The ViewModelBase is a bit more complicated, this class is inspired by Josh Smiths WPF Apps With The Model-View-ViewModel Design Pattern article on MSDN.

```csharp
public class ViewModelBase : INotifyPropertyChanged
{
    private bool throwOnInvalidPropertyName = true;
    
    protected bool ThrowOnInvalidPropertyName { get; set; }

    public event PropertyChangedEventHandler PropertyChanged;

    protected void RaisePropertyChanged(string propertyName)
    {
        this.VerifyPropertyName(propertyName);

        if (PropertyChanged != null)
            PropertyChanged(this, new PropertyChangedEventArgs(propertyName));
    }

    [Conditional("DEBUG")]
    [DebuggerStepThrough]
    public void VerifyPropertyName(string propertyName)
    {
        // Verify that the property name matches a real,
        // public, instance property on this object.
        if (TypeDescriptor.GetProperties(this)[propertyName] == null)
        {
            string msg = "Invalid property name: " + propertyName;

            if (this.ThrowOnInvalidPropertyName)
                throw new Exception(msg);
            else
                Debug.Fail(msg);
        }
    }
}
```

The class  implements the INotifyPropertyChanged interface as seen in many places on the net but whats clever about this class is that in debug mode it the verify property method checks that the property name passed in matches a real property.

## Our First Composite Application Module

Firstly, we need to create another standard C# Class Library, lets be creative and call it MyFirstModule. Delete Class1.cs from the project as we dont need that. Then add a reference the Core project and the following libraries:

* Microsoft.Practices.Composite
* Microsoft.Practices.Composite.Presentation
* Microsoft.Practices.Unity
* PresentationCore
* System.Xaml

This module is going to be a really simple UI which has a textbox, a button and a list box. When the user presses the button the name in the text box will be added to the list box.

Next we need to create some folders (this is the best practice thing I was talking about). Create two folders ViewModels and Views. In the ViewModels folder create and interface called IMyFirstViewModel. Heres what it should like:
```csharp
public interface IMyFirstViewModel
{
    DelegateCommand<string> AddNameToList { get; }
    ObservableCollection<string> Names { get; }
    IMyFirstView View { get; }
}
```

And heres the implementation:

```csharp
public class MyFirstViewModel : IMyFirstViewModel
{
    public MyFirstViewModel(IMyFirstView view)
    {
        this.Names = new System.Collections.ObjectModel.ObservableCollection<string>();
        this.AddNameToList = new DelegateCommand<string>(name => this.Names.Add(name));
        this.Names.Add("Balls");
        this.View = view;
        this.View.ViewModel = this;
    }

    public DelegateCommand<string> AddNameToList { get; private set; }
    public ObservableCollection<string> Names { get; private set; }
    public IMyFirstView View { get; set; }
}
```

In the costructor of the View model the AddNameToList is instatiated with an anonymous method where name will be the value coming from the view in the command binding. Also notice that the constructor requires an object that implements IMyFirstView, this will be injected in the same way I described previously. This is store against the View property then the ViewModel property is set to the current instance of MyFirstViewModel.

Now its time to create the view. Create a new interface in the Views folder called IMyFirstView, it has a single property called ViewModel of type IFirstViewModel.

```csharp
public interface IMyFirstView
{
    IMyFirstViewModel ViewModel { get; set; }
}
```

In the Views folder create a WPF UserControl called MyFirstView, in the code behind file inherit from the ViewBase class you created previously, you may need to use the fully qualified name to avoid any ambiguity with the System.Windows.Controls.ViewBase. MyFirstView also needs to implement IMyFirstView as shown below. Notice that the ViewModel property uses DataContext as a backing field, this is how the View is bound to the ViewModel.

```csharp
public partial class MyFirstView : Core.ViewBase, IMyFirstView
{
    public MyFirstView()
    {
        InitializeComponent();
    }

    public IMyFirstViewModel ViewModel
    {
        get { return DataContext as IMyFirstViewModel; }
        set { DataContext = value; }
    }
}
```

Weve just the base class from UserControl to ViewBase, because its a partial class we need to change the XAML file to reflect this. Open the MyFirstView.xaml file, import the Core namespace, call it core, and change the root element from UserControl to core:ViewBase, so your xaml file should look like this:

```xml
<core:ViewBase x:Class="MyFirstModule.Views.MyFirstView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:core="clr-namespace:GDU.Core;assembly=GDU.Core"
             mc:Ignorable="d"
             d:DesignHeight="300" d:DesignWidth="300">
    <StackPanel Margin="5">
        <TextBox x:Name="txtName"/>
        <Button Content="Add To List" Command="{Binding AddNameToList}"
             CommandParameter="{Binding ElementName=txtName, Path=Text}"/>
        <ListBox ItemsSource="{Binding Names}" />
    </StackPanel>
</core:ViewBase>
```

Ive also changed the root visual from Grid to StackPanel but you can layout the controls differntly if you prefer. There are three controls in the StackPanel, a TextBox, a Button and a ListBox. The buttons Command attributeis bound to the AddNameToList command of the viewmodel and the CommandParameter attribute is bound to the Text property on the textbox. Finally the listboxs ItemsSource is bound to the Names property of the viewmodel.

When Bob is entered in the textbox, when the button is clicked, the AddNameToList command is executed with Bob as a parameter, the anonymous execute method of the command is executed adding the Bob to the Names ObservableCollection.

We now need to create the module code which registers the view and viewmodel with the composite application. So create a new class called MyFirstModule that implements IModule.

```csharp
public class MyFirstModule : IModule
{
    private IUnityContainer container;
    private IRegionManager regionManager;

    public MyFirstModule(IUnityContainer container, IRegionManager regionManager)
    {
        if (object.ReferenceEquals(container, null))
            throw new ArgumentNullException("container");

        if (object.ReferenceEquals(regionManager, null))
            throw new ArgumentNullException("regionManager");

        this.container = container;
        this.regionManager = regionManager;
    }

    public void Initialize()
    {
        container.RegisterType<IMyFirstViewModel, MyFirstViewModel>();
        container.RegisterType<IMyFirstView, MyFirstView>();

        regionManager.RegisterViewWithRegion("MainRegion",
                            () => container.Resolve<IMyFirstViewModel>().View);
    }
}
```

You will notice that the constructor has two parameters container and regionManager these are injected when MyFirstModule is instantiated. In the Intialize method the view and viewmodel are registered with the container. In the RegisterViewWithRegion the first parameter is the region name we declared earlier in the shell, the anonymous method resolves IMyFirstViewModel to create an instance of MyFirstViewModel which when created will have a new instance of MyFirstView injected into the constructor which is accessed through the View property.