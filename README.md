# PowerDNS-Admin Helm

A Helm chart to deploy `pdns-admin` in a Kubernetes cluster.  See https://github.com/pschiffe/docker-pdns for more info on the project.

**Important:** Please note that this Helm chart is intended to install only the Web UI components, not PowerDNS itself.  Therefore, we only deal with the [pdns-admin-uwsgi](https://github.com/pschiffe/docker-pdns#pdns-admin-uwsgi) and [pdns-admin-static](https://github.com/pschiffe/docker-pdns#pdns-admin-static) components of the git project mentioned above.


### Usage

    helm repo add pdns-admin-helm https://raw.githubusercontent.com/boarder981/pdns-admin-helm/master/repo

    helm repo update

    helm install...


### Caveats

Kubernetes injects some default environment variables into each container.  It uses the deployment or container name to prefix these variables, then converts to uppercase and substitutes underscores.  For example, a deployment named `pdns-admin` would create containers with some environment variables prefixed with `PDNS_ADMIN_`.

This is problematic because the uwsgi (backend app) component copies env vars starting with `PDNS_ADMIN_` into the `uwsgi.ini` config file.  These values will NOT be single-quoted.  Due to a limitation documented [here](https://github.com/pschiffe/docker-pdns#pdns-admin-uwsgi), it will break the Python app as it tries to evaluate variables, rather than strings.

Therefore, you must use a name that **does not** begin with a variation of `pdns-admin`.  Specify a release name:

    helm install --name my-cool-pdns-ui ...

and/or set a name override:

    helm install ... \
    --set fullnameOverride="my-cool-pdns-ui"


### Custom Values

The `values.yaml` file contains default values, which most likely will not reflect your environment properly.  Please review each parameter and substitute the proper values.  This can be done either via `--set` like so:

    helm install ... \
      --set mysql.host="mysql.example.com" \
      --set powerdns.version="4.0.0" \
      ...

or in a custom values file:

    # values-override.yaml
    mysql:
      host: mysql.example.com

    powerdns:
      version: 4.0.0
    ...
    ...

run like this:

    helm install -f values-override.yaml ...


### Secrets

Sensitive parameters are required to run the app successfully:

* PowerDNS API key
* MySQL DB password
* LDAP user password (if using LDAP authentication)

By default, this chart assumes that you have a pre-existing secret named `powerdns-admin-secrets` in the desired Kubernetes namespace.  The secret must contain specific data keys.  **IMPORTANT:** the base64 encoded values need to be wrapped in single-quotes.  See the notes [here](https://github.com/pschiffe/docker-pdns#pdns-admin-uwsgi) to find out why.  An example yaml configuration for the secret would be:

    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: powerdns-admin-secrets
      namespace: mynamespace
    type: Opaque
    data:
      mysql_db_password: b64-encoded-string-single-quoted
      api_key: b64-encoded-string-single-quoted
      ldap_password: b64-encoded-string-single-quoted


If you wish to supply the values and let the chart create the secret, you have two options.  Keep in mind these options require sensitive information passed in clear text on the command line or stored in a file.  Proceed with caution.

1. Set the values at chart invocation

        helm install ... \
          --set secret.enabled=true \
          --set powerdns.apiKey="my-api-key" \
          --set mysql.dbPassword="my-db-password" \
          --set ldap.password="my-ldap-password"

2. Provide a custom values file

        secret:
          enabled: true

        mysql:
          dbPassword: my-db-password

        powerdns:
          apiKey: my-api-key

        ldap:
          password: my-ldap-password

   make sure to substitute all relevant values, then run the install command with file option:

        helm install ... \
          -f values-override.yaml


### Ingress Controller

You must have an ingress controller (i.e. nginx, traefik) configured for this helm chart to work properly.  There is no default value for this, so set it in your values override file like so:

    components:
      static:
        ingress:
          annotations:
            kubernetes.io/ingress.class: traefik
    ...
    ...


### Session Affinity

When deploying this with more than one replica, there is an issue where the user logged in to pdns-admin will frequently get kicked back to the login screen.  This is because on any given request, it may hit a different frontend (static) and/or backend (uwsgi) pod.

In order to prevent this, we use session affinity (stickiness).  This can be achieved with ingress annotations on the `static` service and native session affinity on the `uwsgi` service.

The annotations will depend on the Ingress Controller you're using (Traefik, NGINX, etc).  An example for Traefik would be:

    traefik.ingress.kubernetes.io/affinity: "true"
    traefik.ingress.kubernetes.io/session-cookie-name: "sticky"

Set these at `.Values.components.static.service.annotations`
