# Concepts

The following describes a few core concepts in Azure AD referred to throughout this documentation.

## Tenants

A tenant represents an organization in Azure AD. Each tenant will have their own set of applications, users and groups. In order to log in to a tenant, you must use an account specific to that tenant.

NAV has two tenants in Azure AD:

- `nav.no` - available in all clusters, default tenant for production clusters
- `trygdeetaten.no` - only available in `dev-*`-clusters, default tenant for development clusters

!!! warning
    If your use case requires you to use `nav.no` in the `dev-*`-clusters, then you must [explicitly configure this](configuration.md#tenants).
    Note that you _cannot_ interact with clients or applications across different tenants.

The same application in different clusters will result in unique Azure AD clients, with each having their own client IDs and access policies. For instance, the following applications in the same `nav.no` tenant will result in separate, unique clients in Azure AD:

* `app-a` in `dev-gcp`
* `app-a` in `prod-gcp`

## Naming format

An Azure AD client has an associated name within a tenant. NAIS uses this name for lookups and identification.

All clients provisioned through NAIS will be registered in Azure AD using the following naming scheme:

```text
<cluster>:<namespace>:<app-name>
```

???+ example
    ```text 
    dev-gcp:aura:nais-testapp
    ```

## Scopes

A _scope_ is a parameter that is set during authorization flows when requesting a token from Azure AD.

It is used to indicate the intended _audience_ (the expected target resource) for the requested token, which is found in the `aud` claim in the JWT returned from Azure AD.

When consuming a downstream API that expects an Azure AD token, you must therefore set the correct scope to fetch a token
that your API provider accepts.

The scope has the following format:

```text
api://<cluster>.<namespace>.<app-name>
```

For example:

```text
api://dev-gcp.aura.nais-testapp/.default
```

### Default scope

The `/.default` scope is a static scope which indicates to Azure AD that your application is requesting _all_ available scopes that have been granted to your application. 

For example, if your application has access to `api://dev-gcp.aura.nais-testapp/defaultaccess` and `api://dev-gcp.aura.nais-testapp/read`, requesting `api://dev-gcp.aura.nais-testapp/.default` will return a token that contains both of these scopes.
If you want granularity, you may explicitly request the individual scopes instead as needed.

## Client ID

An Azure AD client has its own ID that uniquely identifies the client within a tenant, and is used in authentication requests to Azure AD.

Your application's Azure AD client ID is available at multiple locations:

1. The environment variable `AZURE_APP_CLIENT_ID`, available inside your application at runtime
2. In the Kubernetes resource - `kubectl get azureapp <app-name>`
3. The [Azure Portal](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps). You may have to click on `All applications` if it does not show up in `Owned applications`. Search using the naming scheme mentioned earlier: `<cluster>:<namespace>:<app>`.
