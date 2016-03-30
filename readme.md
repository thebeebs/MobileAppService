#Exercise 1: Create a temporary Mobile App

Go to the URL: [https://tryappservice.azure.com/](https://tryappservice.azure.com/)
Follow the instuctions to create a temporary Mobile App Service. Download the UAP application.

Once downloaded extract the files to a new folder. Build the project to restore the NuGet packages.

Open MainPage.xaml.cs. In this class you can see the way that your application code communicates with the cloud backend service to download objects from the service and to insert, update and delete objects.
At the top of the class, the field todoTable of type IMobileServiceTable<TodoItem> is initialized by calling App.MobileService.GetTable<TodoItem>(). This object is used throughout the class to perform strongly typed data operations for that table.

Build and run your app. When your client app runs, code in the OnNavigatedTo method of MainPage calls RefreshTodoitems which calls your backend service to retrieve any ToDo items stored in the backend database. 

As this is the first time the service has been accessed, it will automatically configure the database and run the Seed method that you looked at previously, which inserts two items. After a few moments, you should see the two items displayed in the client app UI.

Insert some new items in addition to the two items that were added by the Seed method in the service. 

If your development PC has the Windows 10 Mobile emulators installed, select one of the Mobile Emulator …  options from the Debug Target dropdown next to the Start Debugging Button. If you have a real phone device connected and enabled for Developer Mode, select Device from the Debug Target dropdown. If neither of these options are available, you will have to skip the rest of this exercise.

Start debugging. The app will start on your Mobile device or emulator and show the same items you entered into the desktop client and which you stored in your mobile backend service database (which is running locally on your PC at present).

#Exercise 2: Enable offline sync for your app

This exercise shows you how to add offline support to a UWP app using an Azure Mobile App backend. Offline sync allows end-users to interact with a mobile app--viewing, adding, or modifying data--even when there is no network connection. Changes are stored in a local database; once the device is back online, these changes are synced with the remote backend.

## Install the SQLite runtime for Universal Windows Platform.
- In Visual Studio, on the Tools menu, click Extensions and Updates
- In the left pane of the Extensions and Updates wizard, click Online
- In the search box at the top right of the window, enter SQLite    
- When the Search Results display, scroll down until you see SQLite for Universal App Platform. If this SDK is not already installed on your system, select this item, and then click the Download button
- When the UAC prompt displays, click OK.
- In the VSIX Installer window, click Install. After the extension installs, click Close.
- Click the Restart Now button on the Extensions and Updates window and wait for Visual Studio 2015 to restart

## Add a reference to the SQLite runtime dll to your project.
- In Solution Explorer, right click the References node in the project tree and click Add Reference to run the Reference Manager. 
- In the "Universal Windows" category, select the option "Extensions" in the navigation pane at the left.
- Select SQLite for Universal App Platform, and then click OK.

##Install the WindowsAzure.MobileServices.SQLiteStore NuGet package.
- In Solution Explorer, right click the project and click Manage Nuget Packages to run NuGet Package Manager. 
- In the "Online" tab, select the option "Include Prerelease" in the dropdown at the top. Search for SQLiteStore to locate the 2.0.0-beta of WindowsAzure.MobileServices.SQLiteStore. 
- Then, click Install to add the NuGet reference to the project.
- Click I Accept on the License Acceptance window.

##In MainPage.cs, replace the todoTable variable with the following code:
### Snippet: wholOfflineVar
    private IMobileServiceSyncTable<TodoItem> todoTable = App.MobileService.GetSyncTable<TodoItem>()

## Add the following function InitLocalStoreAsync this initialises the offline store. 
###Snippet: wholInitStore
    
    private async Task InitLocalStoreAsync(){
        if (!App.MobileService.SyncContext.IsInitialized)
        {
            var store = new MobileServiceSQLiteStore("localstore.db");
            store.DefineTable<TodoItem>();
            await App.MobileService.SyncContext.InitializeAsync(store);
        }		  
        await SyncAsync();
    }

## In InsertTodoItem, UpdateCheckedTodoItem, and ButtonRefresh_Click add a call to SyncAsync():

    await SyncAsync();

## Create a function called SyncAsync(). It contains some basic error handeling 
### Snippet: whol7sync
    private async Task SyncAsync() {
        String errorString = null;		  
        try
        {
            await App.MobileService.SyncContext.PushAsync();
            // first param is query ID, used for incremental sync
            await todoTable.PullAsync("todoItems", todoTable.CreateQuery()); 
        }		  
        catch (MobileServicePushFailedException ex)
        {
            errorString = "Push failed because of sync errors. " +
                        "You may be offine.\nMessage: " +
                        ex.Message + "\nPushResult.Status: " + 
                        ex.PushResult.Status.ToString();
        }
        catch (Exception ex)
        {
            errorString = "Pull failed: " + ex.Message +
            "\n\nIf you are still in an offline scenario, " +
            "you can try your Pull again when connected with " + 
            "your Mobile Service.";
        }
		  
        if (errorString != null)
        {
            MessageDialog d = new MessageDialog(errorString);
            await d.ShowAsync();
        }
    }
## Build the Project 
Test to see that the project now has offline support.

# Presentation - Adding Authorisation

##Add a backing variable in MainPage.xaml.cs: 

###Snippet: whol1variable

    private MobileServiceUser user;

## Add AuthenticateSync Method

###whol2auth
    // Define a method that performs the authentication process
    // using Facebook sign-in. 
    private async System.Threading.Tasks.Task AuthenticateAsync()
        {
            while (user == null)
            {
                string message;
                // This sample uses the Facebook provider.
                var provider = "Facebook";
             
                try
                {
                    // Sign-in using Facebook authentication.
                    user = await App.MobileService.LoginAsync(provider);
                    message = string.Format("You are now signed in - {0}", user.UserId);
                    MessageDialog d = new MessageDialog(message);
                    await d.ShowAsync();

                }
                catch (InvalidOperationException)
                {
                    message = "You must log in. Login Required";
                }
             
                var dialog = new MessageDialog(message);
                dialog.Commands.Add(new UICommand("OK"));
                await dialog.ShowAsync();
            }
        }
## Inside ButtonLogin_Click add a call to AuthenticateAsync

     await AuthenticateAsync();
     
## Set up Facebook key at http://developer.facebook.com
## Add key to Azure App Service http://portal.azure.com
## Add Addition of the [Authorize] attribute to the TodoItemController class, which restricts access to only authenticated users.

## Cache Authentication:
### Snippet: whol6auth

    private async System.Threading.Tasks.Task AuthenticateAsync()
    {
        string message = string.Empty;
        var provider = "Facebook";
     
        // Use the PasswordVault to securely store and access credentials.
        PasswordVault vault = new PasswordVault();
        PasswordCredential credential = null;
     
        while (credential == null)
        {
            try
            {
                // Try to get an existing credential from the vault.
                credential = vault.FindAllByResource(provider).FirstOrDefault();
            }
            catch (Exception)
            {
                // When no matching resource an error occurs, which we ignore.
            }
     
            if (credential != null)
            {
                // Create a user from the stored credentials.
                user = new MobileServiceUser(credential.UserName);
                credential.RetrievePassword();
                user.MobileServiceAuthenticationToken = credential.Password;
     
                // Set the user from the stored credentials.
                App.MobileService.CurrentUser = user;
     
                try
                {
                    // Try to return an item now to determine if the 
                    // cached credential has expired.
                    await App.MobileService.GetTable<TodoItem>().Take(1)
                            .ToListAsync();
                }
                catch (MobileServiceInvalidOperationException ex)
                {
                    if (ex.Response.StatusCode == 
                            System.Net.HttpStatusCode.Unauthorized)
                    {
                        // Remove the credential with the expired token.
                        vault.Remove(credential);
                        credential = null;
                        continue;
                    }
                }
            }
            else
            {
                try
                {
                    // Login with the identity provider.
                    user = await App.MobileService
                        .LoginAsync(provider);
     
                    // Create and store the user credentials.
                    credential = new PasswordCredential(provider,
                        user.UserId, user.MobileServiceAuthenticationToken);
                    vault.Add(credential);
                } 
            catch (InvalidOperationException)
            {
                message = "You must log in. Login Required";
            }
     
            var dialog = new MessageDialog(message);
            dialog.Commands.Add(new UICommand("OK"));
            await dialog.ShowAsync();
        }
    }



