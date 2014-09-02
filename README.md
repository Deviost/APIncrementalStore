APIncrementalStore
==================

Cutting the long story short, after StackMob [went to hell](https://blog.stackmob.com/2014/02/stackmob-announcement/) and all my development based on that SDK dragged along with it, I decided to implement my own `NSIncrementalStore` subclass.
Yes I have looked around for alternatives and even taking into account what is available, none of them are at the moment capable of supporting how I had designed my app.
I need an implementation that works most of time offline and sync with the backend BaaS when internet is available, which is quite the opposite of the APIs that I had found.
Special thanks to the great Mattt Thompson for the inspiring [AFIncrementalStore] (https://github.com/AFNetworking/AFIncrementalStore).
The code is based on Apple guidelines for [IncrementalStore Programming Guide] (https://developer.apple.com/library/mac/documentation/DataManagement/Conceptual/IncrementalStorePG/Introduction/Introduction.html#//apple_ref/doc/uid/TP40010706)

There are basically three main classes:

1) `APIncrementalStore` - this is the `NSIncrementalStore` subclass that implements what is required to handle the Core Data context.

2) `APDiskCache` - the APIncrementalStore uses this class as a local cache to respond to all the requests. This class exchanges NSDictionaries representations of the managed objects and uses a objectUID to uniquely identify them across the managed contexts (`NSIncrementalStore` and Disk Cache). This class uses a complete core data stack to store the cached objects. There will be one sqlite store for each logged user, so that on distinct cache per user.

3) `APParseSyncOperation` - Responsible for merging the local cache context in background as requested by the `APIncrementalStore`. After some great insights from last WWDC and to be inline with CloudKit aproach, this class is now a subclass of `APWebServiceSyncOperation`, which in turn is a sublass of `NSOperation`. This change was necessary to control how the operataion should behave when the app changes its states due to user actions. For example if the app goes into background it cancel all current operations and return an NSError. This was necessary as all Parse SDK fetch requests get interrupted imediatelly after the app go into background, therefore `APParseSyncOperation` is playing safe and aborting. All objects synced will remain persistant as it saves the contexts to disk per object basis.
Also with this change it become quite feasible to create another subclass of `APWebServiceSyncOperation` and implement another webservice intergration, perhaps using CloudKit next.

![Sync Process](https://raw.githubusercontent.com/flavionegrao/APIncrementalStore/gh-pages/images/Architecture.001.png)

Any question please drop me a message or open an issue.


###Additional columns to Parse classes
Two new columns are necessary on each Parse class that will sync with your Core Data Model:

1) *apObjectUID*: this is the unique identifier generated by APIncrementalStore. The Parse self created objectId is not being used as it can't be created localy, it requires that we save the object first then the SDK sets it to the object. As the main motivation of this repo is the capability of working most of time offline we need to use our own custom object identifier.

2) *apObjectStatus*: the clients don't delete any object, the objects are "marked" as deleted using this property to allow others to sync the deleted objects and properly update their databases. You may want to implement a Parse Cloud Code to execute a "garbage" collector say every 60 days to clean it, you just need to ensure that all clients have synced before you clean it. Also during the sync process, the webservice database may become inconsistent if the sync process of a given client gets interrupted before all objects are populated. The algorithm used by the `APParseSyncOperation` class enumarates all classes, populate its Parse objects and creates placeholders for the relationships if the related object doesn't exist. Problem may happen if any object doesn't get populated and other client syncs it to its cache. Objects with the status APObjectStatusCreated will not be returned by the `APIncrementalStore` to the Persisntant Coordinator until it becomes APObjectStatusPopulated.
 Three possible status are currently defined:

- APObjectStatusCreated - The object has been created as a placeholder from other object during sync process, the object is yet to be populated.
- APObjectStatusPopulated - The object has been fully populated and is ok to be returned by the APIncrementalStore to the requesing Persistant Coordinator
- APObjectStatusDeleted - The object has been deleted and will be removed from the Webserice database as in the near future.

3) *apObjectEntityName*: through this attribute the `APIncrementalStore` adds support to Core Data Entity Inheritance. Only the root entities will be created at Parse, all remaining subentieis will be identified by this attribute. Therefore all attributes and relationships from the root class and its sub-entities will share the same Parse Class.

###Parse Relationships
Please be mindful that relationships are being represented as Parse Pointers for Core Data To-One and Parse Relationships or Parse Array for Core Data To-Many relationships. 

