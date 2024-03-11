# ListBox控制样式自定义与绑定小记


Silverlight中绑定自定义样式的ListBox,要注意以下几点

1. ItemTemplate 指定的属性可以单独放在一个资源文件中.
2. 要绑定的对象,如一个ItemTestData,要从接口INotifyPropertyChanged继承,所使用到的绑定的set属性要调用NotifyPropertyChanged("属性名称");
3. 如果要绑定数据到Item对象, 自定义的ItemTemplate 样式,要在其根LayoutRoot 中指定绑定名称.

假设其根是一个Grid,一个Item由2个TextBlock组成,分别要绑定到一个ItemTestData对象的 Data1和Data2上去, 那么,在Grid下首先要指定一个ItemTestData的别名

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

如果在Item中要加入控件的事件,还要注意的就是资源不能放到全局的资源文件XAML中去了,事件处理需要cs文件

```

```

最后发现,WPF的绑定确实是方便又简单
```
