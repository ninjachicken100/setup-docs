# Overview

This guide is for Dev Team to deploy LibreChat on **OpenShift Local** for development testing purposes.


## Install Guide
For files that are applied, please refer to the [Github Repo](https://github.com/ninjachicken100/setup-docs)


### Installing LibreChat

Pull from [LibreChat](https://github.com/danny-avila/librechat) Github Repo. You only require the helm folder, all other files may be removed.


**Create a new namespace**

```
oc new-project librechat
```

Configure the following files as required from the [Github Repo](https://github.com/ninjachicken100/setup-docs/resource-files/librechat) and apply the files.

You may use the default mongodb used in Librechat's version. In that case, only **librechat-secrets.yaml** need to be applied.

```
oc apply -f librechat-dhi-mongo-policy.yaml -f librechat-secrets.yaml -f mongodb-auth-secrets.yaml -f dhi-secrets.yaml -n librechat
```

Configure [librechat-values-override.yaml](https://github.com/ninjachicken100/setup-docs/blob/main/resource-files/librechat/librechat-values-override-sample.yaml) file and helm install

```
helm install librechat ./ -f librechat-values-override.yaml
```


### Integrating with Keycloak
Create a new realm *e.g. chat-applications*

In the Clients tab, click on 'Create Client' and configure with the following settings:

- **Client ID**: `$APP_NAME-ID`
- **Client Authentication**: `On`
- **Check Direct access grants and Service account roles**

- **Root URL/ Home URL/ Valid post logout redirect URIs/ Web origins:** `$APPLICATION_ROUTE`

- **Valid redirect URIs**: `$APPLICATION_ROUTE/oauth/openid/callback`

Under the Credentials tab, copy the client Secret and paste it into the secret file to be applied for the application

*e.g. if using librechat, paste into `OPENID_CLIENT_SECRET:` in the librechat-sample-secrets.yaml*