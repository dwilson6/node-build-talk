# node-build-talk
A demo of building up a highly efficient NodeJS project build for a CI env. It 
intentionally uses npm vs yarn or pnpm to show that you can gain efficiencies 
without switching to the newest shiny tool.

## Prereqs

* Docker
* NodeJS - install via nvm or directly install the version specified in .nvmrc
* curl

If using nvm...

```bash
nvm install
```

## Stages:

### 1. Create A Simple NodeJS Project

Set up a simple NodeJS project.

```bash
# Create package.json with defaults
npm init -y
# Set package to MIT
npm pkg set license=MIT
# ESM
npm pkg set type=module
```

Notice the package.json
created by npm init.

Add a project dependency

```bash
npm install express
```

Create an index.js file

```bash
# with vscode
code index.js

# or with vim
vim index.js
```

Add a simple express app in index.js

```javascript
import express from 'express';
const app = express();
app.get('/', (req, res) => {
    res.send('Hello World!');
});

app.listen(3000);
console.log('Server is listening on http://localhost:3000');
```

Run the server

```bash
node index.js
```

Hit the api endpoint from another terminal window

```bash
# if you don't have curl, open in a web browser
curl http://localhost:3000
```

Type `ctrl + c` in the original terminal window to kill the server.

If someone wants to run this simple server they need to:
1. Clone the repo
2. Install the correct nodejs version
3. Install npm dependencies
4. Run the server

If someone wants to do that 500 times a day, like say a CI server, the small
amount of time adds up quickly.

This is why we build artifacts. I could do that by zipping my index.js file 
and node_modules dir into an archive and storing it somewhere for later.

build.sh
```bash
apt-get install node@21.1.0 zip
git clone <myrepo>
npm install

mkdir build
zip -r build/my-archive.zip index.js node_modules
# to see the size of the zip
ls -alh build/my-archive.zip
# store the archive somewhere for retrieval on install, in this case a shared folder
cp build/my-archive.zip /artifacts/my-archive.zip
```

But then to run the server, something needs to know the nodejs version required
and what command to use. I could write an install script for that if I know what
operating system the server will be installed on.

install.sh
```bash
apt-get install node@21.1.0
# copy the archive from a shared folder
cp /artifacts/my-archive.zip /usr/src/app/my-archive.zip | unzip
```

run.sh
```bash
node index.js
```

That works but isn't very portable. What if we want to run this on alpine linux?

That is where Docker comes in handy.

### 2. Add Dockerfile

Create a simple Dockerfile for the project.

```bash
code Dockerfile
```

```Dockerfile
FROM node:21.6-slim
WORKDIR /app
COPY . /app
RUN npm install
CMD node index.js
```

Add a .dockerignore file to exclude node_modules so it only copies source files

```bash
echo "node_modules" > .dockerignore
```

Now we have a compact definition of how to build and how to run the server. To
build it, we would do:

```bash
docker build -t node-build-talk .
```

On my [old and slow] machine, a fresh build takes about 40 seconds:
* ~25 seconds to download/extract the base image
* ~11 seconds to run npm install

Run the docker build command above again. Now it takes less than 2 seconds.
Docker layer caching at work!

Try changing something in index.js

```diff
- res.send('Hello World!');
+ res.send('Good Morning!');
```

Now rerun the docker build command again. What happened?

The base image is downloaded/cached so the build is much faster (~11 seconds).
But it reran npm install even though we didn't change any dependencies. This is
because the `COPY . /app` command invalidated the cache since index.js changed.

For a normal project, running npm install can take 2-5 minutes so it is worth
making this step better.

### 3. Optimize NPM Install

There's a simple pattern to have npm install take advantage of the cache if the
no dependencies have changed.

```Dockerfile
FROM node:21.6-slim
WORKDIR /app
COPY package.json /app/
COPY package-lock.json /app/
RUN npm install
COPY . /app
CMD node index.js
```

Run the docker build to prime the build cache

```bash
docker build -t node-build-talk .
```

Now change something in index.js

```diff
- res.send('Good Morning!');
+ res.send('Good Afternoon!');
```