#### Performance considerations
To-Many with a To-One as inverse are fine as the APParseConnector uses only the To-One side of the relationship to populate the local cache and rely on Core Data to fulfil the inverse relation (To-Many).
The problem surfaces when we are dealing with PFRelation, there's a huge bottleneck when syncing a large amount of objects, the reason is that in order to keep consistency fetched from the webservice (i.e. Parse) it is necessary to create and execute another query for every object that it contains. That means we are able to fetch in batches of say 1000 objects for Parse but then multiples queries subsequentely are necessary for each object fetched. That's how PFRelation (Parse) works, we can't ask for the related objects when executing the fetch, perhaps another baas provider might have a better solution but for now we need to put up with that. There's an alternative for PFRelation, wich is the Array, we may read more about the differences [Parse Relationship Documentation](https://www.parse.com/docs/relations_guide).
With Array it is possible to ask for the related objects to come along with the fetched objects, therefore there's no need for the further queries as explained above. There's a significant performance enhancement when changing from PFRelation to Array, however the objects can become large enough to hit de Parse 128K limit per object. So when architecturing your Model take it into account. The recommendation is to use Array on the side where you have less objects and PFRelation on the inverse.
The only way I was able to figure out how to let the Core Data Model be aware about when to use Array or PFRelation for the To-Many relations was to include some meta data directly into the model.

You MUST include the key `APParseRelationshipType` in the Core Data Model for every relationship user info with the values:
```
typedef NS_ENUM(NSUInteger, APParseRelationshipType) {
    APParseRelationshipTypeNonExistent = 0,
    APParseRelationshipTypeArray = 1,
    APParseRelationshipTypePFRelation = 2,
    
};
```
If no key is provided the `APIncrementalStore` will raise an Exception. Have a look at the `Tests/Model.xcdatamodeld` to see how it's implemented.


###Installation

#### API
Easy, you may clone it locally or use CocoaPods:

```
platform :ios, '7.0'
pod 'APIncrementalStore' , :git => 'https://github.com/flavionegrao/APIncrementalStore.git'
#Develop branch
#pod 'APIncrementalStore' , :git => 'https://github.com/flavionegrao/APIncrementalStore.git', :branch => 'development'
```

#### Additional Parse Cloud Code
Unfortunately Parse doesn't provide an out-of-box way to get its server time, and we need it to performe the sync process safely (see `[APParseConnector mergeRemoteObjectsWithContext:fullSync:onSyncObjecterror:]`).
The way I found to overcome it is to include a small Parse Cloud Code named `getTime` and query it using Parse SDK.
To get it working add to each of your apps that you are planing to sync the following cloud code:

```
Parse.Cloud.define("getTime", function(request, response) {
    Parse.Cloud.useMasterKey();
    var date = new Date();
    if (!date){
        response.error(date);
    } else {
        response.success(date);
    }
});
```

#### Core Data Integration
See the class named `CoreDataController` from the example app to see how I am implenting it.

### Run the example app

1) Download the project

2) Create a Parse account and a test App you may name it whatever you want.

3) On your new test Parse App create a class User and add a new object to it with username:test_user and password:1234

4) Navigate to the *Example* project directory in the Terminal

5) Run the following command: `pod install`  - [Install cocoapods](http://guides.cocoapods.org/using/getting-started.html#getting-started) if you don't have it.

6) Open `APIncrementalStore.xcworkspace` in Xcode

7) Navigate to the *Example* project's `AppDelegate.m` file and set the `APParseApplicationId` and `APParseClientKey`

8) You may run it on multiple devices (iOS Simulator + real iOS devices) to check to syncronization process

###Few tips when using the library:
- Don't forget to set Parse keys
- Login with an user (`PFUser`) and pass it as parameter to `APIncrementalStore`.
- `APParseConnector` will sync all entities found on your model and use the exactly same entity names to find the classes from Parse.

###Parse ACLs
`APIncrementalStore` tries to be as much agnostic as possible in regards to the backend webservice, thus pretty much only `APParseConnector` class deals with Parse. If you want to deal with ACLs I can see two ways you may achieve it:

