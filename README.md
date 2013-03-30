#CollectionFS
Is a simple way of handling files on the web in the Meteor environment

Have a look at [Live example](http://collectionfs.meteor.com/)

CollectionFS is a mix of both Meteor.Collection and GridFS mongoDB.
Using Meteor and gridFS priciples we get:
* Security handling
* Handling sharing
* Restrictions eg. only allow certain content types, fields, users etc.
* Reactive data, the collectionFS's methods should all be reactive
* Abillity for the user to resume upload after connection loss, browser crash or cola in keyboard
* At the moment files are loaded into the client as Blob universal way of handling large binary data
* Create multiple cached versions, sizes or formats of your files and get an url to the file - or upload files to another service / server

##How to use?

####1. Install:
```
    mrt add collectionFS
```
*Requires ```Meteorite``` get it at [atmosphere.meteor.com](https://atmosphere.meteor.com)*

####2. Create model: [client, server]
```js
    ContactsFS = new CollectionFS('contacts');
```
*You can still create a ```Contacts = new Meteor.Collection('contacts')``` since gridFS maps on eg. ```contacts.files``` and ```contacts.chunks```*

####3. Adding security in model: [client, server]
*Only needed when using ```accounts-...``` (eg. removed the ```insecure``` package)*
```js
    ContactsFS.allow({
      insert: function(userId, myFile) { return userId && myFile.owner === userId; },
      update: function(userId, files, fields, modifier) {
            return _.all(files, function (myFile) {
              return (userId == myFile.owner);

        });  //EO interate through files
      },
      remove: function(userId, files) { return false; }
    });
```
*The collectionFS supports functions ```.allow```, ```.deny```, ```.find```, ```findOne``` used when subscribing/ publishing from server* 
*It's here you can add restrictions eg. on content-types, filesizes etc.*


####4. Disabling autopublish: 
* If you would rather not autopublish all files, you can turn off the autopublish option.  This is useful if you want to limit the number of published documents or the fields that get published *
#####[server]
```js
    // do NOT autopublish
    ContactsFS = new CollectionFS('contacts', { autopublish: false });

    // example #1 - manually publish with an optional param
    Meteor.publish('contacts.files', function(complete) {
      // sort by handedAt time and only return the filename, handledAt and _id fields
      return ContactsFS.find({ complete: complete }, {
              sort:{ handledAt: 1 }, 
              fields: { _id: 1, filename: 1, handledAt: 1}
      })
    });

    // example #2 - limit results and only show users files they own
    Meteor.publish('contacts.files', function() {
      if (this.userId) {
        return ContactsFS.find({ owner: this.userId }, { limit: 30 });
      }
    });    
```
#####[client]
```js
    // do NOT autosubscribe
    ContactsFS = new CollectionFS('contacts', { autosubscribe: false});

    // example #1 - manually subscribe and show completed only 
    // (goes with example #1 above)

    var showCompleteOnly = true;
    Meteor.subscribe('contacts.files', showCompleteOnly);
```

##Uploading file
####1. Adding the view:
```html
    <template name="queControl">
      <h3>Select file(s) to upload:</h3>
      <input name="files" type="file" class="fileUploader" multiple>
    </template>
```

####2. Adding the controller: [client]
```js
    Template.queControl.events({
      'change .fileUploader': function (e) {
         var files = e.target.files;
         for (var i = 0, f; f = files[i]; i++) {
           ContactsFS.storeFile(f);
         }
      }
    });
```
*ContactsFS.storeFile(f) returns fileId or null, actual downloads are spawned as threads. It's possible to add metadata: ```storeFile(file, {})``` - callback or eventlisteners are on the todo*

##Downloading file
####1. Adding the view:
```html
    <template name="fileTable">
      {{#each Files}}
        <a class="btn btn-primary btn-mini btnFileSaveAs">Save as</a>{{filename}}<br/>
      {{else}}
        No files uploaded
      {{/each}}
    </template>
```

####2. Adding the controller: [client]
```js
    Template.fileTable.events({
      'click .btnFileSaveAs': function() {
        ContactsFS.retrieveBlob(this._id, function(fileItem) {
          if (fileItem.blob)
            saveAs(fileItem.blob, fileItem.filename)
          else
            saveAs(fileItem.file, fileItem.filename);
        });
      } //EO saveAs
    });
```
*In future only a blob will be returned, this will return local file if available. The `Save as` calls the Filesaver.js by Eli Grey, http://eligrey.com - It doesn't work on iPad*

####3. Adding controller helper: [client]
```js
    Template.fileTable.helpers({
      Files: function() {
        return ContactsFS.find({}, { sort: { uploadDate:-1 } });
      }
    });
```

###Create server cache/versions of files and get an url reference
Filehandlers are serverside functions that makes caching versions easier. The functions are run and handled a file record and a blob / ```Buffer``` containing all the bytes.
* Return a blob and it gets named, saved and put in database while the user can continue. When files are created the files are updated containing link to the new file - all done reactivly live.
* If only custom metadata is returned without a blob / Buffer then no files saved but metadata is saved in database.
* If null returned then only filehandler name and a date is saved in database.
* If false returned the filehandler failed and it will be resumed later

Further details in server/collectionFS.server.js
```js
Filesystem.fileHandlers({
  default1: function(options) { //Options contains blob and fileRecord - same is expected in return if should be saved on filesytem, can be modified
    console.log('I am handling 1: '+options.fileRecord.filename);
    return { blob: options.blob, fileRecord: options.fileRecord }; //if no blob then save result in fileURL (added createdAt)
  },
  default2: function(options) {
    if (options.fileRecord.len > 5000000 || options.fileRecord.contentType != 'image/jpeg') //Save som space, only make cache if less than 1Mb
      return null; //Not an error as if returning false, false would be tried again later...
    console.log('I am handling 2: '+options.fileRecord.filename);
    return options; 
  },
  size40x40: function(options) {
    return null;
    // Use Future.wrap for handling async
    /*var im = npm.require('imagemagick'); // Add imagemagick package
    im.resize({
                srcData: options.blob,
                width: 40
           });*/
    console.log('I am handling: '+options.fileRecord.filename+' to...');
    return { extension: 'jpg', blob: options.blob, fileRecord: options.fileRecord }; //or just 'options'...
  },
  size100x100gm: function(options) {
    if (options.fileRecord.contentType != 'image/jpeg')
      return null; // jpeg files only  

    // Use Future.wrap for handling async
    /*
    var dest = options.destination('jpg').serverPath; // Set optional extension

    var gm = npm.require('gm'); // GraphicsMagick required need Meteor package
    gm( options.blob, dest).resize(100,100).quality(90).write(dest, function(err) {
        if(err) {
          // console.log 'GraphicsMagick error ' + err;
          return null;
        }
        else {
          // console.log 'Finished writing image.';
          return destination('jpg').fileURL; // We only return the path for the file, no blob to save since we took care of it
        }
      });
    */
    // I failed to deliver a fileUrl for this, 
    return null;
  }
});
```
*This is brand new on the testbed, future brings easy image handling shortcuts to imagemagic, maybe som sound/video converting and some integration for uploading to eg. google drive, dropbox etc.*

###Future:
* Handlebar helpers? ```{{fileProgress}}```, ```{{fileInQue}}```, ```{{fileAsURL}}```, ```{{fileURL _id}}``` etc.
* Test server side handling image size etc.
* When code hot deploy the que halts, could be tackled in future version of Meteor
* Deviates from gridFS by using string based files.length (Meteor are working on this issue)
* Prepare abillity for special version caching options creating converting images, docs, tts, sound, video, remote server upload etc.
* Make Meteor packages for `GraphicsMagick` etc.
###Notes:
* This is made as ```Make it work, make it fast```, well it's not fast - yet
* No test suite - any good ones for Meteor?
* Current code contains relics in form of logs and timers used in the example ```statistics``` and for debuggin