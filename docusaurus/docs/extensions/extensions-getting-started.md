# Getting Started

This guide will walk through creating a new extension from scratch.

## Prerequisites

You will need a recent version of nodejs installed (Tested with node version: `v16.19.1`).

You'll also need the yarn package manager installed, which can be done with `npm install -g yarn`.

## Creating the Application

In order to develop a new Extension, you need an application UI to host it in during development. Rancher provides a helper to create a skeleton application for you.

### Creating the Skeleton App

`cd` to a folder not within the checkout and run:

```sh
yarn create @rancher/app my-app
cd my-app
```

This will create a new folder `my-app` and populate it with the minimum files needed.

> Note: If you don't want to create a new folder, but instead want the files created in an existing folder, use `yarn create @rancher/app .`

> Note: The skeleton application references the Rancher dashboard code via the `@rancher/shell` npm module.

You can run the app with:

```sh
yarn install
API=<Rancher Backend URL> yarn dev
```

> Note: You will need to have a Rancher backend available and the `API` environment variable above set correctly to reference it.

You should be able to open a browser at https://127.0.0.1:8005 and you'll get the Rancher Dashboard UI. Your skeleton application is a full Rancher UI - but referenced via `npm`.

The next step is to create an extension.

### Creating an Extension

Once again, Rancher provides a helper to add an extension. You can choose to have multiple extensions or a single extension within
the parent folder.

Go ahead and run the following command to create a new extension:

```sh
yarn create @rancher/pkg test
```

This will create a new UI Package in the `./pkg/test` folder.

#### ___Extension Options___

There are two options that can be passed to the `@rancher/pkg` script:
- `-t`: Creates additional boilerplate directories for types, including: 'l10n', 'models', 'edit', 'list', and 'detail'
- `-w`: Creates a workflow file ('build-extension.yml') to be used as a Github action. This will automatically build your extension and release a Helm chart.

