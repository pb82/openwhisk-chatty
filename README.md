Setting up Apache OpenWhisk
===========================

## What is OpenWhisk?

It's a `Function-as-a-Service` platform, somewhat similar to AWS Lambda, but you can host it on your own Hardware. It allows you to not think about whole backend projects and infrastructure and only deploy single functions/units of work called `actions`.

In addition to `actions` OpenWhisk adds the concepts of `events`, `triggers` and `feeds`. `Events` come from outside sources and can call triggers which then lead to the invokation of one or multiple actions. Note that you don't need triggers, you can always just invoke actions directly (which is what we're doing in this doc).

`Feeds` are (as far as i understand) an abstraction of those outside sources of events: you can configure a feed implementation (WebSocket, Webhook, ...) that acts as a source of events.

## Deploying to Openshift

Our goal is to run OpenWhisk on a local Openshift cluster (oc cluster up). The projectodd team has a template for this in their [repository](https://github.com/projectodd/openwhisk-openshift). Make sure your local cluster is running and the `oc` cli is  functional. Clone the repository. Before running it, there is one change that we should make: it turned out that the memory settings for the controller were not sufficient. It would accept small actions but crash when a zip file with some bundled dependencies was submitted. Edit this line: https://github.com/projectodd/openwhisk-openshift/blob/master/template.yml#L29

Set the `-Xmx256m` value to `-Xmx512m`. Now you can execute the template by running:

```sh
$ oc process -f template.yml | oc create -f -
```

This will take some time but it should set up all pods.

## Set up the Openwhisk CLI

Download the latest CLI from here: https://github.com/apache/incubator-openwhisk-cli/releases/

You can put it either in your path or just run the binary directly with `./wsk`. Follow the setup steps described in this [section](https://github.com/projectodd/openwhisk-openshift#configuring-wsk) of the README.

Test the CLI:

```sh
$ wsk -i list
```

This should give you a list of all actions, events and triggers that are installed. The `-i` (insecure) parameter is required because we are most likely running with a untrusted certificate.

## Deploy a small action

Every action must have a `main` function. Save this to a file named `hello.js`:

```js
function main() {
  return {
    result: "Hello World"
  };
}
```

Then deploy the action:

`./wsk action create hello hello.js -i`. This should succeed. For any further update you can also use `update`: `./wsk action update hello hello.js -i`.

### Invoking the action:

Just run `./wsk action invoke hello --result -i`. The `--result` switch is required for us to see the returned data. You should see hello world. If you run it without the `result` switch you will get an invokation id that you can use to retrieve the result at some later point. Kind of like a `Promise` in JavaScript.

## Deploying a more complex action

A simple function is fine but sometimes you want to do more complex things like querying a database, talking to a Google Home etc. For this you probably need libaries. Fortunately OpenWhisk lets you do that. You can create a normal Node.js project with a package.json and install your dependencies (like what's done in this repository). Then just bundle everything including the node_modules directiry:

```sh
$ zip -r action.zip *
```

and submit the whole bundle with:

```sh
./wsk action create <action name> --kind nodejs:6 <path/to/action.zip> -i
```

Note that you need to submit the type of the action (`nodejs:6`) because OpenWhisk can't infer it from the file extension when you submit a bundle.

## Authentication

By default OpenWhisk uses just a generated username and password. When you want to use the Rest API instead of the CLI you have to provide the credentials. You can retrieve them by running:

```
$ ./wsk property get --auth -i
```

## Using the REST API to invoke an action

Set up the `AUTH` env var: 

```sh
$ export AUTH=`./wsk property get --auth -i | awk '{ print $3; }'`
```

Then just use curl to call e.g. the `hello` action:

`curl -u $AUTH https://openwhisk-openwhisk.127.0.0.1.nip.io/api/v1/namespaces/_/actions/hello -X POST --insecure`

(Adapt to URL to your actual OpenWhisk installation)

## Other noteworthy things:

Openwhisk has SKDs for [iOS](https://github.com/apache/incubator-openwhisk-client-swift), [JavaScript](https://github.com/apache/incubator-openwhisk-client-js) and [Golang](https://github.com/apache/incubator-openwhisk-client-go).
