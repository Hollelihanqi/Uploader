# simple-uploader.js [![Build Status](https://travis-ci.org/simple-uploader/Uploader.svg?branch=master)](https://travis-ci.org/simple-uploader/Uploader?branch=master) [![codecov.io](http://codecov.io/github/simple-uploader/Uploader/coverage.svg?branch=master)](http://codecov.io/github/simple-uploader/Uploader?branch=master) [![Build Status](https://saucelabs.com/buildstatus/uploader)](https://saucelabs.com/u/uploader)

[![Sauce Test Status](https://saucelabs.com/browser-matrix/uploader.svg)](https://saucelabs.com/u/uploader)

[中文](./README_zh-CN.md)

![QQ](https://github.com/simple-uploader/Uploader/blob/develop/assets/simple-uploader-QQ-2.png?raw=true)

A JavaScript library providing multiple simultaneous, stable, fault-tolerant and resumable/restartable file uploads via the HTML5 File API.

Forked [flow.js](https://github.com/flowjs/flow.js) but refactor it.

The library is designed to introduce fault-tolerance into the upload of large files through HTTP. This is done by splitting each file into small chunks. Then, whenever the upload of a chunk fails, uploading is retried until the procedure completes. This allows uploads to automatically resume uploading after a network connection is lost either locally or to the server. Additionally, it allows for users to pause, resume and even recover uploads without losing state because only the currently uploading chunks will be aborted, not the entire upload.

Uploader (simple-uploader.js) does not have any external dependencies other than the `HTML5 File API`. This is relied on for the ability to chunk files into smaller pieces. Currently, this means that support is limited to Firefox 4+, Chrome 11+, Safari 6+ and Internet Explorer 10+.

Samples and examples are available in the `samples/` folder. Please push your own as Markdown to help document the project.

## New Features

* Treat `Folder` and `File` as `Uploader.File`

* Treat `Uploader` as a root `Folder`

* New `fileList` property which contains files and folders

## How can I install it?

Download a latest build from https://github.com/simple-uploader/Uploader/releases/
it contains development and minified production files in `dist/` folder.

or use npm:

```console
npm install simple-uploader.js
```

or use git clone

```console
git clone https://github.com/simple-uploader/Uploader
```

## How can I use it?

A new `Uploader` object is created with information of what and where to post:

```javascript
var uploader = new Uploader({
  target: '/api/photo/redeem-upload-token', 
  query: { upload_token: 'my_token' }
})
// Uploader isn't supported, fall back on a different method
if (!uploader.support) location.href = '/some-old-crappy-uploader'
```

To allow files to be either selected and drag-dropped, you'll assign drop target and a DOM item to be clicked for browsing:

```javascript
uploader.assignBrowse(document.getElementById('browseButton'))
uploader.assignDrop(document.getElementById('dropTarget'))
```

After this, interaction with Uploader.js is done by listening to events:

```javascript
uploader.on('fileAdded', function (file, event) {
  console.log(file, event)
})
uploader.on('fileSuccess', function (rootFile, file, message) {
  console.log(rootFile, file, message)
})
uploader.on('fileComplete', function (rootFile) {
  console.log(rootFile)
})
uploader.on('fileError', function (rootFile, file, message) {
  console.log(rootFile, file, message)
})
```

## How do I set it up with my server?

Most of the magic for Uploader.js happens in the user's browser, but files still need to be reassembled from chunks on the server side. This should be a fairly simple task and can be achieved in any web framework or language, which is able to receive file uploads.

To handle the state of upload chunks, a number of extra parameters are sent along with all requests:

* `chunkNumber`: The index of the chunk in the current upload. First chunk is `1` (no base-0 counting here).
* `totalChunks`: The total number of chunks.
* `chunkSize`: The general chunk size. Using this value and `totalSize` you can calculate the total number of chunks. Please note that the size of the data received in the HTTP might be greater than `chunkSize` of this for the last chunk for a file.
* `totalSize`: The total file size.
* `identifier`: A unique identifier for the file contained in the request.
* `filename`: The original file name (since a bug in Firefox results in the file name not being transmitted in chunk multipart posts).
* `relativePath`: The file's relative path when selecting a directory (defaults to file name in all browsers except Chrome).

You should allow for the same chunk to be uploaded more than once; this isn't standard behaviour, but on an unstable network environment it could happen, and this case is exactly what Uploader.js is designed for.

For every request, you can confirm reception in HTTP status codes (can be change through the `permanentErrors` option):

* `200`, `201`, `202`: The chunk was accepted and correct. No need to re-upload.
* `404`, `415`. `500`, `501`: The file for which the chunk was uploaded is not supported, cancel the entire upload.
* _Anything else_: Something went wrong, but try reuploading the file.

## Handling GET (or `test()` requests)

Enabling the `testChunks` option will allow uploads to be resumed after browser restarts and even across browsers (in theory you could even run the same file upload across multiple tabs or different browsers).  The `POST` data requests listed are required to use Uploader.js to receive data, but you can extend support by implementing a corresponding `GET` request with the same parameters:

* If this request returns a `200`, `201` or `202` HTTP code, the chunks is assumed to have been completed.
* If request returns a permanent error status, upload is stopped.
* If request returns anything else, the chunk will be uploaded in the standard fashion.

After this is done and `testChunks` enabled, an upload can quickly catch up even after a browser restart by simply verifying already uploaded chunks that do not need to be uploaded again.

## Full documentation

### Uploader

#### Configuration

The object is loaded with a configuration options:
```javascript
var r = new Uploader({ opt1: 'val', ...})
```
Available configuration options are:

* `target` The target URL for the multipart POST request. This can be a string or a function. If a
function, it will be passed a Uploader.File, a Uploader.Chunk and isTest boolean (Default: `/`)
* `singleFile` Enable single file upload. Once one file is uploaded, second file will overtake existing one, first one will be canceled. (Default: false)
* `chunkSize` The size in bytes of each uploaded chunk of data. The last uploaded chunk will be at least this size and up to two the size, see [Issue #51](https://github.com/23/resumable.js/issues/51) for details and reasons. (Default: `1*1024*1024`)
* `forceChunkSize` Force all chunks to be less or equal than chunkSize. Otherwise, the last chunk will be greater than or equal to `chunkSize`. (Default: `false`)
* `simultaneousUploads` Number of simultaneous uploads (Default: `3`)
* `fileParameterName` The name of the multipart POST parameter to use for the file chunk  (Default: `file`)
* `query` Extra parameters to include in the multipart POST with data. This can be an object or a
 function. If a function, it will be passed a Uploader.File, a Uploader.Chunk object and a isTest boolean
 (Default: `{}`)
* `headers` Extra headers to include in the multipart POST with data. If a function, it will be passed a Uploader.File, a Uploader.Chunk object and a isTest boolean (Default: `{}`)
* `withCredentials` Standard CORS requests do not send or set any cookies by default. In order to
 include cookies as part of the request, you need to set the `withCredentials` property to true.
(Default: `false`)
* `method` Method to use when POSTing chunks to the server (`multipart` or `octet`) (Default: `multipart`)
* `testMethod` HTTP method to use when chunks are being tested. If set to a function, it will be passed a Uploader.File and a Uploader.Chunk arguments. (Default: `GET`)
* `uploadMethod` HTTP method to use when chunks are being uploaded. If set to a function, it will be passed a Uploader.File arguments. (Default: `POST`)
* `allowDuplicateUploads ` Once a file is uploaded, allow reupload of the same file. By default, if a file is already uploaded, it will be skipped unless the file is removed from the existing Uploader object. (Default: `false`)
* `prioritizeFirstAndLastChunk` Prioritize first and last chunks of all files. This can be handy if you can determine if a file is valid for your service from only the first or last chunk. For example, photo or video meta data is usually located in the first part of a file, making it easy to test support from only the first chunk. (Default: `false`)
* `testChunks` Make a GET request to the server for each chunks to see if it already exists. If implemented on the server-side, this will allow for upload resumes even after a browser crash or even a computer restart. (Default: `true`)
* `preprocess` Optional function to process each chunk before testing & sending. To the function it will be passed the chunk as parameter, and should call the `preprocessFinished` method on the chunk when finished. (Default: `null`)
* `initFileFn` Optional function to initialize the fileObject. To the function it will be passed a Uploader.File arguments.
* `readFileFn` Optional function wrapping reading operation from the original file. To the function it will be passed the Uploader.File, the startByte and endByte, the fileType and the Uploader.Chunk. And should call the `readFinished` method with bytes arguments on the chunk when finished.
* `checkChunkUploadedByResponse` Optional function to check chunk was uploaded by XHR response. To the function it will be passed the Uploader.Chunk and the response message. You do not need to upload(test) all chunks now, see [Issue #1](https://github.com/simple-uploader/Uploader/issues/1) for details and reasons, [sample](https://github.com/simple-uploader/Uploader/blob/develop/samples/Node.js/public/app.js#L15).
* `generateUniqueIdentifier` Override the function that generates unique identifiers for each file. (Default: `null`)
* `maxChunkRetries` The maximum number of retries for a chunk before the upload is failed. Valid values are any positive integer and `undefined` for no limit. (Default: `0`)
* `chunkRetryInterval` The number of milliseconds to wait before retrying a chunk on a non-permanent error.  Valid values are any positive integer and `undefined` for immediate retry. (Default: `undefined`)
* `progressCallbacksInterval` The time interval in milliseconds between progress reports. Set it
to 0 to handle each progress callback. (Default: `500`)
* `speedSmoothingFactor` Used for calculating average upload speed. Number from 1 to 0. Set to 1
and average upload speed wil be equal to current upload speed. For longer file uploads it is
better set this number to 0.02, because time remaining estimation will be more accurate. This
parameter must be adjusted together with `progressCallbacksInterval` parameter. (Default 0.1)
* `successStatuses` Response is success if response status is in this list (Default: `[200,201,
202]`)
* `permanentErrors` Response fails if response status is in this list (Default: `[404, 415, 500, 501]`)
* `initialPaused` Initial paused state, default `false`.
* `processResponse` Process xhr response, default `function (response, cb) { cb(null, response) }`. After 0.5.2, `processResponse` will be called with arguments: (response, cb, Uploader.File, Uploader.Chunk).
* `processParams` Process xhr params, default `function (params) {return params}`. After 0.5.2, `processParams` will be called with arguments: (params, Uploader.File, Uploader.Chunk, isTest).

#### Properties

* `.support` A boolean value indicator whether or not Uploader.js is supported by the current browser.
* `.supportDirectory` A boolean value, which indicates if browser supports directory uploads.
* `.opts` A hash object of the configuration of the Uploader.js instance.
* `.files` An array of `Uploader.File` file objects added by the user (see full docs for this object type below).
* `.fileList` An array of `Uploader.File` file(folder) objects added by the user (see full docs for this object type below), but it treated Folder as a `Uploader.File` Object.

#### Methods

* `.assignBrowse(domNodes, isDirectory, singleFile, attributes)` Assign a browse action to one or more DOM nodes.
  * `domNodes` array of dom nodes or a single node.
  * `isDirectory` Pass in `true` to allow directories to be selected (Chrome only, support can be checked with `supportDirectory` property).
  * `singleFile` To prevent multiple file uploads set this to true. Also look at config parameter `singleFile`.
  * `attributes` Pass object of keys and values to set custom attributes on input fields.
   For example, you can set `accept` attribute to `image/*`. This means that user will be able to select only images.
   Full list of attributes: https://www.w3.org/wiki/HTML/Elements/input/file

   Note: avoid using `a` and `button` tags as file upload buttons, use span instead.
* `.assignDrop(domNodes)` Assign one or more DOM nodes as a drop target.
* `.unAssignDrop(domNodes)` Unassign one or more DOM nodes as a drop target.
* `.on(event, callback)` Listen for event from Uploader.js (see below)
* `.off([event, [callback]])`:
  * `.off(event)` Remove all callbacks of specific event.
  * `.off(event, callback)` Remove specific callback of event. `callback` should be a `Function`.
* `.upload()` Start or resume uploading.
* `.pause()` Pause uploading.
* `.resume()` Resume uploading.
* `.cancel()` Cancel upload of all `Uploader.File` objects and remove them from the list.
* `.progress()` Returns a float between 0 and 1 indicating the current upload progress of all files.
* `.isUploading()` Returns a boolean indicating whether or not the instance is currently uploading anything.
* `.addFile(file)` Add a HTML5 File object to the list of files.
* `.removeFile(file)` Cancel upload of a specific `Uploader.File` object on the list from the list.
* `.getFromUniqueIdentifier(uniqueIdentifier)` Look up a `Uploader.File` object by its unique identifier.
* `.getSize()` Returns the total size of the upload in bytes.
* `.sizeUploaded()` Returns the total size uploaded of all files in bytes.
* `.timeRemaining()` Returns remaining time to upload all files in seconds. Accuracy is based on average speed. If speed is zero, time remaining will be equal to positive infinity `Number.POSITIVE_INFINITY`

#### Events

* `.change(event)` File input change event.
* `.dragover(event)` Drop area dragover event.
* `.dragenter(event)` Drop area dragenter event.
* `.dragleave(event)` Drop area dragleave event.
* `.fileSuccess(rootFile, file, message, chunk)` A specific file was completed. First argument `rootFile` is the root `Uploader.File` instance which contains or equal the completed file, second argument `file` argument is instance of `Uploader.File` too, it's the current completed file object, third argument `message` contains server response. Response is always a string. 
Fourth argument `chunk` is instance of `Uploader.Chunk`. You can get response status by accessing xhr 
object `chunk.xhr.status`.
* `.fileComplete(rootFile)` A root file(Folder) was completed.
* `.fileProgress(rootFile, file, chunk)` Uploading progressed for a specific file.
* `.fileAdded(file, event)` This event is used for file validation. To reject this file return false.
This event is also called before file is added to upload queue,
this means that calling `uploader.upload()` function will not start current file upload.
Optionally, you can use the browser `event` object from when the file was
added.
* `.filesAdded(files, fileList, event)` Same as fileAdded, but used for multiple file validation.
* `.filesSubmitted(files, fileList, event)` Same as filesAdded, but happens after the file is added to upload queue. Can be used to start upload of currently added files.
* `.fileRemoved(file)` The specific file was removed from the upload queue. Combined with filesSubmitted, can be used to notify UI to update its state to match the upload queue.
* `.fileRetry(rootFile, file, chunk)` Something went wrong during upload of a specific file, uploading is being 
retried.
* `.fileError(rootFile, file, message, chunk)` An error occurred during upload of a specific file.
* `.uploadStart()` Upload has been started.
* `.complete()` Uploading completed.
* `.catchAll(event, ...)` Listen to all the events listed above with the same callback function.

### Uploader.File

#### Properties

* `.uploader` A back-reference to the parent `Uploader` object.
* `.name` The name of the file(folder).
* `.averageSpeed` Average upload speed, bytes per second.
* `.currentSpeed` Current upload speed, bytes per second.
* `.paused` Indicated if file(folder) is paused.
* `.error` Indicated if file(folder) has encountered an error.
* `.isFolder` Indicated if file(folder) is an `Directory`.

If `.isFolder` is `false` then these properties will be added:

* `.file` The correlating HTML5 `File` object.
* `.relativePath` The relative path to the file (defaults to file name if relative path doesn't exist).
* `.size` Size in bytes of the file.
* `.uniqueIdentifier` A unique identifier assigned to this file object. This value is included in uploads to the server for reference, but can also be used in CSS classes etc when building your upload UI.
* `.chunks` An array of `Uploader.Chunk` items. You shouldn't need to dig into these.

#### Methods

* `.getRoot()` Returns the file's root Uploader.File instance in `uploader.fileList`.
* `.progress()` Returns a float between 0 and 1 indicating the current upload progress of the file.
* `.pause()` Pause uploading the file.
* `.resume()` Resume uploading the file.
* `.cancel()` Abort uploading the file and delete it from the list of files to upload.
* `.retry()` Retry uploading the file.
* `.bootstrap()` Rebuild the state of a `Uploader.File` object, including reassigning chunks and XMLHttpRequest instances.
* `.isUploading()` Returns a boolean indicating whether file chunks is uploading.
* `.isComplete()` Returns a boolean indicating whether the file has completed uploading and received a server response.
* `.sizeUploaded()` Returns size uploaded in bytes.
* `.timeRemaining()` Returns remaining time to finish upload file in seconds. Accuracy is based on average speed. If speed is zero, time remaining will be equal to positive infinity `Number.POSITIVE_INFINITY`
* `.getExtension()` Returns file extension in lowercase.
* `.getType()` Returns file type.

## Origin

Uploader.js was inspired by and evolved from https://github.com/flowjs/flow.js and https://github.com/23/resumable.js.
