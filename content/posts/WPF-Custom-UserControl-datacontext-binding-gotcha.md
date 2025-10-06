---
title: "WPF Custom UserControl Datacontext Binding Gotcha"
date: 2017-03-15
type: post
tags: ["wpf","datacontext","binding",".net","programming"]
image: "/images/wpf-xaml.png"
---
Creating custom user controls in WPF and as well as fully supporting MVVM with binding is a pretty *straightforward* process, but there are a couple of things that, at a first glance, look like they should work but they're not. At least for me. One of those things is when you *bind* `DataContext` to code behind, or to `Self`.

## A simple custom user control

```xml
<UserControl x:Class="SomeWpfApp.CustomUserControl"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006" 
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008" 
             xmlns:local="clr-namespace:SomeWpfApp"
             DataContext="{Binding RelativeSource={RelativeSource Self}}"
             d:DataContext="{d:DesignInstance Type=local:CustomUserControl, IsDesignTimeCreatable=True}"
             mc:Ignorable="d" 
             d:DesignHeight="300" d:DesignWidth="300">
    <Grid>
        <TextBlock Text="{Binding CustomText, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}" />
    </Grid>
</UserControl>
```

Here, we specify a custom user control with a `TextBlock` that binds to a custom dependency property in code behind called `CustomText`.

```csharp
public static readonly DependencyProperty CustomTextProperty =
    DependencyProperty.Register(nameof(CustomText),
        typeof(string),
        typeof(CustomUserControl),
        new FrameworkPropertyMetadata(""));

public string CustomText
{
    get => (string)GetValue(CustomTextProperty);
    set => SetValue(CustomTextProperty, value);
}
```


## A problem and a solution

The problem with code above is that the `DataContext` binding chain is now *broken*.  
In WPF, if not specified otherwise, `DataContext` is passed from [Parent](https://msdn.microsoft.com/en-us/library/system.windows.frameworkelement.parent(v=vs.110).aspx) to it's children. But by binding `DataContext` to *Self* we've broken that chain of inheritance.

Which means that if you try to use this custom control somewhere else and you *bind* to `CustomText` property:

```xml
<CustomUserControl CustomText="{Binding SomeOtherProperty}"/>
```

`CustomUserControl` will expect that `SomeOtherProperty` is actually in its code behind. The solution for this problem is to move `DataContext` binding from parent to first child, like this:

```xml
<UserControl x:Class="SomeWpfApp.CustomUserControl"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006" 
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008" 
             xmlns:local="clr-namespace:SomeWpfApp"
             mc:Ignorable="d" 
             d:DesignHeight="300" d:DesignWidth="300">
    <Grid
        DataContext="{Binding RelativeSource={RelativeSource Mode=FindAncestor, AncestorType=local:CustomUserControl}}"
        d:DataContext="{d:DesignInstance Type=local:AbbeooLogonUserControl, IsDesignTimeCreatable=True}">

        <TextBlock Text="{Binding CustomText, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}" />
    </Grid>
</UserControl>
```

## Different problem and a not so good solution

90% of the time above specified solution will suffice, but there are times where you need to bind `DataContext` to parent. One of those times is when you have some custom properties that are directly manipulating some aspect of a parent element.  
Currently the only solution i found is to specifically point to a place from which to find a bound property. So something like this won't work:

```xml
<CustomUserControl CustomText="{Binding SomeOtherProperty}"/>
```

but if you, lets say, bind a `DataContext` to code behind in a control where you implement your custom control, you can do something like this:

```xml
<CustomUserControl 
    CustomText="{Binding SomeOtherProperty, RelativeSource={RelativeSource Mode=FindAncestor, AncestorType=RootControl}}"/>
```

Sometimes you need to do it like this if your `DataContext` is in another class:

```xml
<CustomUserControl 
    CustomText="{Binding DataContext.SomeOtherProperty, RelativeSource={RelativeSource Mode=FindAncestor, AncestorType=RootControl}}"/>
```

### Summing up
I hope this helped, if you have any questions feel free to contact me!
