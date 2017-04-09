---
layout: post
title: "Study123 Using Azure Media Service, Azure app service and Xamarin for better learning experience"
author: "Ching Chen"
author-link: "https://www.facebook.com/chenchingz"
# author-image: "{{ site.baseurl }}/images/authors/cc.jpg"
date: 2017-03-16
categories: [Azure App Service, Xamarin]
color: "blue"
excerpt: Using Azure Media Service, Azure app service and Xamarin to expand the learning experience from school to home, from home to everywhere.
language: English
verticals: [Education]
---

Study123

Traditional cram schools(Private learning center) are facing digital transformation challenge. Now it is not just an extension of learning experience from school to home. With cloud technology, self learning or after school learning can be very personalized and very efficient. Empowered by Azure Media service and Xamarin, secured on-demand learning video can be viewed on multi-platform to realize study everywhere. With Azure mobile app (offline sync and push notification) and machine learning, recommend content can be proactively pushed to parents and students. Student can access materials which they don¡¦t know, and expand the learning experience from TV to multiple screen, from TV to mobile, from home to everywhere.
The discussion is initiated in Oct. After all video content are encoded in media service, the main discussion focus is about the learning experience on the app. 
In March 2017,  An one day hackathon held in Study123, Microsoft Taiwan teamed up with the Xamarin app team to leverage Azure app service(mobile app) to implement the offline sync and push notification for better user experience. 


Major Core team members:
- Tony Li ¡V CTO, Study123
- Nad Wu ¡V App Engineer, Study123
- Ching Chen ¡V Technical Evangelist, Microsoft Taiwan

<img alt="whiteboard" src="{{ site.baseurl }}/images/Study123/IMAG1326.jpg" width="900">
<img alt="group photo" src="{{ site.baseurl }}/images/Study123/IMAG1329.jpg" width="900">
<img alt="group photo" src="{{ site.baseurl }}/images/Study123/IMAG1358.jpg" width="900">

 
## Customer profile ##

