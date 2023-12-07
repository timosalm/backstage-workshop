# OSS Backstage vs VMware Tanzu Developer Portal Workshop
A workshop that demonstrates the capabilities and challenges of open-source Backstage versus VMware Tanzu Developer Portal.

[Slides](https://github.com/timosalm/backstage-workshop/blob/main/backstage-tdp.pdf)

## OSS Backstage

Backstage is an open platform for building developer portals. 
It is constructed out of three parts:
- Core: The base functionality.
- App: The app is an instance of a Backstage app that is deployed and tweaked. The app ties together core functionality with additional plugins.
- Plugins: Additional functionality to make your Backstage app useful for your company. Plugins can be specific to a company or open-sourced and reusable.

### Workshop Prerequisites
- Node 16 (e.g. installed via [nvm](https://github.com/nvm-sh/nvm))
- [Yarn](https://yarnpkg.com/getting-started)

To explore the UI and basic features of Backstage you can go to [demo.backstage.io](demo.backstage.io).

If you want to run Backstage on your machine or get started with the development of your Backstage app, I recommend following the [official guide](https://backstage.io/docs/getting-started/).

A new Backstage app can easily be created via the following command.
```
npx @backstage/create-app@latest
```
Change the directory to the sub-directory that was created based on the app name you chose.

Start the app with `yarn dev` command on your local computer, which will run both the frontend and backend as separate processes in the same window.
```
yarn dev
```

A sample backstage app is also available in this repository. You can run it with the following commands.
```
(cd oss-backstage && yarn install)
(cd oss-backstage && yarn dev)
```

A new Backstage app comes with the following core features. Have a look at the documentation to discover the capabilities of those in the related section. 
- [Backstage Search](https://backstage.io/docs/features/search/)
- [Backstage Software Catalog](https://backstage.io/docs/features/software-catalog/)
- [Backstage TechDocs](https://backstage.io/docs/features/techdocs/)
- [Backstage Software Templates](https://backstage.io/docs/features/software-templates/)
- [Backstage Kubernetes](https://backstage.io/docs/features/kubernetes/)

In addition, the following plugins are also installed ootb.
- [API Docs](https://github.com/backstage/backstage/blob/master/plugins/api-docs)
- [Github Actions](https://github.com/backstage/backstage/tree/master/plugins/github-actions)
- [Tech Radar](https://github.com/backstage/backstage/tree/master/plugins/tech-radar)

### Add plugins to Backstage

In addition to the core features of Backstage, there is a [large number of open-source plugins available](https://backstage.io/plugins) you can integrate, and you can also [create your own](https://backstage.io/docs/plugins/create-a-plugin).

Here are two examples of how to integrate plugins based on the documentation:
- [Tech Insights](https://github.com/timosalm/backstage-workshop/commit/2ab19d6ed8d62a6aaf2112425b775d64ed31dc58
)
- [Tech Docs](https://github.com/timosalm/backstage-workshop/commit/421a585ad77ec16e960b36e822d3b2cb26d3f3ef
)

## Tanzu Developer Portal (TDP)
[Documentation](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-about.html)

Tanzu Developer Portal is VMware’s commercial Backstage offering. In addition to the core features, like the Software Catalog, TechDocs and Search, it provides [several commercial plugins](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-plugins-about.html).

### Workshop Prerequisites
- Spin up a TAP developer sandbox VIP session or use an existing TAP 1.7 environment
https://tanzu.academy/guides/developer-sandbox-vip
- See prerequisites of the TDP Configurator [here](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-configurator-building.html#prerequisites-0)
- For TDP plugin wrapper creation: 
  * Node 16 (e.g. installed via [nvm](https://github.com/nvm-sh/nvm))
  * [Yarn](https://yarnpkg.com/getting-started)
  * A free https://www.npmjs.com account or a different NPM registry

### Add Custom Plugins using the Tanzu Developer Portal Configurator
![Process for TDP customization](tap-gui-configurator-images-tdp-install-flowchart.png)

[Documentation](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-configurator-about.html)

**To integrate [Backstage open-source plugins](https://backstage.io/plugins) into TDP, they have to be wrapped in a small amount of code.** Otherwise, you would need access to the TDP source code to integrate them in the same way it's done with OSS Backstage.

#### Prepare the Configurator configuration to define the custom TDP plugins 
The configuration for the Configuration has to be provided in the following format. Where the `name` value is the npm registry and module name, and the `version` is the desired front-end plug-in version that exists in the npm registry. The recommendation is to save it in a file named `tdp-config.yaml`
```
app:
  plugins:
    - name: "NPM-PLUGIN-FRONTEND"
      version: "NPM-PLUGIN-FRONTEND-VERSION"
backend:
  plugins:
    - name: "NPM-PLUGIN-BACKEND"
      version: "NPM-PLUGIN-BACKEND-VERSION"
```
**Let's use [this sample](tanzu-developer-portal-configurator/tdp-config.yaml) from the docs.**

[Here](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-configurator-tdp-plug-ins-list.html) is a list of currently available official TDP plugin wrappers for easy consumption. 

More unofficial plugin wrappers are available [here](https://github.com/vrabbi-tap/tdp-plugin-wrappers)

You will learn how to build your own TDP plugin wrappers for existing Backstage plug-ins [later](#creating-your-own-tanzu-developer-portal-plug-in-wrapper-for-an-existing-backstage-plug-in).

If you want to provide your TDP plugin wrappers via a private NPM registry you can learn in the documentation [here](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-configurator-private-registries.html) how to do it.

Note: Default TAP plug-ins cannot be removed from customized portals, but you can hide them via the `tap-gui.app_config.customize.features` properties in tap-values.yaml.

#### Prepare and apply TDP Workload definition
##### Identify your Configurator image
To build a customized TDP, you must identify the Configurator image to pass through the supply chain. 

Run the following command to identify the container image [bundle](https://carvel.dev/imgpkg/docs/v0.38.x/resources/#bundle) containing the Configurator image.
```
CONFIGURATOR_IMAGE_BUNDLE=$(kubectl get -n tap-install $(kubectl get package -n tap-install \
--field-selector spec.refName=tpb.tanzu.vmware.com -o name) -o \
jsonpath="{.spec.template.spec.fetch[0].imgpkgBundle.image}") && echo $CONFIGURATOR_IMAGE_BUNDLE
```

Authenticate with the registry the bundle is available on via `docker login` if the docker runtime is available on your machine or via [imgpkg environment variables](https://carvel.dev/imgpkg/docs/v0.38.x/auth/).

**If you're using TAP developer sandbox for this workshop**, you can run the following commands to fetch the required credentials from the cluster:
```
export IMGPKG_REGISTRY_HOSTNAME="us-docker.pkg.dev"
export IMGPKG_REGISTRY_USERNAME="_json_key_base64"
export IMGPKG_REGISTRY_PASSWORD=$(kubectl get secret -n tap-gui private-registry-credentials --output="jsonpath={.data.\.dockerconfigjson}" | base64 -d | jq -r '.auths."us-docker.pkg.dev".password')
```

After you're successfully authenticated, run the following command to get the Configurator image.
```
export CONFIGURATOR_IMAGE=$(imgpkg describe -b $CONFIGURATOR_IMAGE_BUNDLE -o yaml --tty=true | grep -A 1 \
"kbld.carvel.dev/id: harbor-repo.vmware.com/esback/configurator" | grep "image: " | sed 's/\simage: //g') && echo $CONFIGURATOR_IMAGE
```
##### Define and apply Workload
[Documentation](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-configurator-building.html#build-your-customized-portal-3)

There are **two options** for passing your workload through a supply chain and building your customized portal. You can use **one of the OOTB supply chains or a custom one**, which is documented in the "Use a custom supply chain" tab. Based on your choice the Workload definition looks different, as you have to provide more configuration with the OOTB supply chains.

I recommend using the custom supply chain, which is also the choice for this workshop.
The custom supply chain definition is available [here](tanzu-developer-portal-configurator/tdp-sc-template.yaml) in this repository. It's just copied from the documentation and included placeholders are modified towards a ytt variables or like the `tdp_configurator_bundle` parameter deleted as not required.
Therefore, you have to set some environment variables for this workshop.
- `REGISTRY_HOSTNAME` is the name of the container registry that your developer namespace was configured to push artifacts to
- `IMAGE_REPOSITORY` is the name of the repository (folder) on the REGISTRY-HOSTNAME that you want to push built artifacts to

**If you're using TAP developer sandbox for this workshop**, you can run the following commands to set the required env variables:
```
export REGISTRY_HOSTNAME=$(kubectl get clustersupplychain source-to-url -o jsonpath='{.spec.resources[?(@.name=="image-provider")].params[?(@.name=="registry")].value.server}')
export IMAGE_REPOSITORY=$(kubectl get clustersupplychain source-to-url -o jsonpath='{.spec.resources[?(@.name=="image-provider")].params[?(@.name=="registry")].value.repository}')
```
Apply the custom supply chain to the cluster with the following command.
```
ytt -f tanzu-developer-portal-configurator/tdp-sc-template.yaml -v tdp_configurator.sc.registry_server=$REGISTRY_HOSTNAME -v tdp_configurator.sc.registry_repository=$IMAGE_REPOSITORY | kubectl apply -f -
```

Let's have a closer look at the Workload YTT template, available in the repository [here](tanzu-developer-portal-configurator/tdp-workload-template.yaml).

By setting the `apps.tanzu.vmware.com/workload-type: tdp` label, the Workload will be run through our custom supply chain. The Configurator configuration, encoded as Base64, is set as the `TPB_CONFIG_STRING` environment variable for the container build.
Last but not least, the Configurator image is set as the `source` of the Workload.

Let's process the Workload template with our values and apply it to the Kubernetes cluster. The Kubernetes context should be set to a namespace that is configured as TAP developer namespace, and you want to run the workload in. Otherwise, change the `tanzu apps workload create` part of the command accordingly.
```
ytt -f tanzu-developer-portal-configurator/tdp-workload-template.yaml -v tdp_configurator.image=$(echo $CONFIGURATOR_IMAGE) --data-value-file tdp_configurator.config=tanzu-developer-portal-configurator/tdp-config.yaml | tanzu apps workload create -y -f -
```
Run `tanzu apps workload tail tdp-config --timestamp --since 1h` to view the build logs or view the status of the supply chain run in your Tanzu Developer Portal.

#### Set the customized TDP image
The custom TDP image build takes several minutes. After it finished you can get it via the following command.
```
export CUSTOM_TDP_IMAGE=$(kubectl get images.kpack.io tdp-config -o jsonpath={.status.latestImage}) && echo $CUSTOM_TDP_IMAGE
```

In the current version of TAP (1.7), the custom image will be applied via a YTT overlay, which is also [documented](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-configurator-running.html#prepare-to-overlay-your-customized-image-onto-the-currently-running-instance-1).

There are overlays with minimal differences available for an installation with the "lite"  (default) or the full Tanzu Build Service dependencies.
For this workshop, [the overlay](tanzu-developer-portal-configurator/tdp-overlay-secret-template.yaml) based on the documentation is again modified for YTT.
With the following command, you will apply a Secret with the YTT overlay. Please change the `data.values.tdp_configurator.full_dependencies` value to true, if you have the full Tanzu Build Services dependencies installed.
```
ytt -f tanzu-developer-portal-configurator/tdp-overlay-secret-template.yaml -v tdp_configurator.custom_image=$CUSTOM_TDP_IMAGE --data-value-yaml tdp_configurator.full_dependencies=false | kubectl apply -n tap-install -f -
```

Last but not least, you have to configure the overlay in your `tap-values.yaml` and update your TAP installation.
```
profile: full
tap_gui:
  ...
package_overlays:
- name: tap-gui
  secrets:
  - name: tdp-app-image-overlay-secret
```

**If you're using TAP developer sandbox for this workshop**, you can run the following command to directly change the related TAP configuration Secret.
```
UPDATED_TAP_VALUES=$(kubectl get secret tap-tap-install-values -n tap-install -o jsonpath='{.data.values\.yaml}' | base64 -d | grep -v '.*#! ' | sed "s/patch-tap-gui-timeout/patch-tap-gui-timeout\n  - name: tdp-app-image-overlay-secret/" | base64 -w0)
kubectl patch secret tap-tap-install-values -n tap-install --type json -p="[{\"op\" : \"replace\" ,\"path\" : \"/data/values.yaml\" ,\"value\" : ${UPDATED_TAP_VALUES}}]"
tanzu package installed kick tap -n tap-install -y
```
#### Discover your custom TDP plugin
Go to your Tanzu Developer Portal instance, select an available Software Catalog item for a workload and open the added TechInsights plugin tab.

If you don't have a catalog item with Workload available, you can register the one provided with this workshop: `https://github.com/timosalm/backstage-demo/blob/main/tanzu-developer-portal-configurator/catalog/catalog-info.yaml`

#### More advanced setup
To get an idea of a more advanced setup, you can have a look at mine, which is using GitOps.
[config/tap-install/portal-configurator](https://github.com/timosalm/tap-gitops-install-emea/tree/main/clusters/main-cluster/cluster-config/config/tap-install/portal-configurator)
[values/tdp-configurator-values.yaml](https://github.com/timosalm/tap-gitops-install-emea/blob/main/clusters/main-cluster/cluster-config/values/tdp-configurator-values.yaml)
[values/tap-values.yaml](https://github.com/timosalm/tap-gitops-install-emea/blob/4935e68571f5e5f74c2e5d515646e173d55cb1d9/clusters/main-cluster/cluster-config/values/tap-values.yaml#L117)
[config/tap-install/supply-chains/portal-configurator.yaml](https://github.com/timosalm/tap-gitops-install-emea/blob/main/clusters/main-cluster/cluster-config/config/tap-install/supply-chains/portal-configurator.yaml)

### Creating your own Tanzu Developer Portal plugin wrapper for an existing Backstage plug-in
The [official](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-configurator-create-plug-in-wrapper.html
) documentation for it is WIP. The current state can be viewed [here](https://github.com/benjaminleesmith/docs-tap/blob/main/tap-gui/configurator/create-plug-in-wrapper.hbs.md)

The Backstage plug-in you want to wrap has to be available in a public or private npm registry. 

For this workshop, we'll wrap the [Tech Radar plugin](https://github.com/backstage/backstage/tree/master/plugins/tech-radar), available as a package with the name [@backstage/plugin-tech-radar](https://www.npmjs.com/package/@backstage/plugin-tech-radar) at the public npmjs.com registry. This plugin only consists of a frontend component. 
Sample code is also available in this repository [here](tanzu-developer-portal-configurator/plugin-wrappers).

#### Setup a project
For the sake of simplicity, we'll use some of the Backstage tooling (`@backstage/create-app` and the backstage-cli).

To set up quickly a Backstage project, we use a utility for creating new apps. The easiest way to run it is via `npx`.
```
npx @backstage/create-app@latest --skip-install && cd plugin-wrappers
```
As an app name use for this workshop `plugin-wrappers`.

Remove not required folders.
```
rm -rf packages examples
```
The `packages` directory contains a Backstage app and backend which you only need to build a traditional Backstage app. You also have to remove it from the `workspaces.packages` configuration in the `package.json`.
```
  {
   ... 
   "workspaces": {
     "packages": [
-      "packages/*", # Remove this line
       "plugins/*"
     ]
   }
  }
```

Install the dependencies. 
```
yarn install --ignore-engines
```

#### Creating the Tech Radar plug-in TDP wrapper

##### Setup for the TDP wrapper
Set an environment variable with the name `NPM_REGISTRY_USERNAME` to the username (mine e.g. is [timosalm](https://www.npmjs.com/~timosalm)) of the npmjs.com registry or the prefix for your other registry. 
```
cd tanzu-developer-portal-configurator
yarn backstage-cli new --select plugin --option id=tech-radar-wrapper --scope @${NPM_REGISTRY_USERNAME} --no-private 
```

Remove not required folders, and create a src directory
```
(cd plugins/tech-radar-wrapper && rm -rf dev/ src/ && mkdir src)
```

Replace the dependencies of your plugin wrapper (`plugins/tech-radar-wrapper/package.json`) with 
```
...
  "dependencies": {
    "@backstage/plugin-tech-radar": "^0.6.9",
    "@backstage/core-components": "^0.13.8",
    "@vmware-tanzu/core-common": "1.0.0",
    "@vmware-tanzu/core-frontend": "1.0.0",
    "@material-ui/icons": "^4.9.1",
    "react-router": "6.0.0-beta.0"
  },
...
```
You should always check that the version of the plugin you want to wrap is compatible with TDP's Backstage version. [For 1.7 this is v1.15](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-plugins-about.html#backstage-version-compatibility-6). The `@vmware-tanzu/core-common` and `@vmware-tanzu/core-frontend` packages will be used later for the integration between the Backstage plug-in and TDP. The versions of those have to be compatible with your the TAP version.

Install the dependencies. 
```
(cd plugins/tech-radar-wrapper && yarn install --ignore-engines)
```

##### Wrap the Backstage plug-in
```
cat > plugins/tech-radar-wrapper/src/TechRadarPlugin.tsx <<EOL
import { TechRadarPage } from '@backstage/plugin-tech-radar';
import { AppPluginInterface, AppRouteSurface, SidebarItemSurface } from '@vmware-tanzu/core-frontend';
import { SurfaceStoreInterface } from '@vmware-tanzu/core-common';
import { SidebarItem } from '@backstage/core-components';
import TrackChangesIcon from '@material-ui/icons/TrackChanges';
import React from 'react';
import { Route } from 'react-router';

export const TechRadarPlugin: AppPluginInterface =
  () => (context: SurfaceStoreInterface) => {
    context.applyWithDependency(
      AppRouteSurface,
      SidebarItemSurface,
      (_appRouteSurface, sidebarItemSurface) => {
        _appRouteSurface.add(
          <Route path="/tech-radar"  element={<TechRadarPage width={1500} height={800} />} />
        )
        sidebarItemSurface.addMainItem(
          <SidebarItem icon={TrackChangesIcon} to='tech-radar' text='Tech Radar' />
        );
      }
    );
  }
EOL
```
The above code accomplishes the same thing as described in the documentation [here](https://www.npmjs.com/package/@backstage/plugin-tech-radar#configuration) for the integration of `@backstage/plugin-tech-radar` into a Backstage app but for an integration with TDP.

The `TechInsightsFrontendPlugin: AppPluginInterface = () => (context: SurfaceStoreInterface)`` => {} code is boilerplate that allows us to interact with the various frontend surfaces in Tanzu Developer Portal
`context.applyTo` is a function that takes the class of the surface you want to interact with, and a function that is passed the instance of that class.
The EntityPageSurface keeps track of tabs that appear on the service page. We add a new tab by calling entityPageSurface.servicePage.addTab and passing it the UI component we want it to render
The TechInsightsFrontendPlugin: AppPluginInterface = () => (context: SurfaceStoreInterface) => {} code is boilerplate that allows us to interact with the various frontend surfaces in Tanzu Developer Portal
The EntityPageSurface used above is one example of the many surfaces available in Tanzu Developer Portal.

To explore all the surfaces that are currently available checkout the How to use surfaces guide
For surface API reference documentation see the API documentation for surfaces

---
You also have to export `TechRadarPlugin` in a specific way to be used by the TDP Configurator.
```
cat > plugins/tech-radar-wrapper/src/index.ts <<EOL
export { TechRadarPlugin as plugin } from './TechRadarPlugin';
EOL
```

##### Build and publish your plugin wrapper
To build all your plugin wrappers at once, you can run the following command. 
```
yarn tsc && yarn workspaces run build
```
Otherwise, use this command:
```
(cd plugins/tech-radar-wrapper && yarn tsc && yarn build)
```

Log in to your npm registry.
```
yarn login
```

Publish the plugin wrapper:
```
(cd plugins/tech-radar-wrapper && yarn publish)
```

Go back to the parent directory.
```
cd ..
```
##### Add the new plugin wrapper to the Configurator configuration

Add the plugin wrapper to `tanzu-developer-portal-configuratort/dp-config.yaml`.
```
app:
  plugins:
    - name: "@vmware-tanzu/tdp-plugin-techinsights"
      version: "0.0.2"
    - name: "@<your-npm-registry-username>/plugin-tech-radar-wrapper"
      version: "0.1.0"
...
```

##### Trigger a new custom TDP image build and update the overlay Secret 
Update the Workload to build the new image.
```
ytt -f tanzu-developer-portal-configurator/tdp-workload-template.yaml -v tdp_configurator.image=$(echo $CONFIGURATOR_IMAGE) --data-value-file tdp_configurator.config=tanzu-developer-portal-configurator/tdp-config.yaml | tanzu apps workload apply -y -f -
```

After the supply chain run finished, update the Secret with the YTT overlay with the new image.
```
export CUSTOM_TDP_IMAGE=$(kubectl get images.kpack.io tdp-config -o jsonpath={.status.latestImage}) && echo $CUSTOM_TDP_IMAGE
kubectl delete secret tdp-app-image-overlay-secret -n tap-install
ytt -f tanzu-developer-portal-configurator/tdp-overlay-secret-template.yaml -v tdp_configurator.custom_image=$CUSTOM_TDP_IMAGE --data-value-yaml tdp_configurator.full_dependencies=false | kubectl apply -n tap-install -f -
tanzu package installed kick tap-gui -n tap-install -y
```

#### Discover your custom TDP plugin
Go to your Tanzu Developer Portal instance and open the Tech Radar plugin via the sidebar.