1) A default ACL that you set when your user successfuly login at Parse:
```
PFACL *defaultACL = [PFACL ACL];
// Everybody can read objects created by this user
[defaultACL setPublicReadAccess:YES];
// Moderators can also modify these objects
[defaultACL setWriteAccess:YES forRoleWithName:@"Moderators"];
// And the user can read and modify its own objects
[PFACL setDefaultACL:defaultACL withAccessForCurrentUser:YES];
```
See [Parse Documentation](https://www.parse.com/docs/ios_guide#roles/iOS) for more information

2) If you want to set ACL individually to objects you have created localy add a property named *__ACL* to your core data entity. `APIncrementalStore` will add ACL to the equivalent Parse Object when it finds a managed object that is being synced and
 contains a binary (NSData) property called *__ACL*. The property must be set as Binary and his content should be a JSON object UTF-8 encoded Parse ACL. Follow the same Parse ACL structure found on the REST API methods:
 ```
 {
    "8TOXdXf3tz": { "write": true },
    "role:Members": { "read": true },
    "role:Moderators": {"write": true }
 }
 ```
 Use PFObject objectID to identify specific users or role:<Role Name> for roles.
 On the example project you will find a helper method that shows how to create and included it to a managed object.
 ```-[CoreDataController addWriteAccess:readAccess:isRole:forParseIdentifier:forManagedObject:]```
 
 To adjust your model and avoid including the *__ACL* property by hand to each one of your entities you may include a method to add it programatically:
 
 ```
 - (NSManagedObjectModel*) model {
    
    NSManagedObjectModel* model = [NSManagedObjectModel mergedModelFromBundles:nil];
    NSManagedObjectModel *adjustedModel = [model copy];
    
    for (NSEntityDescription *entity in adjustedModel.entities) {
        
        // Don't add properties for sub-entities, as they already exist in the super-entity
        if ([entity superentity]) {
            continue;
        }
        
        NSAttributeDescription *objectACLProperty = [[NSAttributeDescription alloc] init];
        [objectACLProperty setName:@"__ACL"];
        [objectACLProperty setAttributeType:NSBinaryDataAttributeType];
        [objectACLProperty setOptional:YES];
        [objectACLProperty setIndexed:NO];
        
        [entity setProperties:[entity.properties arrayByAddingObjectsFromArray:@[objectACLProperty]]];
    }
    return adjustedModel;
}
```
 
 The Parse iOS-SDK doesn't allow us to inspect any existing ACL unless you know the user/role and you ask
 for the existing previlegies on that object. Therefore 'APIncrementalStore' will only add ACLs to object, but it will not change any existent privileges.

###Unit Testing
On `UnitTestingCommon.m` config a valid Parse User/Password.
Use a test Parse App as it will include few additional classes needed for testing.

###Version history

####v.0.4.2
- Bug fixes as usual
- APParseSyncOperation sending Push Notification "content-available" through PFPush whenever a local updated object is merged. 
- Better support for Core Data predicates
- SyncOperation uses a second PersistantStoreCoordinator to enable the IncrementalStore to continue reading while the store is syncing.
- Migrate to single side relationship at Parse. All CoreData To-Many relationships need to include a key named APParseRelationshipTypeUserInfoKey under its userInfo Dictionary. PFRelation is disencourage to be used due to its complexity in the sync process, use Parse Array as much as possible.

####v.0.4.1
- Automatic Sync is executed after each context save that hits the 'APIncrementalStore'. Can be turned off by passing APOptionSyncOnSaveKey set to NO when initializing the Store.
- Bug Fix - When syncing a class that has its root class with a PFRelation that doesn't belong to this sincing class we should ignore it. This happens because Parse doesn't send not populated properties along with the fetched objects, which works great for inheritance and allows us to use only the root class to store all subclasses at Parse. The exception is for PFRelations, even being NULL Parse put it in the object. So we have to discaard it.

####v.0.4.0
- Changed APParseConnector to APParseSyncOperation (NSOperation subclass - see WWDC CloudKit View to understand why)
- Changed APObjectIsDeleted to APObjectStatus in order to achieve better control over objects sync process
- Added initial support to iOS APP lifeclycle (background)
- Unit tests ammended to reflect above changes
- Bug fixes as usual
- Example app enhancements

####v.0.3.1
- Added support to Parse Arrays (1st version - please report any bug)
- Added support for multiple apps to coexist using distinct local caches. See @protocol APWebServiceConnector -setEnvID:

####v.0.3.0
- Lots of bug fixes mainly related to inheritance

####v.0.2.9
- Added support to [Core Data Entity Inheritance](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreData/Articles/cdMOM.html#//apple_ref/doc/uid/TP40002328-SW11) 



###Disclaimer
APIncrementalStore is not affiliated, associated, authorized, endorsed by, or in any way officially connected with Parse.com, Parse Inc., or any of its subsidiaries or its affiliates. 
The official Parse web site is available at [www.parse.com](www.parse.com)

###License
APIncrementalStore is available under the MIT license. 
See the LICENSE file for more info.


Cheers. Flavio


