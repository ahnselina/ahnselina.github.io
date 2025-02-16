---
title: docker registry 后端存储分析
date: 2019-04-16 17:19:30
categories:
- 云计算
tags: [docker, docker registry, 存储]
---


![](https://z3.ax1x.com/2021/05/04/gmv7wD.png)
<!-- more -->
Docker 的核心组件包括：
* Docker 客户端 - Client
* Docker 服务器 - Docker daemon
* Docker 镜像 - Image
* Registry
* Docker 容器 - Container:Docker 容器就是 Docker 镜像的运行实例。  
本文主要介绍docker Registry及其后端存储

## docker架构

![](https://z3.ax1x.com/2021/05/04/gmvOfA.jpg)

## docker registry简介
docker registry 用于存储和分发docker镜像（images）。
当前主要使用Docker Registry 2.0，又叫Distribution，其github库为：[https://github.com/docker/distribution](https://github.com/docker/distribution)
原先的1.0版本使用python实现，已经停止更新和维护  
目前的2.0版本使用GO语言实现，新版实现有如下特性：
* push 和 pull更快
* 实现效率更高
* 简化部署
* 可插拔的存储后端
* webhook 通知

### 为什么需要私有的registry
Registry 是存放 Docker 镜像的仓库，Registry 分私有和公有两种。
Docker Hub（https://hub.docker.com/） 是默认的 Registry，由 Docker 公司维护，上面有数以万计的镜像，用户可以自由下载和使用。
出于对速度或安全的考虑，用户也可以创建自己的私有 Registry。  

通常，使用docker拉取镜像是从docker的公共仓库（Docker Hub）中，安装docker之后就可以实现这点。用户如果有Docker Hub的账号，也可以push 镜像到Docker Hub.

对于有的用户或者企业，使用Docker Hub就够用了，但对另一部分则不够用，比如用户想为自己的软件产品维护一个私有的registry。此外，用户可能想部署自己的镜像仓库用于持续集成和测试。对于这类用户，部署自己的私有的registry实例是更好的选择。

## docker registry 存储
前面提到docker registry新版特性有可插拔的存储后端，也就是Docker 镜像仓库后端支持多种储存类型，如文件系统，s3-aws，azure，oss，swift等，主要分为两大类

* 本地储存
* 远程储存

>Registry 做的事情大致可分为：  
1. 加载读配置  
2. 注册handler  
3. 监听  

本质上 Registry 是个 HTTP 服务，启动后，监听在配置文件设定的某端口上。当 http 请求过来后，便会触发之前注册过的 handler。
Handler 包含 manifest、tag、blob、blob-upload、blob-upload-chunk、catalog 等六类，具体请可参考 registry 源码：/registry/handlers/app.go。
```go
	app.register(v2.RouteNameManifest, manifestDispatcher)
	app.register(v2.RouteNameCatalog, catalogDispatcher)
	app.register(v2.RouteNameTags, tagsDispatcher)
	app.register(v2.RouteNameBlob, blobDispatcher)
	app.register(v2.RouteNameBlobUpload, blobUploadDispatcher)
	app.register(v2.RouteNameBlobUploadChunk, blobUploadDispatcher)
```
配置文件包含监听端口、auth 地址、存储驱动信息、回调通知等。


Docker engine 与 registry （即：远程镜像仓库）的通信也有一套完整的 API，大致包含 pull、push 镜像所涉及的认证、授权、镜像存储等相关流程，具体请参考：[Registry API](https://github.com/docker/distribution/blob/master/docs/spec/api.md)。目前常用 registry 版本为 v2，registry v2 拥有断点续传、并发拉取镜像多层等特点。能并发拉取多层是因为镜像的元信息与镜像层数据分开存储，当 pull 一个镜像时，先进行认证获取到 token 并授权通过，然后获取镜像的 manifest 文件，进行 signature 校验。校验完成后，依据 manifest 里的层信息并发拉取各层。其中 manifest 包含的信息有：仓库名称、tag、镜像层 digest 等， 更多，请参考：[manifest 格式文档](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-1.md)。

各层拉下来后，也会先在本地进行校验，校验算法采用 sha256。Push 过程则先将镜像各层并发推至 Registry，推送完成后，再将镜像的 manifest 推至 Registry。  
**Registry 其实并不负责具体的存储工作，具体存储介质根据使用方来定，Registry 只是提供一套标准的存储驱动接口，具体存储驱动实现由使用方实现。**

目前官方 Registry 默认提供的存储驱动包括：微软 Azure、Google GCS、Amazon S3、Openstack Swift、本地存储等。若需要使用自己的对象存储服务，则需要自行实现 Registry 存储驱动。

当然还可以支持自定义的存储，只需要实现distribution/registry/storage/driver/storagedriver.go中如下接口即可：
```go
type StorageDriver interface {
   // Name returns the human-readable "name" of the driver, useful in error
   // messages and logging. By convention, this will just be the registration
   // name, but drivers may provide other information here.
   Name() string

   // GetContent retrieves the content stored at "path" as a []byte.
   // This should primarily be used for small objects.
   GetContent(ctx context.Context, path string) ([]byte, error)

   // PutContent stores the []byte content at a location designated by "path".
   // This should primarily be used for small objects.
   PutContent(ctx context.Context, path string, content []byte) error

   // Reader retrieves an io.ReadCloser for the content stored at "path"
   // with a given byte offset.
   // May be used to resume reading a stream by providing a nonzero offset.
   Reader(ctx context.Context, path string, offset int64) (io.ReadCloser, error)

   // Writer returns a FileWriter which will store the content written to it
   // at the location designated by "path" after the call to Commit.
   Writer(ctx context.Context, path string, append bool) (FileWriter, error)

   // Stat retrieves the FileInfo for the given path, including the current
   // size in bytes and the creation time.
   Stat(ctx context.Context, path string) (FileInfo, error)

   // List returns a list of the objects that are direct descendants of the
   //given path.
   List(ctx context.Context, path string) ([]string, error)

   // Move moves an object stored at sourcePath to destPath, removing the
   // original object.
   // Note: This may be no more efficient than a copy followed by a delete for
   // many implementations.
   Move(ctx context.Context, sourcePath string, destPath string) error

   // Delete recursively deletes all objects stored at "path" and its subpaths.
   Delete(ctx context.Context, path string) error

   // URLFor returns a URL which may be used to retrieve the content stored at
   // the given path, possibly using the given options.
   // May return an ErrUnsupportedMethod in certain StorageDriver
   // implementations.
   URLFor(ctx context.Context, path string, options map[string]interface{}) (string, error)

   // Walk traverses a filesystem defined within driver, starting
   // from the given path, calling f on each file.
   // If the returned error from the WalkFn is ErrSkipDir and fileInfo refers
   // to a directory, the directory will not be entered and Walk
   // will continue the traversal.  If fileInfo refers to a normal file, processing stops
   Walk(ctx context.Context, path string, f WalkFn) error
}

// FileWriter provides an abstraction for an opened writable file-like object in
// the storage backend. The FileWriter must flush all content written to it on
// the call to Close, but is only required to make its content readable on a
// call to Commit.
type FileWriter interface {
   io.WriteCloser

   // Size returns the number of bytes written to this FileWriter.
   Size() int64

   // Cancel removes any written content from this FileWriter.
   Cancel() error

   // Commit flushes all content written to this FileWriter and makes it
   // available for future calls to StorageDriver.GetContent and
   // StorageDriver.Reader.
   Commit() error
}
```
可以参考如下文章进行实现
https://www.ibm.com/developerworks/cn/opensource/os-cn-docker-registry/index.html

### 存储的文件结构
```
.
└── docker
    └── registry
        └── v2
            ├── blobs
            │   └── sha256
            │       ├── 07
            │       │   └── 0766572b4bacfaee9a8eb6bae79e6f6dbcdfac0805c7c6ec8b6c2c0ef097317a
            │       ├── 9a
            │       │   └── 9a597e826a59709a4af34279f496c323d496a79e4c998ee5249a738e391192bb
            │       └── 9c
            │           └── 9ca846b27f6e92f0739af5bba5509357b52be0ce0dd02d216f4dccdacd695a8a
            └── repositories
                ├── alpine
                │   ├── _layers
                │   │   └── sha256
                │   │       ├── 0766572b4bacfaee9a8eb6bae79e6f6dbcdfac0805c7c6ec8b6c2c0ef097317a
                │   │       └── 9ca846b27f6e92f0739af5bba5509357b52be0ce0dd02d216f4dccdacd695a8a
                │   ├── _manifests
                │   │   ├── revisions
                │   │   │   └── sha256
                │   │   │       └── 9a597e826a59709a4af34279f496c323d496a79e4c998ee5249a738e391192bb
                │   │   └── tags
                │   │       └── 3.4
                │   │           ├── current
                │   │           └── index
                │   │               └── sha256
                │   │                   └── 9a597e826a59709a4af34279f496c323d496a79e4c998ee5249a738e391192bb
                │   └── _uploads
                └── library
                    └── alpine
                        ├── _layers
                        │   └── sha256
                        │       ├── 0766572b4bacfaee9a8eb6bae79e6f6dbcdfac0805c7c6ec8b6c2c0ef097317a
                        │       └── 9ca846b27f6e92f0739af5bba5509357b52be0ce0dd02d216f4dccdacd695a8a
                        ├── _manifests
                        │   ├── revisions
                        │   │   └── sha256
                        │   │       └── 9a597e826a59709a4af34279f496c323d496a79e4c998ee5249a738e391192bb
                        │   └── tags
                        │       └── 3.4
                        │           ├── current
                        │           └── index
                        │               └── sha256
                        │                   └── 9a597e826a59709a4af34279f496c323d496a79e4c998ee5249a738e391192bb
                        └── _uploads
```

### 代码流程：
代码前面流程都一样，注册handler之后，HTTP服务起来之后，来一个请求就走对应请求的handler。  
流程示意图：
![](https://z3.ax1x.com/2021/05/04/gmvzOf.jpg)

代码流程：/cmd/registry/main.go:registry.RootCmd.Execute()
-->/registry/root.go:RootCmd.AddCommand(ServeCmd)注册命令-->ServeCmd-->/registry/registry.go:NewRegistry()
-->/registry/registry.go:加载配置文件（默认在/etc/docker/registry/config.yml）-->/registry/handler/app.go:NewApp()中注册handler-->/registry/registry.go:registry.ListenAndServe()监听

注册命令：
```go
func init() {
	RootCmd.AddCommand(ServeCmd)
	RootCmd.AddCommand(GCCmd)
	GCCmd.Flags().BoolVarP(&dryRun, "dry-run", "d", false, "do everything except remove the blobs")
	GCCmd.Flags().BoolVarP(&removeUntagged, "delete-untagged", "m", false, "delete manifests that are not currently referenced via tag")
	RootCmd.Flags().BoolVarP(&showVersion, "version", "v", false, "show the version and exit")
}

// RootCmd is the main command for the 'registry' binary.
var RootCmd = &cobra.Command{
   Use:   "registry",
   Short: "`registry`",
   Long:  "`registry`",
   Run: func(cmd *cobra.Command, args []string) {
      if showVersion {
         version.PrintVersion()
         return
      }
      cmd.Usage()
   },
}
```
ServeCmd中代码片段
```go
        registry, err := NewRegistry(ctx, config)
		if err != nil {
			log.Fatalln(err)
		}

		if config.HTTP.Debug.Prometheus.Enabled {
			path := config.HTTP.Debug.Prometheus.Path
			if path == "" {
				path = "/metrics"
			}
			log.Info("providing prometheus metrics on ", path)
			http.Handle(path, metrics.Handler())
		}

		if err = registry.ListenAndServe(); err != nil {
			log.Fatalln(err)
		}
```

这里用获取blob为例：
```
GET /v2/<name>/blobs/<digest>
```
这个对应的就是调用其handler：blobDispatcher。
注册的地方为
/registry/handlers/app.go。
```go
	app.register(v2.RouteNameManifest, manifestDispatcher)
	app.register(v2.RouteNameCatalog, catalogDispatcher)
	app.register(v2.RouteNameTags, tagsDispatcher)
	app.register(v2.RouteNameBlob, blobDispatcher)
	app.register(v2.RouteNameBlobUpload, blobUploadDispatcher)
	app.register(v2.RouteNameBlobUploadChunk, blobUploadDispatcher)
```

获取blob流程(由于实验使用的s3作为存储后端，所以就看这条流程，其他存储后端看其他具体实现即可)：
/registry/handler/blob.go:blobDispatcher()-->/registry/handler/blob.go:GetBlob()-->/registry/storage/blobserver.go:ServeBlob()-->/registry/storage/driver/s3-aws/s3.go:URLFor()-->S3.GetObjectRequest(&s3.GetObjectInput{
			Bucket: aws.String(d.Bucket),
			Key:    aws.String(d.s3Path(path)),
		})


代码片段：
/registry/handler/blob.go:blobDispatcher()
```go
// blobDispatcher uses the request context to build a blobHandler.
func blobDispatcher(ctx *Context, r *http.Request) http.Handler {
	dgst, err := getDigest(ctx)
	if err != nil {

		if err == errDigestNotAvailable {
			return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
				ctx.Errors = append(ctx.Errors, v2.ErrorCodeDigestInvalid.WithDetail(err))
			})
		}

		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ctx.Errors = append(ctx.Errors, v2.ErrorCodeDigestInvalid.WithDetail(err))
		})
	}

	blobHandler := &blobHandler{
		Context: ctx,
		Digest:  dgst,
	}

	mhandler := handlers.MethodHandler{
		"GET":  http.HandlerFunc(blobHandler.GetBlob),
		"HEAD": http.HandlerFunc(blobHandler.GetBlob),
	}

	if !ctx.readOnly {
		mhandler["DELETE"] = http.HandlerFunc(blobHandler.DeleteBlob)
	}

	return mhandler
}
```

/registry/storage/driver/s3-aws/s3.go:URLFor()
```go
// URLFor returns a URL which may be used to retrieve the content stored at the given path.
// May return an UnsupportedMethodErr in certain StorageDriver implementations.
func (d *driver) URLFor(ctx context.Context, path string, options map[string]interface{}) (string, error) {
	methodString := "GET"
	method, ok := options["method"]
	if ok {
		methodString, ok = method.(string)
		if !ok || (methodString != "GET" && methodString != "HEAD") {
			return "", storagedriver.ErrUnsupportedMethod{}
		}
	}

	expiresIn := 20 * time.Minute
	expires, ok := options["expiry"]
	if ok {
		et, ok := expires.(time.Time)
		if ok {
			expiresIn = time.Until(et)
		}
	}

	var req *request.Request

	switch methodString {
	case "GET":
		req, _ = d.S3.GetObjectRequest(&s3.GetObjectInput{
			Bucket: aws.String(d.Bucket),
			Key:    aws.String(d.s3Path(path)),
		})
	case "HEAD":
		req, _ = d.S3.HeadObjectRequest(&s3.HeadObjectInput{
			Bucket: aws.String(d.Bucket),
			Key:    aws.String(d.s3Path(path)),
		})
	default:
		panic("unreachable")
	}

	return req.Presign(expiresIn)
}
```

## 补充
通过文章[镜像仓库中镜像存储的原理解析](https://supereagle.github.io/2018/04/24/docker-registry/)  发现结论：  
同一镜像，在不同镜像仓库中，存储的方式和内容完全一样。  
文章中通过实验得出如下结论：  
通过 Registry API 获得的两个镜像仓库中相同镜像的 manifest 信息完全相同。
两个镜像仓库中相同镜像的 manifest 信息的存储路径和内容完全相同。
两个镜像仓库中相同镜像的 blob 信息的存储路径和内容完全相同。


## 参考资料：
[docker registry官方github](https://github.com/docker/distribution)
  
[Docker Registry Storage](https://hui.lu/docker-registry-storage/)  

[镜像仓库中镜像存储的原理解析](https://supereagle.github.io/2018/04/24/docker-registry/)  

[Distribution源码分析](https://blog.csdn.net/yuanfang_way/article/category/5904523)  

[Docker镜像的存储机制](https://segmentfault.com/a/1190000014284289)  

[Docker registry 定制实例 – 集成您自己的镜像存储库](https://www.ibm.com/developerworks/cn/opensource/os-cn-docker-registry/index.html)


