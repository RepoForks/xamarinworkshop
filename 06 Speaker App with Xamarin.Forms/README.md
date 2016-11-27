# Conference App with Xamarin.Forms
To dig deeper into the workflow of Xamarin development, we will create a real application in the next modules: A conference app. To keep UI work simple, we will use Xamarin.Forms for this to create the UI only once and share it across the other platforms. Please keep in mind, that Xamarin.Forms layouts are defined in XAML and will be rendered to **native** controls on each platform.

We will create a conference app that will list all sessions with their speakers in seperate tabs. Both, sessions and speakers will also have details pages that will be shown when the user tabs on an item. This results in having three views for the app:

- `MainPage` with two taps for sessions and speakers
- `SessionDetailsPage` with session details
- `SpeakerDetailsPage` with speaker details

## 1. Create the base structure
Creating a new Xamarin.Forms project is similar to creating a Xamarin Platform app as we already did before. In Visual Studio on Windows it's clickinng <kbd>File</kbd> <kbd>New</kbd> <kbd>Project...</kbd>, navigating to the ***Cross-Platform*** section and selecting ***Blank Xaml App (Xamarin.Forms Portable)***. As we are creating a conference app here that uses Xamarin Forms. let's call the project "*Conference.Forms*" and the solution just "*Conference*".

![New Xamarin.Forms project in Visual Studio Screenshot](../Misc/vsnewxamarinformsproject.png)

Let's inspect the Portable Class Library *Conference.Forms*, as this will be our main workplace. As we can see, there have also been iOS, Android and Windows projects for us but they are all linked to the shared project and only have to be touched when we want to implement platform specific features.

The `App.xaml` and its according `App.xaml.cs` file are the main entry point for our Xamarin.Forms app. While the XAML file only defines application wide resources, the `App.xaml.cs` file holds the application lifecycle methods and  kicks off the Xamarin.Forms framework and your UI. The first thing, Xamarin.Forms does here is setting the entry point to a new instance of the `MainPage` class whose logic and layout we can also find in the shared project.

```csharp
MainPage = new Conference.MainPage();
```

