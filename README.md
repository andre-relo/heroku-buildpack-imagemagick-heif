heroku-buildpack-imagemagick-webp-heif
=================================

Use [ImageMagick](www.imagemagick.org) built with [libwebp](https://code.google.com/p/webp/) inside a Heroku `heroku-16` environment. This buildpack will download both libraries, build them from source and install them along side your app.

The rise in popularity and use of HEIF/HEIC(High Efficency Image Format) means your project's image processing also needs to be able to handle this format. The current default version of imagemagick installed on heroku:16 dynos is 6.8.9.9 and does not support processing heic image files. This [Heroku buildpack](http://devcenter.heroku.com/articles/buildpacks) vendors a version of ImageMagick with HEIF support binaries into your project. It is based on several resources including https://github.com/retailzipline/heroku-buildpack-imagemagick-heif.

The orginal buildpack was created for `heroku-18` stacks but this one was modified to work with [Heroku stack:](https://devcenter.heroku.com/articles/stack) `heroku-16`. 

The tar file in the [/build folder](./build) currently contains: 

Version: ImageMagick 7.0.10-19 https://imagemagick.org

You will need to build a new binary if you want to use a newer or different version. To build a new binary see [How to Build a New Binary](#how-to-build-a-new-binary)

## Usage

**NOTE:** _To ensure the newer version of imagemagick is found in the $PATH and installed first make sure this buildpack is added to the top of the buildpack list or at "index 1"._


From your projects "Settings" tab add this buildpack to your app in the 1st position:

```
https://github.com/HiMamaInc/heroku-buildpack-imagemagick-heif
```

**OR**

From the command line:

```
heroku buildpacks:add https://github.com/HiMamaInc/heroku-buildpack-imagemagick-heif --index 1 --app HEROKU_APP_NAME
```

## How to Build a New Binary

The binary in this repo was built in a heroku:16 docker image running in a local dev environment. The tar file was then copied into the `/build` directory in this repo and is used by the [compile script](./bin/compile).

Prerequisites

- Docker installed and running in local dev environment. [Get Docker](https://docs.docker.com/get-docker/)

Steps:

1. Spin up a docker container with the heroku:16 stack. This will build and behave exactly the same way a heroku:16 dyno except you will have write access. (make sure docker is running on your machine). From command line:
 
     ```bash
     $ docker run --rm -it heroku/heroku:16-build
     ```
 
_This will take you to an interactive bash shell as a root user inside the container. The `--rm` flag removes the docker process on exiting.  The `-ti` flag creates the interactive bash shell._
 
 
2. Get the libraries and dependencies you need(some of these already exist on the system):

     ```bash
     $ apt-get update && apt-get install build-essential autoconf libtool git-core
     $ apt-get build-dep imagemagick libmagickcore-dev libde265 libheif
     ```


3. Download the libwebp, libde265 and libheif libraries:

     ```bash
     $ git clone https://github.com/strukturag/libde265.git
     $ git clone https://github.com/strukturag/libheif.git
     $ curl -L --silent "http://downloads.webmproject.org/releases/webp/libwebp-1.1.0.tar.gz" | tar xz
     ```


4. install the libde265 library:

     ```bash
     $ cd libde265/
     $ ./autogen.sh && ./configure && make && make install
     ```


5. Install the libheif library:

    ```bash
    $ cd ../libheif/
    $ ./autogen.sh && ./configure && make && make install
    ```
    
6. Install the libwebp library:

    ```bash
    $ cd ../libwebp-1.1.0/
    $ ./configure && make && make install
    ```


7. Get, Configure and Install Newest Imagemagick:

    ```bash
    $ cd /usr/src/
    $ git clone https://github.com/ImageMagick/ImageMagick.git ImageMagick
    $ cd ImageMagick
    $ ./configure --prefix=/usr/src/imagemagick --with-heic=yes --with-webp --with-gvc=no --with-quantum-depth=8 --disable-static
    $ make && make install
    ```

_Take a break this will take a few min to install._


8. Copy the dependencies into imagemagick lib directory:

    ```bash
    $ cp /usr/local/lib/libde265.so.0 /usr/src/imagemagick/lib
    $ cp /usr/local/lib/libheif.so.1 /usr/src/imagemagick/lib
    $ cp /usr/lib/x86_64-linux-gnu/libomp.so.5 /usr/src/imagemagick/lib
    $ cp /usr/lib/x86_64-linux-gnu/libiomp5.so /usr/src/imagemagick/lib
    ```

_The last 2 libraries are not available at run time on heroku only build time see [Ubuntu Packages on Heroku Stacks](https://devcenter.heroku.com/articles/stack-packages) for more info_


9. Clean up the build and get ready for packaging:

    ```bash
    $ cd /usr/src/imagemagick
    $ strip lib/*.a lib/lib*.so*
    ```

[What does `strip` do?](https://en.wikipedia.org/wiki/Strip_(Unix))


10. Wrap it up with a bow(compress the binary):

    ```bash
    $ cd /usr/src/imagemagick
    $ rm -rf build
    $ mkdir build
    $ tar czf /usr/src/imagemagick/build/imagemagick.tar.gz bin include lib
    ```


11. Copy the compressed file/tarball from the docker container into the repo(_you need to have cloned this repo locally_):
 
    ```bash
    # List current running docker processes to find out the NAME of your container
    $ docker ps
    # copy the binary from the container to the build directory in the repo on your local machine
    $ docker cp <NAME_of_docker_container>:/usr/src/imagemagick/build/imagemagick.tar.gz <path_to_build_folder_in_git_repo>
    ```
     

**DO NOT EXIT YOUR CONTAINER or ALL will be lost, open a new tab in your terminal**
 
_You may need to delete the old tarball from the bin folder first or to be safe copy the file from the container to your local machine before adding to the repo so you have a copy of the old binary tarball._


12. Commit and Push to repo


### Clear cache(_Not Sure if this is necessary)
Since the installation is cached you might want to clean it out due to config changes.

1. `heroku plugins:install heroku-repo`
2. `heroku repo:purge_cache -app HEROKU_APP_NAME`

### Credits
https://medium.com/@eplt/5-minutes-to-install-imagemagick-with-heic-support-on-ubuntu-18-04-digitalocean-fe2d09dcef1
https://github.com/brandoncc/heroku-buildpack-vips
https://github.com/steeple-dev/heroku-buildpack-imagemagick
https://github.com/retailzipline/heroku-buildpack-imagemagick-heif
