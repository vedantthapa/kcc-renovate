# kcc-renovate

_Failed_ Attempt at using Renovate bot to upgrade config connector operator.

Since config-connector doesn't provide a helm chart, the idea was to define a Flux [`GitRepository`](https://github.com/vedantthapa/kcc-renovate/blob/main/kubernetes/components/controllers/config-connector.yaml#L2-L11) resource pointing to the [official config-connector repository](https://github.com/GoogleCloudPlatform/k8s-config-connector) and a [flux kustomization resource](https://github.com/vedantthapa/kcc-renovate/blob/main/kubernetes/components/controllers/config-connector.yaml#L13-L35) pointing to the relevant location on the repo where operator configurations are stored. Renovate bot would then [submit a PR to update the tag](https://github.com/vedantthapa/kcc-renovate/commit/3fe62644e0d4f8a094e2340ece34063587b9abad) defined within the GitRepository resource, thereby upgrading the operator in the cluster once it's PR was merged.

Even though the CRDs are present, the repository doesn't seem to store the complete operator configuration. For example, [this kustomization file](https://github.com/GoogleCloudPlatform/k8s-config-connector/blob/master/operator/config/autopilot-manager/kustomization.yaml) within the `./operator/config/autopilot-manager/` patches the resources using a `manager_image_patch.yaml`, however, the repository instead only stores a `manager_image_patch_template.yaml` file which is [used during runtime to produce a `manager_image_patch.yaml`](https://github.com/GoogleCloudPlatform/k8s-config-connector/blob/master/operator/Makefile#L82-L83). This runtime hydration won't be possible within Flux.

Another possible target is `./install-bundles` to directly configure the controller and CRDs, but all of it's sub-directories have [`hostPort` defined in their relevant deployment manifests](https://github.com/GoogleCloudPlatform/k8s-config-connector/blob/master/install-bundles/install-bundle-autopilot-workload-identity/0-cnrm-system.yaml#L2543) which doesn't play well with GKE Autopilot constraints.

Google only [supports the distribution of config-connector manifests via it's storage bucket](https://github.com/GoogleCloudPlatform/k8s-config-connector/issues/575#issuecomment-981936933). Although it's possible to directly point to it using a Flux [`Bucket`](https://github.com/vedantthapa/kcc-renovate/blob/main/kubernetes/components/controllers/bucket.yaml) source, it raises the following error -
```sh
> k describe bucket -n flux-system
...
Events:
  Type     Reason                 Age                 From               Message
  ----     ------                 ----                ----               -------
  Warning  BucketOperationFailed  11m (x66 over 14h)  source-controller  fetch from bucket 'configconnector-operator' failed: failed to get '1.120.1/release-bundle.tar.gz' object: googleapi: Error 412: The type of authentication token used for this request requires that Uniform Bucket Level Access be enabled., conditionNotMet
```
