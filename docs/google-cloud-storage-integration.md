# Using Google Cloud Storage as your storage provider

This example shows how to add and retrieve files using Google Cloud Storage.
Additionally, this example will also show you how to list uploaded files and remove any of them.

*See production-ready code below.*

## Prerequisite

We will use next packages: *Request* (NPM), *Random* (meteor), *Underscore* (meteor)

```shell
meteor add underscore
meteor add random
meteor npm install request
```

### Step 1: install [google-cloud-storage](https://github.com/googleapis/nodejs-storage)

```shell
npm install @google-cloud/storage"
```

### Step 2: Setup your Google Cloud Storage

- Sign into the [Google Cloud Console site](https://console.developers.google.com)
- Go to your project and under Storage click on Create Bucket, and name your Bucket
  - Don't forget the name of this bucket, you'll need it later
- Under APIs & Auth click on Credentials
- Create an OAuth Service Account for your project (*if you don't already have one*)
- If you do not have a private key, click "*Generate new key*"
  - This will download a JSON file to your computer
  - Keep this JSON file, it is your key to the account, and you will need it later
  - Also, __GUARD THIS JSON FILE AS YOUR LIFE__. IF ANYONE GETS AHOLD OF IT, THEY WILL HAVE FULL ACCESS TO YOUR ACCOUNT!

```js
let gcloud, gcs, bucket, bucketMetadata, Request, bound, Collections = {};

if (Meteor.isServer) {
  // use require() as "'import' and 'export' may only appear at the top level"
  const Random = require('meteor/random');
  const Storage = require('@google-cloud/storage');
  gcs = new Storage('google-cloud')({
    projectId: 'YOUR_PROJECT_ID', // <-- Replace this with your project ID
    keyFilename: 'YOUR_KEY_JSON'  // <-- Replace this with the path to your key.json
  });
  bucket = gcs.bucket('YOUR_BUCKET_NAME'); // <-- Replace this with your bucket name
  bucket.getMetadata(function(error, metadata, apiResponse) {
    if (error) {
      console.error(error);
    }
  });
  Request = Npm.require('request');
  bound = Meteor.bindEnvironment(function(callback) {
    return callback();
  });
}

Collections.files = new FilesCollection({
  debug: false, // Set to true to enable debugging messages
  storagePath: 'assets/app/uploads/uploadedFiles',
  collectionName: 'uploadedFiles',
  allowClientCode: false,
  onAfterUpload(fileRef) {
    // In the onAfterUpload callback, we will move the file to Google Cloud Storage
    _.each(fileRef.versions, (vRef, version) => {
      // We use Random.id() instead of real file's _id
      // to secure files from reverse engineering
      // As after viewing this code it will be easy
      // to get access to unlisted and protected files
      const filePath = 'files/' + (Random.id()) + '-' + version + '.' + fileRef.extension;
      // Here we set the neccesary options to upload the file, for more options, see
      // https://googlecloudplatform.github.io/gcloud-node/#/docs/v0.36.0/storage/bucket?method=upload
      const options = {
        destination: filePath,
        resumable: true
      };

      bucket.upload(fileRef.path, options, (error, file) => {
        bound(() => {
          let upd;
          if (error) {
            console.error(error);
          } else {
            upd = {
              $set: {}
            };
            upd['$set'][`versions.${version}.meta.pipePath`] = filePath;
            this.collection.update({
              _id: fileRef._id
            }, upd, (updError) => {
              if (updError) {
                console.error(updError);
              } else {
                // Unlink original files from FS
                // after successful upload to Google Cloud Storage
                this.unlink(this.collection.findOne(fileRef._id), version);
              }
            });
          }
        });
      });
    });
  },
  interceptDownload(http, fileRef, version) {
    let ref, ref1, ref2;
    const path = (ref= fileRef.versions) != null ? (ref1 = ref[version]) != null ? (ref2 = ref1.meta) != null ? ref2.pipePath : void 0 : void 0 : void 0;
    const vRef = ref1;
    if (path) {
      // If file is moved to Google Cloud Storage
      // We will pipe request to Google Cloud Storage
      // So, original link will stay always secure
      const remoteReadStream = getReadableStream(http, path, vRef);
      this.serve(http, fileRef, vRef, version, remoteReadStream);
      return true;
    }
    // While the file has not been uploaded to Google Cloud Storage, we will serve it from the filesystem
    return false;
  }
});

if (Meteor.isServer) {
  // Intercept file's collection remove method to remove file from Google Cloud Storage
  const _origRemove = Collections.files.remove;

  Collections.files.remove = function(search) {
    const cursor = this.collection.find(search);
    cursor.forEach((fileRef) => {
      _.each(fileRef.versions, (vRef) => {
        let ref;
        if (vRef != null ? (ref = vRef.meta) != null ? ref.pipePath : void 0 : void 0) {
          bucket.file(vRef.meta.pipePath).delete((error) => {
            bound(() => {
              if (error) {
                console.error(error);
              }
            });
          });
        }
      });
    });
    // Call the original removal method
    _origRemove.call(this, search);
  };
}

function getReadableStream(http, path, vRef){
  let array, end, partial, remoteReadStream, reqRange, responseType, start, take;

  if (http.request.headers.range) {
    partial = true;
    array = http.request.headers.range.split(/bytes=([0-9]*)-([0-9]*)/);
    start = parseInt(array[1]);
    end = parseInt(array[2]);
    if (isNaN(end)) {
      end = vRef.size - 1;
    }
    take = end - start;
  } else {
    start = 0;
    end = vRef.size - 1;
    take = vRef.size;
  }

  if (partial || (http.params.query.play && http.params.query.play === 'true')) {
    reqRange = {
      start: start,
      end: end
    };
    if (isNaN(start) && !isNaN(end)) {
      reqRange.start = end - take;
      reqRange.end = end;
    }
    if (!isNaN(start) && isNaN(end)) {
      reqRange.start = start;
      reqRange.end = start + take;
    }
    if ((start + take) >= vRef.size) {
      reqRange.end = vRef.size - 1;
    }
    if ((reqRange.start >= (vRef.size - 1) || reqRange.end > (vRef.size - 1))) {
      responseType = '416';
    } else {
      responseType = '206';
    }
  } else {
    responseType = '200';
  }

  if (responseType === '206') {
    remoteReadStream = bucket.file(path).createReadStream({
      start: reqRange.start,
      end: reqRange.end
    });
  } else if (responseType === '200') {
    remoteReadStream = bucket.file(path).createReadStream();
  }

  return remoteReadStream;
}
```