[Study123](https://www.study123.com.tw/) was founded at Taiwan in Oct. 2013. CEO, CTO and main investors has been in education industry for more than 20 years.  Study123 is the leading e-learning content and platform provider for junior/senior high school students in Taiwan since 2012 which totally subvert traditional learning method. They partnered with well-known cram school teachers to compile teaching materials and record learning videos for students to preview and review lessons after school which means students don¡¦t need to go to cram schools anymore. Moreover, students can watch learning videos on multiple devices and ask questions online. 
 
## Problem statement ##

Here is the main consideration of the app.
1.	All learning material, include video and books, are produced by themselves. Content copyright protection is very important. 
2.	Video content needs to be seen smoothly even in bad network bandwidth on legacy Android phone.
3.	Hardware integration (camera and Bluetooth device) is important to learning experience.
4.	Offline sync should include both files and data. 
5.	Proactively push notification for new content

The main focus on this hackfest is about the integration between Azure service and client app.

 
## Solution, steps, and delivery ##

The different components involved in the development:
1.	Building the Xamarin.Forms cross-platform UI to display protected Video content  
2.	Media Service for video encode, encryption and packaging
3.	Custom controls for specific hardware integration
4.	Offline sync user data and file from service to client and from client to service
5.	Push notification proactively when new content/report is ready
Key technology in this case

<img alt="Architecture Diagram" src="{{ site.baseurl }}/images/Study123/architecture.png" width="214">

1.	Azure media service is the solution to encode, encryption and package media content. 
2.	Xamarin is the solution for them to build a cross platform learning app. 
3.	Azure app service(mobile app) is to provide better service such as offline sync, data file sync, and push notification.
4.	Machine learning for better recommendation
The main focus in this hackfest is about the integration between app and server to provide a concrete learning app.  


-Use Azure Media Player to test play video on Azure Media Service
 
 <img alt="Media Player" src="{{ site.baseurl }}/images/Study123/mediaplayer.png" width="400">
 
-Xamarin.Forms cross-platform UI for protected on-demand video
A custom control is implemented for ink while video is playing

 <img alt="Media Player" src="{{ site.baseurl }}/images/Study123/appmedia.png" width="400">
 

-Push notification 

For push notification, we use notification hub only and notification through app service.
I provide the POC code below and then integrated to their existing Xamarin code.
Here is a part sample code about registration to notification hub
 
 ```csharp
   protected override void OnRegistered(Context context, string registrationId)
        {
            Log.Verbose(Constants.TAG, AppResource.ResisterTitle + registrationId);
            RegistrationID = registrationId;

            createNotification(AppResource.ResisterTitle,
                                AppResource.RegisterDesp);

            Hub = new NotificationHub(Constants.NotificationHubName, Con-stants.ListenConnectionString,
                                        context);
            try
            {               
                Hub.UnregisterAll(registrationId);
            }
            catch (Exception ex)
            {
                Log.Error(Constants.TAG, ex.Message);
            }
            // add tags if any
            var tags = new List<string>() { };

            try
            {
                var hubRegistration = Hub.Register(registrationId, tags.ToArray());
            }
            catch (Exception ex)
            {
                Log.Error(Constants.TAG, ex.Message);
            }
        }

  ```

-Synchronizing Offline Data with Azure Mobile Apps
To implement both file and data, we add a storage controller service for storage file sync up management.
 
 ```csharp
  public class VideoItemStorageController : StorageController<VideoItem>
    {
        [HttpPost]
        [Route("tables/VideoItem/{id}/StorageToken")]
        public async Task<HttpResponseMessage> PostStorageTokenRequest(string id, StorageTokenRequest value)
        {
            StorageToken token = await GetStorageTokenAsync(id, value);

            return Request.CreateResponse(token);
        }

        // Get the files associated with this record
        [HttpGet]
        [Route("tables/ VideoItem /{id}/MobileServiceFiles")]
        public async Task<HttpResponseMessage> GetFiles(string id)
        {
            IEnumerable<MobileServiceFile> files = await GetRecordFilesAsync(id);

            return Request.CreateResponse(files);
        }

        [HttpDelete]
        [Route("tables/ VideoItem /{id}/MobileServiceFiles/{name}")]
        public Task Delete(string id, string name)
        {
            return base.DeleteFileAsync(id, name);
        }

    }

  ```

-Add Offline Sync to Xamarin Project

Initialize the local Sore(SQLite) for data offline sync local storage
  ```csharp
  #if OFFLINE_SYNC_ENABLED
            var store = new MobileServiceSQLiteStore(offlineDbPath);
            store.DefineTable<VideoItem>();

            // Initialize file sync
            this.client.InitializeFileSyncContext(new VideoItemFileSyncHandler(this), store);

            //Initializes the SyncContext using the default IMobileServiceSyncHandler.
            //this.client.SyncContext.InitializeAsync(store);
            this.client.SyncContext.InitializeAsync(store, StoreTrackingOptions.NotifyLocalAndServerOperations);


            this.videoITemTable = client.GetSyncTable<VideoItem>();
#else
            this.todoTable = client.GetTable<TodoItem>();
#endif

  ```

-Add a FileHelp for file Syncup in Xamarin Client app
FileHelper
  ```csharp
  public class FileHelper
    {
        public static async Task<string> CopyVideoItemFileAsync(string itemId, string filePath)
        {
            IFolder localStorage = FileSystem.Current.LocalStorage;

            string fileName = Path.GetFileName(filePath);
            string targetPath = await GetLocalFilePathAsync(itemId, fileName);

            var sourceFile = await localStorage.GetFileAsync(filePath);
            var sourceStream = await sourceFile.OpenAsync(FileAccess.Read);

            var targetFile = await localStorage.CreateFileAsync(targetPath, CreationCol-lisionOption.ReplaceExisting);

            using (var targetStream = await targetFile.OpenAsync(FileAccess.ReadAndWrite))
            {
                await sourceStream.CopyToAsync(targetStream);
            }

            return targetPath;
        }

        /// <summary>
        /// Get Local File Path for file sync
        /// </summary>
        /// <param name="itemId"></param>
        /// <param name="fileName"></param>
        /// <returns></returns>
        public static async Task<string> GetLocalFilePathAsync(string itemId, string fileName)
        {
            IPlatform platform = DependencyService.Get<IPlatform>();

            string recordFilesPath = Path.Combine(await platform.GetTodoFilesPathAsync(), itemId);

            var checkExists = await FileSys-tem.Current.LocalStorage.CheckExistsAsync(recordFilesPath);
            if (checkExists == ExistenceCheckResult.NotFound)
            {
                await FileSystem.Current.LocalStorage.CreateFolderAsync(recordFilesPath, CreationCollisionOption.ReplaceExisting);
            }

            return Path.Combine(recordFilesPath, fileName);
        }

        /// <summary>
        /// Delete local file after sync complete
        /// </summary>
        /// <param name="fileName"></param>
        /// <returns></returns>
        public static async Task DeleteLocalFile-Async(Microsoft.WindowsAzure.MobileServices.Files.MobileServiceFile fileName)
        {
            string localPath = await GetLocalFilePathAsync(fileName.ParentId, file-Name.Name);
            var checkExists = await FileSys-tem.Current.LocalStorage.CheckExistsAsync(localPath);

            if (checkExists == ExistenceCheckResult.FileExists)
            {
                var file = await FileSystem.Current.LocalStorage.GetFileAsync(localPath);
                await file.DeleteAsync();
            }
        }
    }


  ```
-Add FileSync in offline sync call
  ```csharp
  #if OFFLINE_SYNC_ENABLED
        public async Task SyncAsync()
        {
            ReadOnlyCollection<MobileServiceTableOperationError> syncErrors = null;

            try
            {
                await this.client.SyncContext.PushAsync();

                //add file push async
                await this.videoITemTable.PushFileChangesAsync();

                await this.videoITemTable.PullAsync(
                    "allVideoItems",
                    this.videoITemTable.CreateQuery());
            }
            catch (MobileServicePushFailedException exc)
            {
                if (exc.PushResult != null)
                {
                    syncErrors = exc.PushResult.Errors;
                }
            }
            catch (Exception ex)
            {
                Debug.WriteLine(@"Exception: {0}", ex.Message);
            }
            if (syncErrors != null)
            {
                foreach (var error in syncErrors)
                {
                    if (error.OperationKind == MobileServiceTableOperationKind.Update && error.Result != null)
                    {
                        await error.CancelAndUpdateItemAsync(error.Result);
                    }
                    else
                    {
                        // Discard the change
                        await error.CancelAndDiscardItemAsync();
                    }

                    Debug.WriteLine(@"Error executing sync operation. Item: {0} ({1}). Operation discarded.", error.TableName, error.Item["id"]);
                }
            }
        } 
  internal async Task DownloadFileAsync(MobileServiceFile file)
        {
            var todoItem = await videoITemTable.LookupAsync(file.ParentId);
            IPlatform platform = DependencyService.Get<IPlatform>();

            string filePath = await FileHelper.GetLocalFilePathAsync(file.ParentId, file.Name);
            await platform.DownloadFileAsync(this.videoITemTable, file, filePath);
        }

        internal async Task<MobileServiceFile> AddImage(VideoItem todoItem, string imagePath)
        {
            string targetPath = await FileHelper.CopyVideoItemFileAsync(todoItem.Id, imagePath);
            return await this.videoITemTable.AddFileAsync(todoItem, Path.GetFileName(targetPath));
        }

        internal async Task DeleteImage(VideoItem todoItem, MobileServiceFile file)
        {
            await this.videoITemTable.DeleteFileAsync(file);
        }

        internal async Task<IEnumerable<MobileServiceFile>> GetImageFilesAsync(VideoItem todoItem)
        {
            return await this.videoITemTable.GetFilesAsync(todoItem);
        }

  ```

## Conclusion ##

The integration between Xamarin and Azure app service is smooth.
One good Feedback is that the support and integration with Azure is getting better. Using Xamrin and Azure app service provides a more efficient way for developer so that they can focus on their business. 
So far there are 5000+ videos on Media Service. Student can learn from the video, can do the test exam, and can ask question through the app. 
Personalization is the key to the next step. Azure machine learning has also been tested separately. 

This is just a start of the digital transformation in self learning platform.
We are looking forward to see how technology can change the learning experience.  


## Additional resources ##
Here is the reference document about offline sync and push notification

-[Offline Data Sync in Azure Mobile Apps](https://docs.microsoft.com/en-us/azure/app-service-mobile/app-service-mobile-offline-data-sync)
-[Enable Offline sync in Xamarin](https://docs.microsoft.com/en-us/azure/app-service-mobile/app-service-mobile-xamarin-forms-get-started-offline-data)

-[Connect to Azure Storage in your Xamarin.Forms app](https://docs.microsoft.com/en-us/azure/app-service-mobile/app-service-mobile-xamarin-forms-blob-storage)

-[Azure Push Notification](https://docs.microsoft.com/en-us/azure/app-service-mobile/app-service-mobile-xamarin-forms-get-started-push)

-[send push notification from Azure mobile app](https://developer.xamarin.com/guides/xamarin-forms/cloud-services/push-notifications/azure/)

-[Add push notifications to your Xamarin.Forms app](https://docs.microsoft.com/en-us/azure/app-service-mobile/app-service-mobile-xamarin-forms-get-started-push)

Reference Sample Code

-[Azure Mobile Apps - structured data sync with files](https://azure.microsoft.com/en-us/resources/samples/app-service-mobile-dotnet-todo-list-files/)