> Note: Using the `-w` option to create an automated workflow will require additonal prequesites, see the [Release](#creating-a-release) section.

### Configuring an Extension

Replace the contents of the file `./pkg/test/index.js` with:

```ts
import { importTypes } from '@rancher/auto-import';
import { IPlugin } from '@shell/core/types';

// Init the package
export default function(plugin: IPlugin) {
  // Auto-import model, detail, edit from the folders
  importTypes(plugin);

  // Provide extension metadata from package.json
  plugin.metadata = require('./package.json');

  // Load a product
  // plugin.addProduct(require('./product'));
}
```

Next, create a new file `./pkg/test/product.js` with this content:

```ts
export function init($plugin, store) {
  const { product } = $plugin.DSL(store, $plugin.name);

  product({
    icon:                  'gear',
    inStore:               'management',
    removable:             false,
    showClusterSwitcher:   false,
  });
}
```

## Running the App

We've created a bare bones extension and exposed a new 'product' that will appear in the top-level slide-in menu. At this stage, it does
nothing other than that!

You should now be able to run the UI again with:

```sh
yarn dev
```

Open a web browser to https://127.0.0.1:8005 and you'll see a new 'Example' nav item in the top-level slide-in menu.

> Note: You should be able to make changes to the extension and the UI will hot-reload and update in the browser.

## Building the Extension

Up until now, we've run the extension inside of the skeleton application - this is the developer workload.

To build the extension so we can use it independently, run:

```sh
yarn build-pkg test
```

This will build the extension as a Vue library and the built extension will be placed in the `dist-pkg` folder.

## Loading Into Rancher

When we run `yarn dev`, our test extension will be automatically loaded into the application - this allows us to develop
the extension with hot-reloading. To test loading the extension dynamically, we can update configuration to tell Rancher not to include our extension.

To do this, create a new `.env` file in the root `test-app` folder, and add these contents:

```
EXCLUDES_PKG='test'
```

If necessary, bring in the environment variables by running `source .env`.

Now, run the UI with:

```sh
yarn dev
```

Open a web browser to https://127.0.0.1:8005 and you'll see that the Example nav item is not present - since the extension was not loaded.

> Note: You need to be an admin user to test Extensions in the Rancher UI

Go to the user avatar in the top-right and go to 'Preferences'. Under 'Advanced Features', check the `Enable Extension developer features' checkbox.

Now, bring in the slide-in menu (click on the hamburger menu in the top-left) and click on 'Extensions'.

Go to the three dot menu and select 'Developer load' - you'll get a dialog allowing you to load the extension into the UI.

In the top input box `Extension URL`, enter:

```
https://127.0.0.1:8005/pkg/test-0.1.0/test-0.1.0.umd.min.js
```

Press 'Load' and the extension will be loaded, you should see a notification telling you the extension was loaded and if you bring in the side menu again, you should see the Example nav item there now.

This illustrates dynamically loading an extension.

> Note that when we started the UI, it serves up any extensions in the `dist-pkg` folder under the `/pkg` route of the app. Also note that when we build extensions they are versioned, so you'll see that reflected in the URL we used.

### Loading into another Rancher Instance

In the steps above, we were able to load the extension into our test application. We can load the extension into any running Rancher instance.

Run the following:

```sh
yarn serve-pkgs
```

This will start a small web server (on port 4500) that serves up the contents of the `dist-pkg` folder. It will output which extensions are being served up - in our case you should see output like that below - it shows the URLs to use for each of the available extensions.

```console
Serving catalog on http://127.0.0.1:4500

Serving packages:

  test-0.1.0 available at: http://127.0.0.1:4500/test-0.1.0/test-0.1.0.umd.min.js
```

In a different Rancher UI, you should be able to follow the steps in the previous section, but instead use the URL from the output above in the Developer Load dialog.

You'll notice that if you reload the Rancher UI, the extension is not persistent and will need to be added again. You can make it persistent by checking the `Persist extension by creating custom resource` checkbox in the Developer Load dialog.

## Creating a Release

Creating a Release for your extension is the official avenue for loading extensions into any Rancher instance. As mentioned in the [Introduction](./introduction.md), the extension can be packaged into a Helm chart and added as a Helm repository to be easily accessible from your Rancher Manager.

We have created a workflow for [Github Actions](https://docs.github.com/en/actions) which will automatically build, package, and release your extension as a Helm chart. Then it will give your Github repository a [Helm repository](https://helm.sh/docs/topics/chart_repository/) endpoint which we can use to consume the chart in Rancher. 

### Release Prerequisites

In order to have a Helm repository you will need to create the (`gh-pages`) on your Github repository.

### Adding the Release Workflow

To add the workflow to your extension, use the `-w` option when running the `@rancher/pkg` script. For instance:

```sh
yarn create @rancher/pkg test -w
```

This will create a `.github` directory within the root folder of your app which will contain the `build-extension.yml` workflow file. Initially the release is gated by a Push or Pull Request targeting the `main` branch. To update your workflow with different events to trigger the workflow, you can find more information in the [Github docs](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows).

> Note: If you wish to build and publish the Helm chart manually or to a specific registry, you can follow the steps listed in the [Advanced section](./advanced#publishing-the-extension-manually).

### Consuming the Helm chart

After releasing the Helm chart you will be able to consume this from the Rancher UI by adding your Helm repository's URL to the App -> Repository list. If you used the automated workflow to release the Helm chart, you can find the URL within your Github repository under the "github-pages" Environment. 

The URL should be listed as: `https://<organization>.github.io/<repository>`

Once the URL has been added to the repository list, the extension should appear within the Extensions page.

## Wrap-up

This guide has showed you how to create a skeleton application that helps you develop and test one or more extensions.

We showed how we can develop and test those with hot-reloading in the browser and how we can build our extensions into a package that we can dynamically load into Rancher at runtime. We also went over how to release our extensions as Helm charts using the automated workflow.
