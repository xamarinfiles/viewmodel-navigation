# Another example of ViewModel navigation in Xamarin.Forms

A Xamarin.Forms app demonstrating ViewModel-to-ViewModel navigation adapted from Matthew Soucoup's original VM-first article ["Xamarin Forms - View Model First Navigation"](https://codemilltech.com/xamarin-forms-view-model-first-navigation/) from February 2016.

I've used my own version of VM-first navigation with a lot of success on several Xamarin.Forms projects in the nearly 2 years since [codemillmatt's](https://github.com/codemillmatt) article came out, tweaking my implementation each time based on different project requirements and smoothing out the rough edges with successive iterations.  The key differences in my version are:

1. I only use ContentPages and ignore TabbedPages and MasterDetailPages which simplifies the logic.  (I have had better luck implementing menus with vanilla Xamarin.Forms controls that look the same/similar on each OS than TabbedPages or MasterDetailPages which mimic the native OSes and look very different from each other.  I also avoid custom renderers wherever possible. More on that in a blog post later.)

2. I prefer a greater degree of ViewModel encapsulation and keeping the app state in cache (:+1: for [Akavache](https://github.com/akavache/Akavache)) for non-trivial projects.  In my version, I only pass the ViewModel type to the navigation service and let it instantiate both the ViewModel and its View (Page in Xamarin.Forms):

```csharp
public async Task PushAsync(Type viewModelType, bool animated = false)
{
    // Create the new View based on the ViewModel
    var view = InstantiateView(viewModelType);

    // Use XF's navigation to add the new View/Page to the stack
    await FormsNavigation.PushAsync((Page)view, animated);
}

private IViewFor InstantiateView(Type viewModelType)
{
    // Look up the View for the corresponding ViewModel
    var viewType = _viewModelViewDictionary[viewModelType];

    // Instantiate the ViewModel
    var viewModel = (BaseViewModel)Activator.CreateInstance(viewModelType);

    // Instantiate the View and set its context to the ViewModel
    var view = (Page)Activator.CreateInstance(viewType);
    view.BindingContext = viewModel;

    // Add the ViewModel reference to the View for later use
    var viewFor = (IViewFor)view;
    viewFor.ViewModel = viewModel;

    return viewFor;
}        
```

3. Many app designs have a shared page like a home page that child pages return to after popping themselves and any of their children off the navigation stack.  This helps to clean up memory (especially when large resource files are involved) and gives a more intuitive navigation flow when the user switches between parallel menu items either using the app's menu buttons or the phone's back button/gesture if it has one.  However, this shared page may not be the first/root page in the navigation stack.  For example, any login or join pages may be kept on the navigation stack beneath the shared page in case the user logs out or backs out of the app.  Therefore, I support navigating to and from the first/root page and any higher shared page on the stack in addition to the usual push, pop, and replace functionality:

```csharp
public interface INavigationService
{
    void RegisterViewModels(System.Reflection.Assembly asm);

    #region Navigation Methods

    Task PopAsync(bool animated = false);

    Task PopModalAsync(bool animated = false);

    Task PopToRootAsync(bool animated = false);

    Task PopToSharedAsync(bool animated = false);

    Task PushAsync(Type viewModelType, bool animated = false);

    Task PushFromRootAsync(Type viewModelType, bool animated = false);

    Task PushFromSharedAsync(Type viewModelType, bool animated = false);

    Task PushModalAsync(Type viewModelType, bool animated = false);

    Task ReplaceAsync(Type viewModelType, bool animated = false);

    #endregion
}
```
