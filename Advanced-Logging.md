There is a goal to make logs easier to extract from whatever platform Couchbase Lite is running on, and this document will discuss how to do it.

### Initial Implementation

The initial implementation will be kept straight forward.  There will be a way to set the maximum size for log storage, and an API that provides a list or some other sort of iterator to the paths of the current log files so that they can be extracted.  Beyond that will be an application level concern about what to do with them.

Proposed API:
`void c4log_setMaxStorage(uint bytes);`<br>
`C4Slice[] c4log_getPaths(int* size);`

The first function will set the maximum amount of storage to be used by the log files, and the second will return an array of `C4Slice` instances that will each contain the absolute path to one of the log files.  These log files must then be decoded to readable format via a separate tool.

### Possible future enhancements

Integration with Sync Gateway for easy transfer off device?<br>
In process decoding?