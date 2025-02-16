---
title: 编译docker源码并由二进制启动docker
date: 2019-04-24 13:31:30
categories:
- 云计算
tags: [docker, docker源码]
---


![](https://z3.ax1x.com/2021/05/04/gmvdWn.jpg)

<!-- more -->
  
本文主要记录如何由docker源码编译出docker二进制，并由二进制启动docker。
由于项目使用的docker版本为17.03.2-ce，所以本文就以此版本为例。

## 编译docker源码
其实编译docker源码是一个比较复杂过程，但幸运的是docker官方提供了一个Makefile和Dockerfile，将编译docker的复杂的操作都封装起来了，使得我们手动编译的过程简单多了。解决一些源下载慢的问题后其实还挺简单方便的。

准备事项：
* 硬件环境：编译docker的过程是挺耗费内存的，如果你使用虚拟机的话，建议至少分2GB的内存。我这里使用的OS是Centos，硬件是4核4GB的服务器。
* 因为docker是在容器里面去编译的，所以我们需要先安装docker
* 到[https://github.com/moby/moby/releases](https://github.com/moby/moby/releases) 下载对应的17.03.2-ce版本源码

* 构建编译docker的环境。docker的编译是在容器中进行的，构建这个容器的命令很简单，在下载的docker源码目录执行make build即可。我们看Makefile文件可以发现，执行make build的时候其实值执行了如下命令：
```
docker build ${BUILD_APT_MIRROR} ${DOCKER_BUILD_ARGS} -t "$(DOCKER_IMAGE)" -f "$(DOCKERFILE)" .`
```
Dockerfile就与Makefile文件处于同一个文件目录，其内容如下：
```
1 # This file describes the standard way to build Docker, using docker
  2 #
  3 # Usage:
  4 #
  5 # # Assemble the full dev environment. This is slow the first time.
  6 # docker build -t docker .
  7 #
  8 # # Mount your source in an interactive container for quick testing:
  9 # docker run -v `pwd`:/go/src/github.com/docker/docker --privileged -i -t docker bash
 10 #
 11 # # Run the test suite:
 12 # docker run --privileged docker hack/make.sh test-unit test-integration-cli test-docker-py
 13 #
 14 # # Publish a release:
 15 # docker run --privileged \
 16 #  -e AWS_S3_BUCKET=baz \
 17 #  -e AWS_ACCESS_KEY=foo \
 18 #  -e AWS_SECRET_KEY=bar \
 19 #  -e GPG_PASSPHRASE=gloubiboulga \
 20 #  docker hack/release.sh
 21 #
 22 # Note: AppArmor used to mess with privileged mode, but this is no longer
 23 # the case. Therefore, you don't have to disable it anymore.
 24 #
 25
 26 FROM debian:jessie
 27
 28 # allow replacing httpredir or deb mirror
 29 ARG APT_MIRROR=deb.debian.org
 30 RUN sed -ri "s/(httpredir|deb).debian.org/$APT_MIRROR/g" /etc/apt/sources.list
 31
 32 # Add zfs ppa
 33 COPY keys/launchpad-ppa-zfs.asc /go/src/github.com/docker/docker/keys/
 34 RUN apt-key add /go/src/github.com/docker/docker/keys/launchpad-ppa-zfs.asc
 35 RUN echo deb http://ppa.launchpad.net/zfs-native/stable/ubuntu trusty main > /etc/apt/sources.list.d/zfs.list
 36
 37 # Packaged dependencies
 38 RUN apt-get update && apt-get install -y \
 39         apparmor \
 40         apt-utils \
 41         aufs-tools \
 42         automake \
 43         bash-completion \
 44         binutils-mingw-w64 \
 45         bsdmainutils \
 46         btrfs-tools \
 47         build-essential \
 48         clang \
 49         cmake \
 50         createrepo \
 51         curl \
 52         dpkg-sig \
 53         gcc-mingw-w64 \
 54         git \
 55         iptables \
 56         jq \
 57         libapparmor-dev \
 58         libcap-dev \
 59         libltdl-dev \
 60         libnl-3-dev \
 61         libprotobuf-c0-dev \
 62         libprotobuf-dev \
 63         libsqlite3-dev \
 64         libsystemd-journal-dev \
 65         libtool \
 66         mercurial \
 67         net-tools \
 68         pkg-config \
 69         protobuf-compiler \
 70         protobuf-c-compiler \
 71         python-dev \
 72         python-mock \
 73         python-pip \
 74         python-websocket \
 75         ubuntu-zfs \
 76         xfsprogs \
 77         vim-common \
 78         libzfs-dev \
 79         tar \
 80         zip \
 81         --no-install-recommends \
 82         && pip install awscli==1.10.15
 83 # Get lvm2 source for compiling statically
 84 ENV LVM2_VERSION 2.02.103
 85 RUN mkdir -p /usr/local/lvm2 \
 86         && curl -fsSL "https://mirrors.kernel.org/sourceware/lvm2/LVM2.${LVM2_VERSION}.tgz" \
 87                 | tar -xzC /usr/local/lvm2 --strip-components=1
 88 # See https://git.fedorahosted.org/cgit/lvm2.git/refs/tags for release tags
 89
 90 # Compile and install lvm2
 91 RUN cd /usr/local/lvm2 \
 92         && ./configure \
 93                 --build="$(gcc -print-multiarch)" \
 94                 --enable-static_link \
 95         && make device-mapper \
 96         && make install_device-mapper
 97 # See https://git.fedorahosted.org/cgit/lvm2.git/tree/INSTALL
 98
 99 # Configure the container for OSX cross compilation
100 ENV OSX_SDK MacOSX10.11.sdk
101 ENV OSX_CROSS_COMMIT a9317c18a3a457ca0a657f08cc4d0d43c6cf8953
102 RUN set -x \
103         && export OSXCROSS_PATH="/osxcross" \
104         && git clone https://github.com/tpoechtrager/osxcross.git $OSXCROSS_PATH \
105         && ( cd $OSXCROSS_PATH && git checkout -q $OSX_CROSS_COMMIT) \
106         && curl -sSL https://s3.dockerproject.org/darwin/v2/${OSX_SDK}.tar.xz -o "${OSXCROSS_PATH}/tarballs/${OSX_SDK}.tar.xz" \
107         && UNATTENDED=yes OSX_VERSION_MIN=10.6 ${OSXCROSS_PATH}/build.sh
108 ENV PATH /osxcross/target/bin:$PATH
109
110 # Install seccomp: the version shipped in trusty is too old
111 ENV SECCOMP_VERSION 2.3.1
112 RUN set -x \
113         && export SECCOMP_PATH="$(mktemp -d)" \
114         && curl -fsSL "https://github.com/seccomp/libseccomp/releases/download/v${SECCOMP_VERSION}/libseccomp-${SECCOMP_VERSION}.tar.gz" \
115                 | tar -xzC "$SECCOMP_PATH" --strip-components=1 \
116         && ( \
117                 cd "$SECCOMP_PATH" \
118                 && ./configure --prefix=/usr/local \
119                 && make \
120                 && make install \
121                 && ldconfig \
122         ) \
123         && rm -rf "$SECCOMP_PATH"
124
125 # Install Go
126 # IMPORTANT: If the version of Go is updated, the Windows to Linux CI machines
127 #            will need updating, to avoid errors. Ping #docker-maintainers on IRC
128 #            with a heads-up.
129 ENV GO_VERSION 1.7.5
130 RUN curl -fsSL "https://golang.org/dl/go${GO_VERSION}.linux-amd64.tar.gz" \
131         | tar -xzC /usr/local
132
133 ENV PATH /go/bin:/usr/local/go/bin:$PATH
134 ENV GOPATH /go
135
136 # Compile Go for cross compilation
137 ENV DOCKER_CROSSPLATFORMS \
138         linux/386 linux/arm \
139         darwin/amd64 \
140         freebsd/amd64 freebsd/386 freebsd/arm \
141         windows/amd64 windows/386 \
142         solaris/amd64
143
144 # Dependency for golint
145 ENV GO_TOOLS_COMMIT 823804e1ae08dbb14eb807afc7db9993bc9e3cc3
146 RUN git clone https://github.com/golang/tools.git /go/src/golang.org/x/tools \
147         && (cd /go/src/golang.org/x/tools && git checkout -q $GO_TOOLS_COMMIT)
148
149 # Grab Go's lint tool
150 ENV GO_LINT_COMMIT 32a87160691b3c96046c0c678fe57c5bef761456
151 RUN git clone https://github.com/golang/lint.git /go/src/github.com/golang/lint \
152         && (cd /go/src/github.com/golang/lint && git checkout -q $GO_LINT_COMMIT) \
153         && go install -v github.com/golang/lint/golint
154
155 # Install CRIU for checkpoint/restore support
156 ENV CRIU_VERSION 2.2
157 RUN mkdir -p /usr/src/criu \
158         && curl -sSL https://github.com/xemul/criu/archive/v${CRIU_VERSION}.tar.gz | tar -v -C /usr/src/criu/ -xz --strip-components=1 \
159         && cd /usr/src/criu \
160         && make \
161         && make install-criu
162
163 # Install two versions of the registry. The first is an older version that
164 # only supports schema1 manifests. The second is a newer version that supports
165 # both. This allows integration-cli tests to cover push/pull with both schema1
166 # and schema2 manifests.
167 ENV REGISTRY_COMMIT_SCHEMA1 ec87e9b6971d831f0eff752ddb54fb64693e51cd
168 ENV REGISTRY_COMMIT 47a064d4195a9b56133891bbb13620c3ac83a827
169 RUN set -x \
170         && export GOPATH="$(mktemp -d)" \
171         && git clone https://github.com/docker/distribution.git "$GOPATH/src/github.com/docker/distribution" \
172         && (cd "$GOPATH/src/github.com/docker/distribution" && git checkout -q "$REGISTRY_COMMIT") \
173         && GOPATH="$GOPATH/src/github.com/docker/distribution/Godeps/_workspace:$GOPATH" \
174                 go build -o /usr/local/bin/registry-v2 github.com/docker/distribution/cmd/registry \
175         && (cd "$GOPATH/src/github.com/docker/distribution" && git checkout -q "$REGISTRY_COMMIT_SCHEMA1") \
176         && GOPATH="$GOPATH/src/github.com/docker/distribution/Godeps/_workspace:$GOPATH" \
177                 go build -o /usr/local/bin/registry-v2-schema1 github.com/docker/distribution/cmd/registry \
178         && rm -rf "$GOPATH"
179
180 # Install notary and notary-server
181 ENV NOTARY_VERSION v0.4.2
182 RUN set -x \
183         && export GOPATH="$(mktemp -d)" \
184         && git clone https://github.com/docker/notary.git "$GOPATH/src/github.com/docker/notary" \
185         && (cd "$GOPATH/src/github.com/docker/notary" && git checkout -q "$NOTARY_VERSION") \
186         && GOPATH="$GOPATH/src/github.com/docker/notary/vendor:$GOPATH" \
187                 go build -o /usr/local/bin/notary-server github.com/docker/notary/cmd/notary-server \
188         && GOPATH="$GOPATH/src/github.com/docker/notary/vendor:$GOPATH" \
189                 go build -o /usr/local/bin/notary github.com/docker/notary/cmd/notary \
190         && rm -rf "$GOPATH"
191
192 # Get the "docker-py" source so we can run their integration tests
193 ENV DOCKER_PY_COMMIT e2655f658408f9ad1f62abdef3eb6ed43c0cf324
194 RUN git clone https://github.com/docker/docker-py.git /docker-py \
195         && cd /docker-py \
196         && git checkout -q $DOCKER_PY_COMMIT \
197         && pip install -r test-requirements.txt
198
199 # Install yamllint for validating swagger.yaml
200 RUN pip install yamllint==1.5.0
201
202 # Install go-swagger for validating swagger.yaml
203 ENV GO_SWAGGER_COMMIT c28258affb0b6251755d92489ef685af8d4ff3eb
204 RUN git clone https://github.com/go-swagger/go-swagger.git /go/src/github.com/go-swagger/go-swagger \
205         && (cd /go/src/github.com/go-swagger/go-swagger && git checkout -q $GO_SWAGGER_COMMIT) \
206         && go install -v github.com/go-swagger/go-swagger/cmd/swagger
207
208 # Set user.email so crosbymichael's in-container merge commits go smoothly
209 RUN git config --global user.email 'docker-dummy@example.com'
210
211 # Add an unprivileged user to be used for tests which need it
212 RUN groupadd -r docker
213 RUN useradd --create-home --gid docker unprivilegeduser
214
215 VOLUME /var/lib/docker
216 WORKDIR /go/src/github.com/docker/docker
217 ENV DOCKER_BUILDTAGS apparmor pkcs11 seccomp selinux
218
219 # Let us use a .bashrc file
220 RUN ln -sfv $PWD/.bashrc ~/.bashrc
221 # Add integration helps to bashrc
222 RUN echo "source $PWD/hack/make/.integration-test-helpers" >> /etc/bash.bashrc
223
224 # Register Docker's bash completion.
225 RUN ln -sv $PWD/contrib/completion/bash/docker /etc/bash_completion.d/docker
226
227 # Get useful and necessary Hub images so we can "docker load" locally instead of pulling
228 COPY contrib/download-frozen-image-v2.sh /go/src/github.com/docker/docker/contrib/
229 RUN ./contrib/download-frozen-image-v2.sh /docker-frozen-images \
230         buildpack-deps:jessie@sha256:25785f89240fbcdd8a74bdaf30dd5599a9523882c6dfc567f2e9ef7cf6f79db6 \
231         busybox:latest@sha256:e4f93f6ed15a0cdd342f5aae387886fba0ab98af0a102da6276eaf24d6e6ade0 \
232         debian:jessie@sha256:f968f10b4b523737e253a97eac59b0d1420b5c19b69928d35801a6373ffe330e \
233         hello-world:latest@sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7
234 # See also "hack/make/.ensure-frozen-images" (which needs to be updated any time this list is)
235
236 # Install tomlv, vndr, runc, containerd, tini, docker-proxy
237 # Please edit hack/dockerfile/install-binaries.sh to update them.
238 COPY hack/dockerfile/binaries-commits /tmp/binaries-commits
239 COPY hack/dockerfile/install-binaries.sh /tmp/install-binaries.sh
240 RUN /tmp/install-binaries.sh tomlv vndr runc containerd tini proxy bindata
241
242 # Wrap all commands in the "docker-in-docker" script to allow nested containers
243 ENTRYPOINT ["hack/dind"]
244
245 # Upload docker source
246 COPY . /go/src/github.com/docker/docker
```

我们会遇到的问题是下载问题，所以好几个地方需要换成国内的源：

第一、替换Debian源为中国的源（大约第29行）：
```
# 修改前
ARG APT_MIRROR=deb.debian.org

# 修改后
ARG APT_MIRROR=ftp.cn.debian.org
```

第二、修改pip源为国内的源（大约第82行）
```
# 修改前
&& pip install awscli==1.10.15

# 修改后
&& pip install awscli==1.10.15 -i http://mirrors.aliyun.com/pypi/simple
```

第三、修改下载go安装包的地址（大约第129行）。这里需要注意GO_VERSION的版本，不同版本的地址可以去[http://golangtc.com/download](http://golangtc.com/download)获取。
```
# 修改前
ENV GO_VERSION 1.7.5
RUN curl -fsSL "https://golang.org/dl/go${GO_VERSION}.linux-amd64.tar.gz" \
| tar -xzC /usr/local

# 修改后
ENV GO_VERSION 1.7.5
RUN curl -fsSL "http://golangtc.com/static/go/1.7.5/go1.7.5.linux-amd64.tar.gz" \
| tar -xzC /usr/local
```

做如上修改之后就可以使用make build命令了，如果不出什么错的话，编译完之后就会有一个叫docker-dev的容器。当然上述命令可能由于github下载不稳定导致比较慢，如果受不了这种情况，另一个比较快的办法是，购买一台海外的云服务器，然后也不用做上述修改，直接make build，很简单快捷: )

* 编译docker：就在刚才目录执行make binary命令即可：


编译出的结果如下
```
[root@vultr moby-17.03.2-ce]# ll bundles/latest/binary-client/
total 13808
lrwxrwxrwx 1 root root       17 Apr 23 16:58 docker -> docker-17.03.2-ce
-rwxr-xr-x 1 root root 14128576 Apr 23 16:58 docker-17.03.2-ce
-rw-r--r-- 1 root root       52 Apr 23 16:58 docker-17.03.2-ce.md5
-rw-r--r-- 1 root root       84 Apr 23 16:58 docker-17.03.2-ce.sha256
[root@vultr moby-17.03.2-ce]# ll bundles/latest/binary-daemon/
total 69160
-rwxr-xr-x 1 root root  8932648 Apr 23 16:59 docker-containerd
-rwxr-xr-x 1 root root  8381448 Apr 23 16:59 docker-containerd-ctr
-rw-r--r-- 1 root root       56 Apr 23 16:59 docker-containerd-ctr.md5
-rw-r--r-- 1 root root       88 Apr 23 16:59 docker-containerd-ctr.sha256
-rw-r--r-- 1 root root       52 Apr 23 16:59 docker-containerd.md5
-rw-r--r-- 1 root root       84 Apr 23 16:59 docker-containerd.sha256
-rwxr-xr-x 1 root root  3047368 Apr 23 16:59 docker-containerd-shim
-rw-r--r-- 1 root root       57 Apr 23 16:59 docker-containerd-shim.md5
-rw-r--r-- 1 root root       89 Apr 23 16:59 docker-containerd-shim.sha256
lrwxrwxrwx 1 root root       18 Apr 23 16:59 dockerd -> dockerd-17.03.2-ce
-rwxr-xr-x 1 root root 39989264 Apr 23 16:59 dockerd-17.03.2-ce
-rw-r--r-- 1 root root       53 Apr 23 16:59 dockerd-17.03.2-ce.md5
-rw-r--r-- 1 root root       85 Apr 23 16:59 dockerd-17.03.2-ce.sha256
-rwxr-xr-x 1 root root   772408 Apr 23 16:59 docker-init
-rw-r--r-- 1 root root       46 Apr 23 16:59 docker-init.md5
-rw-r--r-- 1 root root       78 Apr 23 16:59 docker-init.sha256
-rwxr-xr-x 1 root root  2534781 Apr 23 16:59 docker-proxy
-rw-r--r-- 1 root root       47 Apr 23 16:59 docker-proxy.md5
-rw-r--r-- 1 root root       79 Apr 23 16:59 docker-proxy.sha256
-rwxr-xr-x 1 root root  7092608 Apr 23 16:59 docker-runc
-rw-r--r-- 1 root root       46 Apr 23 16:59 docker-runc.md5
-rw-r--r-- 1 root root       78 Apr 23 16:59 docker-runc.sha256
```
![](https://z3.ax1x.com/2021/05/04/gmvBQ0.jpg)



## 由二进制安装启动docker
编译出二进制文件之后就可以安装启动docker了，其主要步骤就是将编译出来的二进制拷贝到指定目录，然后启动服务。  
安装启动步骤：
*  编译出docker二进制或者下载docker发布的linux的二进制包
*  生成docker.service的文件并设定到/usr/lib/systemd/system目录下
*  拷贝docker的二进制文件docker*到/usr/bin或者执行路径可以找到的目录
*  systemctl restart docker
*  systemctl enable docker

为了方便，将该过程脚本化，脚本名为install_docker.sh：
```
#!/bin/sh
############################################################
# This script can install docker from docker binary files. #
# So first need the dir contains the docker bin files.     #
# You can compile the source code for binary files         #
# or get docker-ce binary from:                            #
# https://download.docker.com/linux/static/stable/x86_64/  #
############################################################


SYSTEMDDIR=/usr/lib/systemd/system
SERVICEFILE=docker.service
DOCKERDIR=/usr/bin
SERVICENAME=docker

precheck(){
  echo "Check the env before install docker..."
  if [ -f ${DOCKERDIR}/${SERVICENAME} ]; then
     echo "already had docker, remove current docker..."
     echo "Stop current ${SERVICENAME} service..."
     systemctl stop ${SERVICENAME}
     echo "Remove current ${SERVICENAME} bin files"
     rm -f ${DOCKERDIR}/${SERVICENAME}*
  fi
  echo "Finish precheck."
}


usage(){
  echo "Usage: $0 DOCKER_BIN_FILE_DIR"
  echo "       $0 ./docker/"
  echo "Get docker-ce binary from: https://download.docker.com/linux/static/stable/x86_64/ or compile source code yourself"
  echo "eg: wget https://download.docker.com/linux/static/stable/x86_64/docker-17.09.0-ce.tgz"
  echo ""
}


if [ $# -ne 1 ]; then
  usage
  exit 1
else
  DOCKERBIN="$1"
fi

precheck
if [ ! -d ${DOCKERBIN} ]; then
  echo "Docker binary dir does not exist, please check it"
  echo "Get docker-ce binary from: https://download.docker.com/linux/static/stable/x86_64/ or you can compile the docker source code for binary files"
  echo "eg: wget https://download.docker.com/linux/static/stable/x86_64/docker-17.09.0-ce.tgz"
  exit 1
fi

echo "##binary : ${DOCKERBIN} copy to ${DOCKERDIR}"
cp -p ${DOCKERBIN}/* ${DOCKERDIR} >/dev/null 2>&1
which ${SERVICENAME}

echo "##systemd service: ${SERVICEFILE}"
echo "##docker.service: create docker systemd file"
cat >${SYSTEMDDIR}/${SERVICEFILE} <<EOF
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target docker.socket
[Service]
Type=notify
EnvironmentFile=-/run/flannel/docker
WorkingDirectory=/usr/local/bin
ExecStart=/usr/bin/dockerd \
                -H tcp://0.0.0.0:4243 \
                -H unix:///var/run/docker.sock \
                --selinux-enabled=false \
                --log-opt max-size=1g
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

echo ""

systemctl daemon-reload
echo "##Service status: ${SERVICENAME}"
systemctl status ${SERVICENAME}
echo "##Service restart: ${SERVICENAME}"
systemctl restart ${SERVICENAME}
echo "##Service status: ${SERVICENAME}"
systemctl status ${SERVICENAME}

echo "##Service enabled: ${SERVICENAME}"
systemctl enable ${SERVICENAME}

echo "## docker version"
docker version

```

脚本使用说明：  
使用脚本的时候需要将所有的docker二进制放到一个目录中，比如就放到当前的docker目录中，那么使用方法便是
```
sh install_docker.sh ./docker
```
![](https://z3.ax1x.com/2021/05/04/gmvyeU.jpg)

安装启动完毕查看docker是否正常
```
docker version
Client:
 Version:      17.03.2-ce
 API version:  1.27
 Go version:   go1.7.5
 Git commit:   1
 Built:        Tue Apr 23 08:57:53 2019
 OS/Arch:      linux/amd64

Server:
 Version:      17.03.2-ce
 API version:  1.27 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   1
 Built:        Tue Apr 23 08:57:53 2019
 OS/Arch:      linux/amd64
 Experimental: false
```
或者使用
```
systemctl status docker
```
查看

## 遇到的问题
### 没有指定DOCKER_GITCOMMIT的问题

>.git directory missing and DOCKER_GITCOMMIT not specified
  Please either build with the .git directory accessible, or specify the
  exact (--short) commit hash you are building using DOCKER_GITCOMMIT for
  future accountability in diagnosing build issues.  Thanks!
  
解决办法：  
     手动指定即可，例如
```
 make DOCKER_GITCOMMIT=bb80604 binary
```

### Failed to start Docker Application Container Engine或者initializing graphdriver: driver not supported

![](https://z3.ax1x.com/2021/05/04/gmv2FJ.jpg)
```
Apr 24 10:19:15 vultr.guest systemd[1]: docker.service: main process exited, code=exited, status=203/EXEC
Apr 24 10:19:15 vultr.guest systemd[1]: Failed to start Docker Application Container Engine.
Apr 24 10:19:15 vultr.guest systemd[1]: Unit docker.service entered failed state.
Apr 24 10:19:15 vultr.guest systemd[1]: docker.service failed.
Apr 24 10:19:15 vultr.guest systemd[1]: docker.service holdoff time over, scheduling restart.
Apr 24 10:19:15 vultr.guest systemd[1]: Stopped Docker Application Container Engine.
Apr 24 10:19:15 vultr.guest systemd[1]: start request repeated too quickly for docker.service
Apr 24 10:19:15 vultr.guest systemd[1]: Failed to start Docker Application Container Engine.
Apr 24 10:19:15 vultr.guest systemd[1]: Unit docker.service entered failed state.
Apr 24 10:19:15 vultr.guest systemd[1]: docker.service failed.

```

错误原因：error initializing graphdriver: driver not supported
解决办法：在 /etc/docker 目录的daemon.json文件中加入以下配置
vi /etc/docker/daemon.json
```
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```
再次启动
```
systemctl start docker
```

另外一个解决办法是：  
sudo mv /var/lib/docker /var/lib/docker.old，然后重启

这个方法有个弊端，因为docker的所有文件都是在这个文件夹里面的，所以**所有的container和image**都会没了


## 参考资料：
[https://github.com/moby/moby/releases](https://github.com/moby/moby/releases) 

[从源码编译docker](https://niyanchun.com/compile-docker-from-source.html)  

[docker之路：docker 开发环境配置](https://delveshal.github.io/2018/06/04/docker%E4%B9%8B%E8%B7%AF%EF%BC%9Adocker-%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE/)

[Docker17.03-CE插件开发案例](https://jimmysong.io/posts/docker-plugin-develop/)

[二进制方式安装docker](https://blog.csdn.net/liumiaocn/article/details/71157586)

[Docker无法启动 error initializing graphdriver: driver not supported](https://www.imooc.com/article/70557)

[docker安装脚本](https://github.com/liumiaocn/easypack/blob/master/docker/install-docker.sh)


[Releases unbuildable because of unset `DOCKER_GITCOMMIT`](https://github.com/moby/moby/issues/27581)

[Docker源码编译和开发环境搭建](https://jimmysong.io/posts/docker-dev-env/)

[升级内核后无法启动docker](https://urzz.me/2017/12/17/failed-start-docker-after-upgrade-kernel/)