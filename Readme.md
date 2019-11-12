# 1.13 = Enterprise Go
## Workshop

By Gary Miller
(gm@wx.io)

This documentation covers the technical content from the workshop.
Specifically Go Modules.

## Prerequisites

An installed Go 1.13 environment.

## Exercise 1. Go Mod Replace


``` bash
# Get and build a version of the wx /works/ cli tool
go get -x github.com/wxio/wx@v0.0.6
# Create and populate a .wx.yml config file
cat > .wx.yaml < EOF
workspaces:
- path: repos/wx
  url: https://github.com/wxio/wx.git
- path: repos/dna
  url: https://github.com/wxio/dna.git
- path: repos/lcmd
  url: https://github.com/wxio/lcmd.git
- path: repos/gconau19
  url: https://github.com/wxio/gconau19.git
- path: repos/okmod
  url: https://github.com/wxio/okmod.git
- path: repos/goodmod
  url: https://github.com/wxio/goodmod.git
- path: repos/go-git-update
  url: https://github.com/wxio/go-git-update.git
EOF
# Use wx to git clone all the repos specified in the config file
wx git clone
#
cd repos/gconau19/
#
go build
#
./gconau19

```

Now lets break it.

``` bash
go mod edit -dropreplace=github.com/wxio/badmod
go build
# output
# go: github.com/wxio/okmod@v1.0.0 requires
#        github.com/wxio/badmod@v0.0.0-00010101000000-000000000000: invalid version: unknown revision 000000000000
```

Message: 

1. Go only looks at the replaces in the go.mod of the module being built.
1. It is necessary to copy (and in this can modify) `replace` and `exclude` entries from the third-party go.mod files.

## Exercise 2. Private Repo

### Simple

Use-case:

1. Private code in SaaS bitbucket.org/myuserA & bitbucket.org/myuserB
1. Bitbucket auth via ssh

Environment Setup
``` bash
## Tell go tooling simply which repo urls are private
go env -w GOPRIVATE=bitbucket.org
## or via environment variable
# export GOPRIVATE=bitbucket.org
## or in the case of multiple differ repos
# export GOPRIVATE=bitbucket.org/myuserA,bitbucket.org/myuserB
#
## Tell git to use ssh instead of https
git config --global url."git@bitbucket.org:".insteadOf "https://bitbucket.org/"
#
go build
```

### Complex

Use-case:

1. Private code in on premise git repo
1. Access git repo via the git protocol (This is insecure with no auth; for testing only)

Environment Setup
``` bash
go env -w GOPRIVATE=private.org
git config --global url."git://private.org/".insteadOf "https://private.org/"
```

Note: Import paths need to have `.git` in the path.

Here the road forks depending on the path in the go.mod being imported.

### Path 1. - Same library and import path 

``` go
package gconau19

import (
    "private.org/gcon.git"
)
 ...
```
Go will resolve the `require` automatically.

### Path 2. - Different library and import path 

``` go
package gconau19

import (
    "github.com/wxio/okmod"
)
 ...
```

Here the `go.mod` needs a `replace` statement.

go.mod
``` go
module github.com/wxio/gconau19
go 1.13
replace github.com/wxio/badmod => private.org/gcon.git latest
require github.com/wxio/okmod v1.0.0

```

To setup **Path 2** 

``` bash
# Start a local git daemon
cd repos/go-git-update
./git-serve.sh
```

Add private.org to `/etc/hosts`

In a new shell
``` bash
# add code private repo
cd repos/goodmod
git remote add private git://private.org/gcon.org
git push
# get gconau19 to use private
cd ../gconau19
go env -w GOPRIVATE=private.org
git config --global url."git://private.org/".insteadOf "https://private.org/"
go mod edit -replace=github.com/wxio/badmod=private.org/gcon.git@latest
go build
```

## Development Environment in a Box

The setup for the docker based environment can be found if the `diabox` directory in the https://github.com/wxio/lcmd repository.

### gRPC Mono repo

The protobuf mono including the `lcmd` service definition as at https://github.com/wxio/dna/tree/master/store/wxio .

To modify the lcmd service edit the `lcmd.proto` and from the `lcmd` directory run 

```
godna -d --logtostderr generate ../
```

(compile the godna binary if needed).

### lcmd server and grpcui


Compile and run the `lcmd` service

```
./lcmd grpc_server
```

The `grpcui` used to access the `lcmd` service can be found as https://github.com/fullstorydev/grpcui in the `cmd/grpcui` folder.

```
git clone https://github.com/fullstorydev/grpcui.git
cd grpcui/cmd/grpcui
go build
./grpcui -plaintext -bind 0 -port 50052 localhost:50051
```

Access grpcui at
http://localhost:50052/
