# Introduction

Node.js depends on the OpenSSL library to implement its crypto, TLS/SSL and HTTPS features.

However, Node.js doesn't use OpenSSL's standard build process. Instead, a custom build process based on GYP that uses OpenSSL's source is maintained in parallel of OpenSSL's original build process. In addition to that, because the state of OpenSSL's support for Node.js supported platforms varies over time, it is sometimes necessary to maintain additional patches that fix some important issues.

For these two reasons, upgrading the OpenSSL version used by Node.js is not trivial. Some knowledge of how OpenSSL is built and embedded within the node binary and of the existing floating patches is necessary. This document describes both of these aspects so that upgrading the OpenSSL version used by a given version of Node.js does not have to be done by the few people with such knowledge.

The information in this document is valid for Node.js versions v0.10.x and v0.12.x.

## Floating patches

Following is the list of the current patches that are floated on top of OpenSSL's source:
* [2b21c45f75043f8e5728650e24a4e972ade18cf1](https://github.com/joyent/node/commit/2b21c45f75043f8e5728650e24a4e972ade18cf1). `deps: separate sha256/sha512-x86_64.pl for openssl`. This change is needed because Node.js' GYP-based build system cannot use the same PERL script to generate different implementations of the sha algorithm by passing different command line parameters like the original OpenSSL build system does.
* [7817fbd692120887619d07228882dd19461109b6](https://github.com/joyent/node/commit/7817fbd692120887619d07228882dd19461109b6). `deps: fix openssl assembly error on ia32 win32`. 
* [c4b9be7c5a97b9cac99cd599dbd995da556a5a17](https://github.com/joyent/node/commit/c4b9be7c5a97b9cac99cd599dbd995da556a5a17). `openssl: replace symlinks by #include shims`. As stated by the commit message, Git for Windows cannot create symlinks, so symlink sources are replaced by headers that #include the symlinks' targets instead.
* [6b97c2e98627b5189e01b871f9130b5efc41988d](https://github.com/joyent/node/commit/6b97c2e98627b5189e01b871f9130b5efc41988d). `openssl: fix keypress requirement in apps on win32`. Node.js uses openssl's `s_client` program to run some of its tests. However, on Windows this program requires the user to press a key to continue, which makes Node.js' tests time out. This change makes it continue without needing any user interaction.
* [fa4aa347ebbe325e934bcda6f93a15e2e6524f1b] (https://github.com/joyent/node/commit/fa4aa347ebbe325e934bcda6f93a15e2e6524f1b). `deps: add -no_rand_screen to openssl s_client`. In openssl s_client on Windows, RAND_screen() is invoked to initialize
random state but it takes several seconds in each connection.
This added -no_rand_screen to openssl s_client on Windows to skip
RAND_screen() and gets a better performance in the unit test of
test-tls-server-verify.

## How Node.js builds its OpenSSL dependency

The OpenSSL dependency is present at `deps/openssl`. It does not only contain the vanilla OpenSSL source tree, but is instead a "wrapper" that uses a custom build system (`openssl.gyp` and `buildinf.h`), pre-built assembly files (in `asm`) and a centralized configuration header file (in `config`).

Following are more detailed informations about some of the important files and directories in `deps/openssl`:
* `deps/openssl/openssl/`: This directory is the vanilla OpenSSL directory. When upgrading, one overwrites this directory with the content of an OpenSSL release tarball. All the other files in `deps/openssl` that are not in `deps/openssl/openssl` are custom and modified in some way by the Node.js project.
* `deps/openssl/asm/`: This directory contains the assembly files generated by the custom `Makefile` in the same directory. This `Makefile` is basically a port of every `Makefile` found in every subdirectory of `deps/openssl/openssl` (the vanilla OpenSSL source tree) that contains a cipher implementation (most if not all of them in `deps/openssl/openssl/crypto`). The assembly files generated are created in subdirectories for each supported architecture.
* `deps/openssl/buildinf.h` and `deps/openssl/config/`: `deps/openssl/config/opensslconf.h` was added so that binary add-on developers could rely on one single header configuration file instead of having to include different configuration files depending on the current platform.
* `deps/openssl/openssl.gyp`: This file is [included by Node.js' master build file](https://github.com/joyent/node/blob/v0.12.2-release/node.gyp#L203) and more or less replicates the behavior of OpenSSL's original build system.

## Performing the upgrade

Performing an openssl upgrade is a simple process once the existing floating patches are identified:

1. Get the source code of the new OpenSSL release to which Node.js needs to be updated. It can be downloaded on [the OpenSSL downloads page](http://openssl.org/source/). Please note that the OpenSSL projects doesn't use a very common versioning scheme, and it can be confusing. For instance, the 1.0.1* and 1.0.2* versions are very different, and although the differences between the two version numbers might seem minor, an upgrade from the former to the latter would be a very significant upgrade. Node v0.10.x and v0.12.x both use OpenSSL 1.0.1*, so make sure to pick the latest 1.0.1 version when upgrading OpenSSL in these branches.
2. At this point, it is possible to either:
  1. Completely replace the content of `deps/openssl/openssl` with the content from the OpenSSL source tarball.
  2. Compute the diff between the current OpenSSL version used by Node and the one to which we want to upgrade and apply the resulting patch.
The former solution requires more work if changes between two releases are small, but it doesn't have the issue of the latter with changes that overlap with existing floating patches. If in doubt, choose the former approach.
3. Apply the floating patches listed in the ["Floating patches" section](#floating-patches) by using `git cherry-pick`.
4. Run `make` in `deps/openssl/asm`.
5. Test the upgrade by following the instructions in [the "Testing an OpenSSL upgrade" section](#testing-an-openssl-upgrade).
6. Update version-specific links in the API docs. See [#9338](https://github.com/nodejs/node/pull/9338) for an example.

## Testing an OpenSSL upgrade

After completing the upgrade of the OpenSSL dependency in the source tree, the next step is generally to test it.

First, run the standard tests suite on all platforms by using Jenkins. In order to do that, push a topic branch (named e.g `openssl-upgrade-1-0-1z` if this upgrade upgrades OpenSSL to version 1.0.1z) containing the result of the upgrade to joyent/node. Then, if the upgrade is in the v0.12 branch, create a pull request and run the [node-accept-pull-request Jenkins job](http://jenkins.nodejs.org/job/node-accept-pull-request/) (make sure to _not_ check the "Apply changes" checkbox). Run [the `node-review` Jenkins job](http://jenkins.nodejs.org/job/node-review/) if the upgrade needs to be merged in the v0.10 branch.

Another tests suite that is very important to run is the one that can be found at `test/external/ssl-options`. It tests that the various combinations of openssl-related command line options (`--enable-ssl2`, `--enable-ssl3`) and
tls.connect, tls.createServer (and the `https` equivalent) `cipher`, `secureOptions` and `secureProtocol` options work as expected.

To run it, simply change the current directory to `test/external/ssl-options` and install all the npm dependencies:
```
$ cd test/external/ssl-options
$ npm install
```
Now, simply run the `test.js` program using the node binary that needs to be tested:
```
$ ../../../node test.js -j 60
```
The `-j X` option is used to run `X` tests in parallel. Due to the large number of tests that need to be run, this tests suite can take a long time to complete. On my 2.3GHz i7 MacBook Pro with 16GB of ram, it takes around 5 minutes to complete with `-j 60`.

Other command line options are available: `[-j N] [--list-tests] [-s startIndex] [-e endIndex] [-o outputFile]`. 
It is also possible to set the `DEBUG` environment variable to `test-node-ssl` to display debug output.