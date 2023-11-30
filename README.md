# OSS Backstage vs VMware Tanzu Developer Portal Workshop
A workshop that demonstrates the capabilities and challenges of opensource Backstage versus VMware Tanzu Developer Portal.

## OSS Backstage
## Tanzu Developer Portal (TDP)
[Documentation](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-about.html)

Tanzu Developer Portal is VMware’s commercial Backstage offering.

### Workshop Prerequisites
- Spin up a TAP developer sandbox session or use an existing TAP 1.7 environment
https://tanzu.academy/guides/developer-sandbox
- See prerequisites of the TDP Configurator [here](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-configurator-building.html#prerequisites-0)

### Add Custom Plugins using the Tanzu Developer Portal Configurator
![Process for TDP customization](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/Images/tap-gui-configurator-images-tdp-install-flowchart.png)

[Documentation](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-configurator-about.html)

**To integrate [Backstage open-source plugins](https://backstage.io/plugins) into TDP, they have to be wrapped in a small amount of code.** Otherwise, you would need access to the TDP source-code to integrate them in the same way it's done with OSS Backstage.

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

#### Prepare TDP Workload definition
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

##### Set the customized TDP image
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


#### Creating your own Tanzu Developer Portal plug-in wrapper for an existing Backstage plug-in
WIP: https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/tap-gui-configurator-create-plug-in-wrapper.html