As we are working with multiple views later, it is a good idea, to set up a `NavigationPage` container which is one of the [Xamarin.Forms pre-defined page containers](https://developer.xamarin.com/guides/xamarin-forms/controls/pages) that allows navigation between multiple pages and set its initial UI to the `MainPage` instance.

```csharp
public App()
{
    InitializeComponent();

    var content = new Conference.MainPage();
    MainPage = new NavigationPage(content);
}
```

## 2. Create the models
Now it's time to give thought to the models that we want to use. Actually, our conference should deal with two types of objects: **Speakers** and **Sessions**. So let's create a model class for each. Before, let's take a second and think about where to place them. In a cross-platform scenario and in general in every cleanly seperated code project we should also be aware of what's the best project to put a new function, class or service in!

As the models we want to create do not have any platform specifics and are not used in a specific scenario only, we should definitely put them into a new project that holds everything we want to span across multiple usage scenarios. For this, let's create a new ***Portable Class Library*** by right-clicking on the Solution in Visual Studio an selecting <kbd>Add</kbd> <kbd>New Project...</kbd>. In the ***Visual C#*** section, we will find the ***Class Library (Portable)*** project template. Let's call it ***Conference.Core*** and add it to our solution.

![Add Portable Class Library in Visual Studio Screenshot](../Misc/vsaddpcl.png)

> **Hint:**  At this point, we should take a short digression on Portable Class Libaries. In .NET, these are projects that combine a subset of APIs and Types and offer the developer to use these across multiple projects. Inside the settings of Portable Class Libaries, we can choose which platforms, we want to target and the Library will offer all of the overlapping APIs. Unfortunately, this drives fragmentation and whenever you want to increase the target platform range of the library, some APIs might get lost and you have to rethink your code. That is why the .NET Foundation introduced [.NET Standard](https://blogs.msdn.microsoft.com/dotnet/2016/09/26/introducing-net-standard/), which is a pre-defined set of APIs that every .NET platform has to implement. So whenever we choose .NET Standard as target, we can combine the library with any other .NET platform that supports this standard. And (surprise): Xamarin does!

To change the target platforms of our Portable Class Library, just right-click on it, choose <kbd>Properties</kbd> and click the ***Target .NET Platform Standard*** link, that you can find in the first ***Library*** tab, to make out Library super compatible and future proof.

![Change PCL target to .NET Standard in Visual Studio Screenshot](../Misc/vschangetonetstandard.png)

Now we can finally create the classes for the **Speakers** and **Sessions** models.

```csharp
public class Speaker
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Title { get; set; }
    public string Bio { get; set; }
    public string Email { get; set; }
    public string ImagePath { get; set; }
}
```

```csharp
public class Session
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Room { get; set; }
    public DateTime StartTime { get; set; }
    public string SpeakerId { get; set; }
}
```

The last thing, we need to do is connecting the Portable Class Library with our Xamarin.Forms project. For this, just right-click the ***References*** folder and select <kbd>Add Reference...</kbd>. A new Windows opens that allows to manage project references. In the ***projects*** tab, we find the recently created ***Conference.Core*** project and can add it to the Xamarin.Forms project by simply checking the box next to it.

![Add References in Visual Studio Screenshot](../Misc/vsaddreference.png)

## 3. Create the ViewModels
It's a common approach to use the Mode-View-ViewModel pattern (short: MVVM) in Xamarin.Forms and so will we do. This pattern enables us to have a very clean seperation between every section, which allows us to build our application open for extensions and have a nice seperation of code when it comes to sharing and reusability.

> **Hint:** One of MVVM's main characteristics is the *ViewModel*, which provides logic for the *View* without knowing the *View* itself. It basically is, what the *Contoller* does in other design patterns but can be reused in multiple Frontend projects, because it does not know anything about the View it serfes and only provides *Properties* and *Commands* (methods) that can be bound to the *View* later. This connection is called *Binding*. More information about MVVM can be found at the [MSDN library](https://msdn.microsoft.com/en-us/library/hh848246.aspx) or [Codeproject](http://www.codeproject.com/Articles/100175/Model-View-ViewModel-MVVM-Explained).

As we can use the ViewModel not only for our Xamarin.Forms project but also for other Frontend solutions that are based on MVVM later, it's good practice, to seperate the ViewModel from the Xamarin.Forms project. Please be aware that it does **not** belong to our *Conference.Portable* project, as this should only contain those parts of our application that could be shared with *any other* layer (event with the backend). So let's keep it clean and create another ***Portable Class Library*** just as we did in the previous section and call it "*Conference.Frontend*". Dont's forget, to switch it to *.NET Standard* before referencing it to the *Conference.Forms* project.

### 3.1 Create a ViewModel that implements INotifyPropertyChanged
Now we can create our ViewModel in the new project. Add a new class and name it `MainViewModel` as it should provide the properties and commands for our `MainPage`. The first thing we need to to, to make our new class to a ViewModel is implementing `INotifyPropertyChanged`. This is important for data binding in MVVM Frameworks and is an interface that, when implemented, lets our view know about changes to the model. Just copy the followning lines to your class to have a `OnPropertyChanged()` method, that will notify the View about changes, whenever it gets called.

```csharp
public class MainViewModel : INotifyPropertyChanged
{
    // Implementation of INotifyPropertyChanged
    public event PropertyChangedEventHandler PropertyChanged;
    private void OnPropertyChanged([CallerMemberName] string name = null) => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
}
```

### 3.2 Add a busy indicator
It's a good pracice to ptovide a busy indicator to the view sothat it can show  this information to the user. So let's add a property to our ViewModel that indicates, if we are currently calculating or downloading stuff.

```csharp
private bool isBusy;
public bool IsBusy
{
    get { return isBusy; }
    set { isBusy = value; OnPropertyChanged(); }
}
```

As you can see, we call `OnPropertyChanged()` in the setter to inform the View whenever the `IsBusy` property changes.

### 3.3 Create lists for Speakers and Sessions
We also need to add two lists that the view can use for displaying speakers and sessions to the user. For these, we should use the `ObservableCollection` class that is an implementaion of `List` that notifies the view not only when it got set but also when its children changed.

```csharp
private ObservableCollection<Speaker> _Speakers;
public ObservableCollection<Speaker> Speakers
{
    get { return _Speakers; }
    set { _Speakers = value; OnPropertyChanged(); }
}

private ObservableCollection<Session> _Sessions;
public ObservableCollection<Session> Sessions
{
    get { return _Sessions; }
    set { _Sessions = value; OnPropertyChanged(); }
}
```

This is everything we need to provide bindable properties to our `MainPage` for now. So let's move on and create ViewModels for the other views we have.

### 3.4 Create the other ViewModels
The ViewModels for `SessionDetailsPage` and `SpeakerDetailsPage` look similar but are much simpler as they only have to provide the selected speaker or session.

1. Create two new classes and name them `SessionDetailsViewModel` and `SpeakerDetailsViewModel`
1. Implement the `INotifyPropertyChanged` interface as we did it in the `MainViewModel` above
1. Copy the `PropertyChanged` event and `OnPropertyChanged` method from the `MainViewModel` to the new ViewModels
1. Add a bindable property for `CurrentSession` and `CurrentSpeaker` to the according ViewModels
    ```csharp
    private Speaker currentSpeaker;
    public Speaker CurrentSpeaker
    {
        get { return currentSpeaker; }
        set { currentSpeaker = value; OnPropertyChanged(); }
    }
    ```
    ```csharp
    private Session currentSession;
    public Session CurrentSession
    {
        get { return currentSession; }
        set { currentSession = value; OnPropertyChanged(); }
    }
    ```

That's it. Now we have three ViewModels, one for each view that we can bind to the UI that we will create in the next step.

## 4. Create the MainPage
Finally we can start creating the UI. As we remember from the start, we want to create three views for our application that we created ViewModels for. Let's begin creating MainPage and bind its UI to the ViewModel properties!

### 4.1 Create the tabs
When taking a look at the `MainPage.xaml` file, we can see, that Xamarin.Forms created a new page of type `ContentPage` for us. As we want to create a page with multiple tabs, we need to change this to a `TabbedPage`, which is a [pre-defined Xamarin.Forms Layout](https://developer.xamarin.com/guides/xamarin-forms/controls/layouts/). The tabs get defined by creating two `ContentPage`s inside which will contain the content of our session and speaker tabs.

```xml
<TabbedPage
    xmlns="http://xamarin.com/schemas/2014/forms"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    xmlns:local="clr-namespace:Conference"
    x:Class="Conference.Forms.MainPage"
    Title="Conference">

    <ContentPage Title="Sessions">
        <!-- Sessions Tab -->
    </ContentPage>

    <ContentPage Title="Speakers">
        <!-- Speakers Tab -->
    </ContentPage>
</TabbedPage>
```

Please notice, that we also added a `Title="Conference"` property to the `TabbedPage` root element to give the whole container a header. As we changed the page from a `ContentPage` to a `TabbedPage`, we also have to change this in the `MainPage.xaml.cs` code behind file.

```csharp
public partial class MainPage : TabbedPage
{
    // ...
}
```

### 4.2 Binding Context
Here we can also set the `BindingContext` of the page. It defines against which classes properties we can bind UI components to. As we are creating the `MainPage` here, the `BindingContext` should point to an instance of `MainViewModel`. Let's save the `MainViewModel` instance as a private field just in case we need to access it later.

```csharp
public partial class MainPage : TabbedPage
{
    private MainViewModel viewModel;

    public MainPage()
    {
        InitializeComponent();
        viewModel = new MainViewModel();
        BindingContext = viewModel;
    }
}
```

Now we can bind the MainPages's UI elements to the ViewModel's properties. Also, the view now gets informed, whenever something in the ViewModel changes, because of the `INotifyPropertyChanged` implemetation.

### 4.3 Session and Speaker Lists
Let's get back to the layout inside the `MainPage.xaml.cs` file and create the lists for sessions and speakers. In Xamarin.Forms lists are described with the `ListView` class. It has a property called `ItemsSource` which defines from which source it should display items and that is bindable. So we can bind it directly to the ViewModel.

```xml
<!-- Sessions Tab -->
<ContentPage Title="Sessions">
    <ListView ItemsSource="{Binding Sessions}" />
</ContentPage>
```

Inside the `ListView` we can also define, how a displayed item should look like. This has to be defined in the `ListView.ItemTemplate` property and can be any view. Everything, we create inside this template can also be bound to a single the current context.

> **Hint:** Every element in XAML has its own Binding Context. Inside a `ListView.ItemTemplate`, this context differs from the `ListView` itself. While it has the page's context (a `MainViewModel` in this case), it gives a single instance of its `ItemsSource` to its children. This means `ListView.ItemTemplate` has a single item as its Binding Context.

We could either define our own item layout here or chose one of the [predefined Xamarin.Forms Cells](https://developer.xamarin.com/guides/xamarin-forms/controls/cells/). The [Text Cell](https://developer.xamarin.com/api/type/Xamarin.Forms.TextCell/) looks like something we can use. It offers a primary and secondary text, which fits perfectly for the session list and can be bound to the `Name` and `Room` properties that we implemented in our `Session` model.



```xml
<!-- Sessions Tab -->
<ContentPage Title="Sessions">
    <ListView ItemsSource="{Binding Sessions}">
        <ListView.ItemTemplate>
            <DataTemplate>
                <TextCell 
                    Text="{Binding Name}"
                    Detail="{Binding Room}" />
            </DataTemplate>
        </ListView.ItemTemplate>
    </ListView>
</ContentPage>
```

The same can be used for the list of speakers, that we want to show in the second tab. Here we could use the predefined [Image Cell](https://developer.xamarin.com/api/type/Xamarin.Forms.ImageCell/), which is a Text Cell with a little image next to it, where we could show the speaker image that can be found in the `ImagePath` property of our `Speaker` class. Notice, that we just need to provide a URL to the Image Cell. Xamarin Forms handles the download and caching automatically for us.

```xml
<!-- Speakers Tab -->
<ContentPage Title="Speakers">
    <ListView ItemsSource="{Binding Speakers}">
        <ListView.ItemTemplate>
            <DataTemplate>
                <ImageCell
                    Text="{Binding Name}" 
                    Detail="{Binding Title}" 
                    ImageSource="{Binding ImagePath}" />
            </DataTemplate>
        </ListView.ItemTemplate>
    </ListView>
</ContentPage>
```

And that's it, we created the MainPage. We should try it out and run the application! Right-click on the platform projects for iOS, Android and Windows that you want to test your app on and select <kbd>Set as StartUp Project</kbd>. Now run your application.

![Screenshot of the current App status on iOS, Android and Windows](../Misc/formsconferenceempty.png)

Looks good, but pretty empty, mh? That's because we don't have any data to display yet. Let's move over to the next section and change that!

## 5. Get the data
Communicating with a potential Backend or managing data is something that is not platform or Xamarin.Forms specific, so let's get back to the shared *Conference.Forms* project where our ViewModels are located. Here we can implement our data management logic.

### 5.1 Creating the Conference Service
To keep the code seperation clean, we should create a new service class that handles data for us. Let's imagine, we had a Congerence Server that manages sessions and speakers somewhere in the cloud that we can connect to and create a new class called `ConferenceService`.

Inside, we should define two async methods that return a list of speakers and sessions and call them `GetSessionsAsync` and `GetSpeakersAsync`. We define these classes as async, because we don't know how long the server connection might take. To not block the UI thread, we can run them on a separate thread easily by defining them as `async`.

> **Hint:** In C#, you can easily let methods run on a separate thread by defining their execution asyncronously with the `async` keyword. These methods have to return a `Task` or `Task<WithReturnType>` then. You can call the method using the `await` keyword.

```csharp
public class ConferenceService
{
    public async Task<List<Session>> GetSessionsAsync()
    {
        return new List<Session>();
    }

    public async Task<List<Speaker>> GetSpeakersAsync()
    {
        return new List<Speaker>();
    }
}
```

### 5.2 Talking to a webserver via HttpClient
As you can see, we are just returning empty lists at the moment so it is time to fill them. For this, we need to talk a webserver, which is where the `HttpClient` class comes into play. Good thing, it is part of the .NET Standard, so we can use it easily on our Portable Class Library. `HttpClient` offers the very simple to use `GetStringAsync(string)` method, that sends a GET-Request to an URL and returns its string-content.

```csharp
public class ConferenceService
{
    private HttpClient httpClient;

    public ConferenceService()
    {
        httpClient = new HttpClient();
    }

    // ...
}
```

### 5.3 Download session and speaker information
We will be talking about backends in detail in the next module so let's just pretend we had a powerful backend here and mock the data. For this, I've prepated two JSON files for a list of demo sessions and speakers and [uploaded them to GitHub](). You can see, that they match the structure of our `Session` and `Speaker` classes.

As you will have noticed, the data is formatted in JSON, which is a common and widely spread standard for data exchange. We need a way to convert JSON to C# classes and vice versa so let's get some help from the community and add the great [Json.NET NuGet package](https://www.nuget.org/packages/newtonsoft.json/) to our project, which allows us to work with JSON easily.

![Add Json.NET NuGet package in Visual Studio Screenshot](../Misc/vsaddjsonnet.png)

Now we can download the mocked Session and Speaker lists form the GitHub servers and convert them into instances of our C# classes easily.

```csharp

```


So let's send our `HttpClient`'s GET-Request to them.



## 6. Create the Details Pages

## 7. Wrapping up what we have learned
- Explain Differnce between differnt PCLs (what are they for? With whom will they be shared?)
- structure is really important. Maybe add Solution Folders (Screenshot)