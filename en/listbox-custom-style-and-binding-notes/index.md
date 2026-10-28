# Notes on Custom ListBox Control Styles and Data Binding


When binding a custom-styled ListBox in Silverlight, there are a few things to watch out for:

1. The properties specified by ItemTemplate can live in a separate resource file.
2. The object you bind to, say an ItemTestData, must inherit from the INotifyPropertyChanged interface, and every bound set property you use must call NotifyPropertyChanged("property name");
3. If you want to bind data to the Item object, the custom ItemTemplate style must declare the binding name in its root LayoutRoot.

Suppose the root is a Grid and an Item consists of 2 TextBlocks, each bound to Data1 and Data2 of an ItemTestData object. Then under the Grid you first need to declare an alias for ItemTestData:

```
<Grid.Resources>
      <src:ItemTestData x:Key="itemTestData"/>
</Grid.Resources>
注意src是ItemTestData所在的命名空间,x:Key为指定一个别名
```

```
然后在TextBlock中就可以指定绑定的属性
```

```
<Grid.Resources>
      <src:ItemTestData x:Key="itemTestData"/>
</Grid.Resources>
注意src是ItemTestData所在的命名空间,x:Key为指定一个别名
```

```
然后在TextBlock中就可以指定绑定的属性
```

```
<TextBlock x:Name="someData" Text="{Binding Data1}"/>
```

```
最后,别忘了指定ListBox的ItemsSource到一个ItemTestData的集合上.\\
```

If you want to add control events inside the Item, one more thing to watch out for: the resource can't go into the global resource XAML file, because event handling needs the cs file.

```

```

In the end I found that WPF's binding really is convenient and simple.

