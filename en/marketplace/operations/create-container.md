# {{ managed-k8s-name }} product requirements

To add a software product for {{ managed-k8s-name }} in {{ marketplace-short-name }}, upload its packages to the [registry](../../container-registry/concepts/registry.md) in {{ yandex-cloud }}. This section will help you prepare your {{ marketplace-short-name }} product for {{ managed-k8s-full-name }}.

You can also create products for {{ compute-full-name }} to run on [Linux](create-image.md) and [Windows](create-image-ms.md) operating systems by following the relevant guidelines.

## Hosting and naming images {#registry}

* [A Helm chart](https://helm.sh/docs/topics/charts/) and all its included docker images must be placed in the [registry](../../container-registry/concepts/registry.md) of the publisher created in {{ container-registry-full-name }}. For information on [creating a registry](../../container-registry/operations/registry/registry-create.md) and [uploading an image](../../container-registry/operations/docker-image/docker-image-push.md), see the relevant documentation sections.

* The name of the product's Helm chart should follow the format below:

   ```
   cr.yandex/<registry-id>/<vendor-name>/<product-name>/<chart>
   ```
   * `<registry-id>`: Publisher's registry ID.
   * `<vendor-name>`: Name of the product vendor.
   * `<product-name>`: Product name.
   * `<chart>`: Helm chart name.

* The names of the product's docker images should follow the format below:

   ```
   cr.yandex/<registry-id>/<vendor-name>/<product-name>/<component-name>:<tag>
   ```
   * `<registry-id>`: Publisher's registry ID.
   * `<vendor-name>`: Name of the product vendor.
   * `<product-name>`: Product name.
   * `<component-name>`: Name of the product component being provided as a docker image.
   * `<tag>`: Docker image tag. Don't use the `latest` tag.

During publication, all the images included in a product are moved from the publisher's registry to the public `yc-marketplace` registry. The entire product hierarchy defined by the publisher is maintained in the process.

> For example, the `cr.yandex/{{ tf-cloud-id }}/yandex-cloud/prometheus/pushgateway:1.0` image will be published as `cr.yandex/yc-marketplace/yandex-cloud/prometheus/pushgateway:1.0`.

For more information on working with the registry, see [{#T}](../../container-registry/operations/helm-chart/helm-chart-push.md) and [{#T}](../../container-registry/operations/docker-image/docker-image-push.md).

## Helm chart specifics {#special-requirements}

A Helm chart must contain a file called `values.yaml`, which will contain a list of all the docker images as parameters. Docker image names in `values.yaml` must start with the prefix `.Values` and point to the images hosted in the publisher's registry so that product publication and further installation in the user's cluster proceed without error.

Generic pod specification without parameters:

```yaml
# pod spec
spec:
   containers:
   - image: cr.yandex/<registry-id>/<vendor-name>/<product-name>/<component-name>:<tag>
```

A pod specification with the image name replaced by the YAML path variable described in `values.yaml`:

```yaml
# pod spec
spec:
   containers:
   - image: {{ .Values.images.pushgateway }}
```

```yaml
# values.yaml
images:
   pushgateway: cr.yandex/<registry-id>/<vendor-name>/<product-name>/<component-name>:<tag>
```

## Manifest {#manifest}

To publish a product, you need a manifest document that describes product deployment parameters. Prepare a manifest and [upload](../../storage/operations/objects/upload.md) it to {{ objstorage-full-name }}.

The manifest uses YAML format and contains the following data:

1. `helm_chart`: Required field. Contains the product's Helm chart name and tag.

   ```
   helm_chart:
     name: cr.yandex/<registry-id>/<vendor-name>/<product-name>/<chart>
     tag: <tag>
   ```

1. `requirements`: Required field. Required parameters of the cluster where the product will be deployed. This section must include the `k8s_version` parameter that defines the range of supported {{ k8s }} versions.

   ```
   requirements:
     k8s_version: ">=1.18"
   ```

1. `images`: Required field. Contains a list of metadata of the images included in the product. The values of image metadata variables are references in YAML Path format to variables from `values.yaml`. Entries can be in one of the following formats:

   * Image name, registry address, and tag are described in separate fields:

      ```
      images:
      - registry: images.app.image.registry
        name_without_registry: images.app.image.name
        tag: images.app.image.tag
      ```

      ```
      # values.yaml
      images:
         app:
          image:
            registry: "cr.yandex"
            name: "<registry-id>/<vendor-name>/<product-name>/<component-name>"
            tag: "<tag>"
      ```

   * Image name and registry address are described in one field, the tag in a different field:

      ```
      images:
        - name_with_registry: images.app.config.image.name
          tag: images.app.config.image.tag
      ```

      ```
      # values.yaml
      images:
        app:
          config:
            image:
              name: "cr.yandex/<registry-id>/<vendor-name>/<product-name>/<component-name>"
              tag: "<tag>"
      ```

   * The full path to the image is provided:

      ```
      images:
        - full: images.app.image.name
      ```

      ```
      # values.yaml
      images:
        app:
          image:
            name: "cr.yandex/<registry-id>/<vendor-name>/<product-name>/<component-name>:<tag>"
      ```

1. `user_values`: Optional parameter. A list of product variables that the user can override while installing or modifying an already installed product via the {{ yandex-cloud }}  management console. Each variable is described by the required fields below:

   * `name`: YAML Path of the variable from `values.yaml`.
   * `title`: Short variable description, can be either in Russian or English. The value must start with a capital letter.

     ```
     user_values:
     - name: app.port
       title:
       en: <English_title>
       ru: <Title_in_Russian>
     ```

   * `description`: Variable description, can be either in Russian or English. The value must start with a capital letter.

      ```
      user_values:
        - name: app.port
          title: <Title>
          description:
            en: <English_description>
            ru: <Description_in_Russian>
      ```

   * Variable type. Legal values:

      * `boolean_value`. May contain a default value.

        ```
        user_values
          - name: <Name>
            title: <Title>
            description: <Description>
            boolean_value:
              default_value: true
        ```

      * `integer_value`. May contain a default value, the <q>required</q> flag, and a range of legal values.

        ```
        user_values:
          - name: <Name>
            title: <Title>
            description: <Description>
            integer_value:
              default_value: <default_integer>
              required: true
              restrictions:
                min: <integer_1>
                max: <integer_2>
        ```

      * `string_value`. May contain the <q>required</q> and the <q>secret field</q> flags as well as a length limit for the value.

        ```
        user_values
          - name: <Name>
            title: <Title>
            description: <Description>
            string_value:
              required: true
              secret: true
              length_restrictions:
                min: <min_string_length>
                max: <max_string_length>
        ```

      * `string_selector_value`: String from a pre-defined list. May contain a default value, the <Q>required</q> flag, and a list of legal values.

        ```
        user_values
          - name: <Name>
            title: <Title>
            description: <Description>
            string_selector_value:
              default_value: <value_1>
              required: true
              values:
                - <value_1>
                - <value_2>
                - <value_3>
        ```

      * `integer_selector_value`: Integer value from a pre-defined list. May contain a default value, the <q>required</q> flag, and a list of legal values.

        ```
        user_values
          - name: <Name>
            title: <Title>
            description: <Description>
            integer_selector_value:
              default_value: <integer_1>
              required: true
              values:
                - <integer_1>
                - <integer_2>
                - <integer_3>
        ```

The variable values specified by the user when installing the product in a Kubernetes cluster will override the values from `values.yaml`.

## Example manifest and corresponding variable file {#examples}

### Manifest {#manifest}

```
# Link to helm chart in publisher registry.
helm_chart:
  name: cr.yandex/{{ tf-cloud-id }}/Vendor/Product/chart
  tag: 1.0-0

# Required parameters.
requirements:
  k8s_version: ">=1.18"

images:
  - registry: app1.image.registry
    name_without_registry: app1.image.name
    tag: app1.image.tag
  - name_with_registry: app2.config.image.name
    tag: app2.config.image.tag
  - full: another-whatever-key.subkey.name

# Configurable parameters that might be changed by end user during installation of product. Should be presented in values.yaml
# Supported types: integer, boolean, string, string selector, integer selector.
user_values:
  - name: app.port
    title:
      en: Application port
      ru: Application port
    description:
      en: Port that application will listen to
      ru: Port on which the application accepts incoming requests
    integer_value:
      default_value: 8080
      required: true
      restrictions:
        min: 8000
        max: 9000
  - name: app.tls.use
    title:
      en: TLS
      ru: TLS
    description:
      en: Use TLS
      ru: Use TLS
    boolean_value:
      default_value: true
  - name: app.admin.password
    title:
      en: Admin password
      ru: Administrator password
    description:
      en: Password of administrator, should be at least 8 symbols
      ru: An administrator password must be at least 8 characters long
    string_value:
      required: true
      secret: true
      length_restrictions:
        min: 8
        max: 20
  - name: app.selector.string
    title:
      en: Custom string selector
      ru: String option
    description:
      en: One value string selector
      ru: Selecting a single string value
    string_selector_value:
      default_value: opt1
      required: true
      values:
        - opt1
        - opt2
        - opt3
  - name: app.selector.integer
    title:
      en: Custom integer selector
      ru: Number option
    description:
      en: One value integer selector
      ru: Selecting a single integer value
    integer_selector_value:
      default_value: 1
      required: true
      values:
        - 1
        - 2
        - 3
```

### values.yaml variable file {#values}

```
# An example of values.xml related to publisher manifest above.
replicaCount: 1
podAnnotations: {}
podSecurityContext: {}
...
app1:
 image:
  registry: cr.yandex/{{ tf-cloud-id }}/
  name: service-images/application-1
  tag: 1.0
app2:
  name: application-name
  config:
  # image can be declared on any level
   image:
    name: cr.yandex/{{ tf-cloud-id }}/service-images/application-2
    tag: 2.0
   pullPolicy: IfNotPresent
another-whatever-key: # key name is not fixed
 subkey:
  name: cr.yandex/{{ tf-cloud-id }}/service-images/application-3:3.0
...

# values
app:
  port: 80
  tls:
    use: true
  admin:
    password: ""
  selector:
    string: "opt1"
    integer: 1  
```