Then rerun the docker build again. What happened this time?

It only took 1.1 seconds because it used the cache for the npm install layer.

Bump the version in package.json

```diff
-1.1.0
+1.2.0
```

Now rerun the docker build and see that it runs npm install instead of using the
cache. Any changes in package.json and package-lock.json will invalidate the cache.

But now every time the docker cache is invalidated (package.json changes), npm
has to re-download all dependencies.

### 4. Add Docker Cache Volume

The next step is to enable the npm cache to be shared across docker builds on
the same machine. This can be done by mounting a cache volume for the npm cache
dir during the docker build.

Change the npm install line in the Dockerfile to mount a cache volume to
/root/.npm which is the default cache dir (`npm config get cache` will show that).

```diff
-RUN npm install
+RUN --mount=type=cache,target=/root/.npm npm install
```

Run the docker build to prime the cache volume

```bash
docker build -t node-build-talk .
```

Change the version in package.json. This will cause npm install to run the next
time we run the docker build.

```diff
-1.2.0
+1.3.0
```

Then run the docker build again. Note we change package.json instead of using
the `--no-cache` docker option since the latter affects cache volumes.

That shaved a little time off the npm install as npm reused its local cache for
the install. It took 3 seconds instead of 4. If your npm install took 90 seconds
normally, the savings would be significant.

Now we have a build that is fast in most cases but let's say we have a CI system
that creates new build nodes each day. That means the first build each day for
every branch will be slow because there will be no cache. 

If we make a small change in source code, the build will need to download the
base image and install dependencies despite no dependencies changing.
Thankfully, there's a solution for this case.

### 5. Docker Registry Caching

Docker builds can embed docker cache information so you can reference a remote
image to avoid building a docker layer that exists in that remote image but that
doesn't exist locally.

To enable this we'll add the `--cache-to` arg to the docker build command so it
will embed cache info in the image. We also change the tag so we can push it to
a registry (DockerHub or a private registry). Run `docker login` if not already
authenticated.

```bash
# change dwilson6 to your DockerHub username
docker build --cache-to type=inline -t dwilson6/node-build-talk:cache .
docker push dwilson6/node-build-talk:cache
```

Now we will remove the local image we built and the node base image so we can
test building from the cache.

```bash
docker rmi node-build-talk dwilson6/node-build-talk dwilson6/node-build-talk:cache
# confirm that the image is no longer there
docker images | grep node
```

Now add the `--cache-from` arg to the docker build command and it should not
need to build the image at all since there are no changes.

```bash
docker build --cache-to type=inline --cache-from dwilson6/node-build-talk:cache -t dwilson6/node-build-talk:latest .
```

Now change something in index.js which will make the next build do a partial build.

```diff
- res.send('Good Afternoon!');
+ res.send('Good Evening!');
```

Run the docker build again and now it should use the cache for the npm install
but not the COPY for the source code.

```bash
docker build --cache-to type=inline --cache-from dwilson6/node-build-talk:cache -t dwilson6/node-build-talk:latest .
```

Now make a change in package.json

```diff
-1.3.0
+1.4.0
```

Run the docker build again and now it should run the npm install and also the
COPY for the source code.

```bash
docker build --cache-to type=inline --cache-from dwilson6/node-build-talk:cache -t dwilson6/node-build-talk:latest .
```

Notice how it didn't even need to download the base image or cached image. You
could have a single cache image tag for the main branch or have one per branch.

Of course there are other improvements we can make that are outside the scope of this talk.

* Use multi stage builds to not ship dev dependencies or tests (and keep image size small)
* Use a build arg to inject the NodeJS version so it only needs to be specified in one place (.nvmrc)
* Add tests


This concludes our journey through efficient build caching with docker.


## Resources

Here's some helpful resources on these various caching strategies.

* [Docker Build Cache](https://docs.docker.com/build/cache/)
* [Exporting The Build Cache](https://docs.docker.com/build/cache/backends/)
* [2022 Guide On Remote Cache Support](https://www.docker.com/blog/image-rebase-and-improved-remote-cache-support-in-new-buildkit/)
