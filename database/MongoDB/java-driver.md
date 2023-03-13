# MongoDB-Java-Driver

## GridFS

### download

​		从`MongoDB-GridFS` 中下载文件，可以通过 `GridFSBucket` 或者 `GridFsTemplate` 下载，前者是 `MongoDB-Driver` 提供的API，后者是`SpringBoot` 封装的工具。

```java
// GridFSBucket
// Create a gridFSBucket using the default bucket name "fs"
GridFSBucket gridFSBucket = GridFSBuckets.create(myDatabase);

// Create a gridFSBucket with a custom bucket name "files"
GridFSBucket gridFSBucket = GridFSBuckets.create(myDatabase, "files");

//DownloadFromStream
FileOutputStream streamToDownloadTo = new FileOutputStream("/tmp/mongodb-tutorial.pdf");
gridFSBucket.downloadToStream(fileId, streamToDownloadTo);
streamToDownloadTo.close();

//OpenDownloadStream
GridFSDownloadStream downloadStream = gridFSBucket.openDownloadStream(fileId);
int fileLength = (int) downloadStream.getGridFSFile().getLength();
byte[] bytesToWriteTo = new byte[fileLength];
downloadStream.read(bytesToWriteTo);
downloadStream.close();

//============================================================================================

//GridFsTemplate
GridFsTemplate template = this.getGridFsTemplate(bucketName);
Query query = new Query();// 封装你的查询条件
GridFSFile gridFsFile = template.findOne(query);
GridFsResource resource = template.getResource(gridFsFile);
```

​		从代码中知道，不论是`GridFSBucket` 还是`GridFsTemplate` 都是通过 `GridFSDownloadStream` 对象下载文件，该对象会每次从 `mongo` 中读取一个分片，直接将其转成 `byte[]` 保存在内存中，一个分片默认的大小是 `255KB` ，可以通过 `GridFSUploadOptions`  对象进行更改。