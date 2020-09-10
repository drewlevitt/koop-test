# koop-test
This is a test deployment of a Koop.js server as part of the Code for Pittsburgh food access map project.

The high-level idea is that we have a bunch of food access location data in a CSV with latlongs, and we have an ArcGIS Online web app to display those data, but we can only use critical AGOL features such as filter widgets if we make our data available in a format akin to an AGOL hosted feature layer. Enter [Koop.js](https://koopjs.github.io/): an open-source project from Esri that creates a lightweight web server that ingests geospatial data from a variety of formats and makes it available in a variety of formats, including Geoservices which should meet our needs here. 

In this repository, I'm setting up a Koop server and deploying it on Heroku, whose free tier should be sufficient to let us keep the Koop server up 24/7. We will probably need to recreate some or all of this work in an official [C4P](https://github.com/CodeForPittsburgh/) repository, but this is a start and maybe we can just fork it into the C4P GitHub account.

## How I Did It, by Victor Frankenstein
That's a *Young Frankenstein* reference, for what it's worth.

Goodness knows I'm not as good at documentation as Max, but I figure I'll keep some notes on what I did to get this working, so we can recreate the process later and elsewhere.

### Install Koop
Per [the Koop quickstart guide](https://koopjs.github.io/docs/basics/quickstart), Koop requires Node.js and `npm` (which is a part of Node.js). So:

1. Install [Node.js](https://nodejs.org/en/). I installed version 12.18.3 LTS but I imagine the 14.10.0 Current version would work fine too.
2. Verify that Node.js installation was successful by opening a terminal and typing `node -v` and/or `npm -v`. This latter command printed `6.14.6` for me.
3. Type `npm install -g @koopjs/cli` to install the Koop CLI. 

### Create the Koop app

1. In the *parent* directory of where you want the app to live (say, `C:\Users\drew\Documents\GitHub`), run `koop new app koop-test`. This will create a directory called `koop-test` and pre-populate a Git configuration, Node.js dependencies, etc.
2. I don't really understand how `koop new app` interacts with existing Git or GitHub repositories. I initially created a repo called `koop-test` on GitHub and cloned the repo to my local machine, but running `koop new app` overwrote the contents of this directory, or at least the files with name conflicts.
3. In the `koop-test` directory, you can run `npm start` to start the dev server (a quick server that is available locally for rapid development purposes). You can then visit http://localhost:8080/ to see the dev server in action. (Initially, all it does is display "Welcome to Koop!") You can also run `koop serve` and get the same result (I think).
4. We need the [CSV provider](https://github.com/koopjs/koop-provider-csv), so within the `koop-test` directory, run `koop add provider koop-provider-csv`.

### Configure the Koop CSV provider
The basic idea of Koop is that it ingests data from any of several **provider** plugins, intermediately converts the data to GeoJSON, then exports data to any of several **output** plugins. Koop comes with the GeoServices output plugin already installed (because, as far as I can tell, making geospatial data available via the GeoServices API is the primary point of Koop) but we need to install the CSV provider plugin separately.

Helpful documentation [here](https://github.com/koopjs/koop-provider-csv#configuration). 

1. Open `koop-test/config/default.json` in a text editor.
2. Define one or more CSV sources, following the format in the documentation.
3. It might be helpful to assign an `idField` in the config file - Koop chirps at me that no `idField` was set, but reassures me that it created an `OBJECTID` field for me instead.

### Receive some feature data!

1. Start the Koop dev server via `koop serve` and/or `npm start` (again, not sure whether there's a distinction).
2. Some trial and error (and some review of the [GeoServices specification](https://www.esri.com/~/media/Files/Pdfs/library/whitepapers/pdfs/geoservices-rest-spec.pdf)) yields the following endpoint as a location where the whole dataset will be returned in JSON format compatible with the GeoServices API: http://localhost:8080/koop-provider-csv/food-data/FeatureServer/0/query
3. However, this combination of Koop provider (CSV) and output (GeoServices) plugins, by default, returns the first 2000 features only. Our dataset has more than 2000 features (we can tell because `exceededTransferLimit` is `true`), so we need to pass a URL parameter overriding this default limit. The [Koop FAQ](https://koopjs.github.io/docs/basics/faqs) sheds some light on this - we can use `resultRecordCount` to get more than 2000 results. We could just pass `resultRecordCount=999999` or some other very large number, but this is inelegant. The FAQ doesn't mention it but a lucky guess reveals that `resultRecordCount=-1` causes the endpoint to return *all* results: http://localhost:8080/koop-provider-csv/food-data/FeatureServer/0/query?resultRecordCount=-1

### Install Heroku
Properly speaking, one doesn't "install Heroku" per se. Heroku is a [platform-as-a-service](https://en.wikipedia.org/wiki/Heroku) company, not a piece of software. By "install Heroku," rather, I mean "install software on my local machine that lets me interact with Heroku." Specifically, this is the [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli). The below checklist is basically stolen from that Heroku support article.

1. Install [Git](https://git-scm.com/downloads), a prerequisite for the Heroku CLI. (Or, because I already had Git installed, open Git Bash and run `git update-git-for-windows` just to make sure I'm using the latest version.)
2. Install [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli#download-and-install). 
3. Verify that installation was successful by opening a terminal and typing `heroku --version` - this printed `heroku/7.42.13 win32-x64 node-v12.16.2` for me.
4. Type `heroku login` to connect Heroku CLI with your (our!) Heroku login credentials.

### Configure the Koop app for deployment to Heroku
Another very helpful tutorial, [here](https://koopjs.github.io/docs/deployment/heroku).

1. Open `package.json` in the `koop-test` directory and specify the version of Node.js needed for our Koop app. This is simply the version of Node.js I installed earlier (v12.x). See the [guide](https://koopjs.github.io/docs/deployment/heroku) for the specific syntax needed.
2. Again pulling code from the tutorial, edit `src/index.js` to use the port number that Heroku assigns dynamically to each dyno (virtual machine) instead of the static port number specified in `config/default.json`.
    1. I found, when trying to run step 3 below, that the code provided by the Koop tutorial needed a slight change: `process.NODE_ENV.$PORT` needed to be replaced by `process.env.PORT`. This [Heroku help page](https://help.heroku.com/P1AVPANS/why-is-my-node-js-app-crashing-with-an-r10-error) helped me identify the correct syntax.
    2. In case you're wondering, like I was, where the `process` object referenced in the new code comes from, it is a [global variable](https://www.twilio.com/blog/working-with-environment-variables-in-node-js-html) created when a Node.js process starts up.
3. From the `koop-test` directory, run `heroku local web` to test whether the app will run correctly. This will make the app available at http://localhost:5000/ (note: different port number than before).

### Deploy the Koop app to Heroku
Still working from this [tutorial](https://koopjs.github.io/docs/deployment/heroku).

1. From the `koop-test` directory, run `heroku create` to create a new app on Heroku. This will give the app a random name, but we can [change the name](https://devcenter.heroku.com/articles/creating-apps#creating-an-app-without-a-name) later.
2. 


Below, for the record, is the boilerplate README.md created by `koop new app`.

# koop-cli-new-project

A minimal Koop project template from [Koop CLI](https://github.com/koopjs/koop-cli).

See the [specification](https://koopjs.github.io/docs/usage/koop-core) for more details.

## Configuration

This project is configured with [config](https://www.npmjs.com/package/config). As a community practice, it is recommended to namespace the configuration for plugins in order to avoid any potential key conflict.

## Development

### Testing

This project uses [mocha](https://www.npmjs.com/package/mocha) as the testing framework and [chaijs](https://www.chaijs.com/) as the assertion library. All test files in the `test` directory should have the special extension `.test.js`, which will be executed by the command:

```
$ npm test
```

### Dev Server

This project by default uses the [Koop CLI](https://github.com/koopjs/koop-cli) to set up the dev server. It can be invoked via

```
$ npm start
```

The server will be running at `http://localhost:8080` or at the port specified at the configuration.

For more details, check the [Koop CLI documentation](https://github.com/koopjs/koop-cli/blob/master/README.md).

