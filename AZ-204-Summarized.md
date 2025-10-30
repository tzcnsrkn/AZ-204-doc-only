# [Resource group](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal)

Resource group is a logical container into which Azure resources (web apps, databases, storage aacounts) are deployed and managed.

**Required** by all services/resources to be created.  
Updating resource requires resource group to be passed (unless resource name is unique).

API: [az group create](https://learn.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az-group-create) | [New-AzResourceGroup](https://learn.microsoft.com/en-us/powershell/module/az.resources/new-azresourcegroup)

```sh
az group create
    --name
    --location
```

Move resource to a different resource group / subscription:

```sh
$res1=$(az resource show -g OldRG -n res1Name --resource-type "<type>" --query id --output tsv)
az resource move --destination-group NewRG --ids $res1 # Make sure NewRG already exists
# Add `--destination-subscription-id $newSubscription` to change subscription
```

- `Microsoft.Web/sites`: Web app
- `Microsoft.Web/serverfarms`: App service plan

<br>

# [Azure API Management](https://docs.microsoft.com/en-us/azure/api-management/)

## [Tiers](https://learn.microsoft.com/en-us/azure/api-management/api-management-features)

| Feature                                 | Consumption                                                                   | Developer                                              | Basic                                               | Standard                                        | Premium                                                                     | Basic v2 / Standard v2 / Premium v2                                                                                                           |
| :-------------------------------------- | :---------------------------------------------------------------------------- | :----------------------------------------------------- | :-------------------------------------------------- | :---------------------------------------------- | :-------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------- |
| **Intended Use**                        | Serverless, pay-per-call for bursty or event-driven APIs                      | Dev / test, full feature parity (non-prod)             | Small production workloads                          | Production; typical enterprise workloads        | Global scale, enterprise, multi-region deployments                          | Newer platform with improved reliability and options                                                                                          |
| **SLA**                                 | SLA with limited features                                                     | No SLA                                                 | SLA provided (classic Basic)                        | SLA provided                                    | SLA provided                                                                | SLA provided; v2 brings enhancements                                                                                                          |
| **Scale / Multi-Region**                | Auto-scale; single-region gateway instance                                    | Single instance, not for production scale              | Single-region scale with fixed capacity             | Multi-unit scale in a region; higher throughput | Multi-region deployment, high throughput, multi-gateway                     | Improved scale and isolation options vs classic SKUs                                                                                          |
| **Virtual Network / Private Endpoints** | No VNet integration (public gateway)                                          | VNet injection only                                    | No VNet integration/injection                       | No VNet integration/injection                   | VNet injection only + multi-region networking                               | Standard v2+ : inbound private endpoint connections (simplified integration w/Azure Private Link) <br> Premium v2: VNet injection+integration |
| **Self-Hosted Gateway**                 | ❌                                                                            | ✅                                                     | ❌                                                  | ❌                                              | ✅                                                                          | ✅ (Premium v2 only)                                                                                                                          |
| **Estimated Throughput (requests/sec)** | Variable; auto-scale. Practical limits depend on region and policy complexity | Low; not intended for high throughput (hundreds req/s) | Moderate; hundreds to ~1k req/s per unit (estimate) | Higher; ~1k-2k req/s per unit (estimate)        | High; several thousand req/s per unit (typical guidance ~4k req/s per unit) | Similar to corresponding non-v2 SKUs but with improved efficiency; throughput per unit varies by v2 SKU                                       |

**📝 NOTE:** Developer tier supports Private Endpoints, BUT you must choose one or the other: VNet Injection or Private Endpoint. You cannot use both simultaneously.

## Components

- **API Gateway**: Endpoint that routes API calls, verifies credentials, enforces quotas and limits, transforms requests/responses specified in policy statements, caches responses, and produces logs/metrics for monitoring.
- **Management Plane**: Administrative interface for service settings, define and import API schema, packaging APIs into products, policy setup, analytics, and user management.
- **Developer Portal**: Auto-generated website for API documentation that enables developers to access API details, use interactive consoles, create an account and subscribe for API keys, analyze usage, download API definitions, and manage API keys.

**📝 NOTE:** Every tier except Consumption features Developer Portal.

## Products

Products bundle APIs for developers. They have a title, description, and usage terms. They can be **Open** (usable without subscription) or **Protected** (requires subscription). Subscription approval is either auto-approved or needs admin approval.

## Groups

Groups control product visibility to developers. API Management has three non-removable/immutable system groups:

- **Administrators**: Manage service instances, APIs, operations, and products. Azure subscription admins are in this group.
- **Developers**: Authenticated portal users can build apps using your APIs. They access the Developer portal to use API operations and can be created by admins, invited, or self-registered. They belong to multiple groups and can subscribe to group-visible products.
- **Guests**: Unauthenticated portal users with potential read-only access, such as viewing APIs but not calling them.

Apart from system groups, admins can form custom groups or use external groups from related Microsoft Entra ID tenants.

## [API Gateways](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/gateway)

API gateways, such as Azure's API Management gateway, manage communication between clients and multiple front- and back-end services, which eliminates the need for clients to know specific endpoints (think of gateways as _reverse proxy_). These gateways streamline service integration, particularly when services are updated or introduced. They also take care of tasks like SSL termination, authentication, and rate limiting. Azure's API Management gateway specifically proxies API requests, enforces policies, and gathers telemetry data.

- **Managed** gateways are default components deployed in Azure for each API Management instance. They handle all API traffic, regardless of where the APIs are hosted.
- **Self-hosted** gateways are optional, containerized versions of managed gateways. They are suited for hybrid and multicloud environments, allowing management of on-premises APIs and APIs across multiple clouds from a single Azure API Management service.

## [Policies overview](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-policies)

A an XML formatted set of statements executed sequentially on an API's request or response. They can be applied at different scopes:

- **Global Scope**: ALL APIs in the APIM instance. Apply organization-wide policies that affect every API and operation. Ideal for enforcing security measures like IP filtering or logging across all APIs.
- **Workspace Scope**: Target APIs within a specific workspace. Best for internal organization, separating APIs based on teams or departments.
- **Product Scope**: Group multiple APIs under a single product to simplify the subscription process. Best for external organization, grouping APIs based on functionality or business logic.
- **API Scope**: Apply policies that affect all operations within a specific API. Useful for API-specific transformations or validations.
- **Operation Scope**: Fine-grained control over individual API operations. Ideal for operation-specific validations or transformations (e.g. GET /users/{id}).

### Types of policies

- **Inbound**: Applied before routing to the backend. Examples: validation, authentication, rate limiting, request transformation.
- **Backend**: Applied before the request reaches the backend. Examples: URL rewriting, setting headers.
- **Outbound**: Applied to the response before sent to the client. Examples: response transformation, caching, adding headers.

**📝 NOTE:** If client expects response in certain format (example: XML), check question to see what kind of endpoint is used (example: JSON). If they are different, transform policy with inbound/outbound direction should be applied (example: `json-to-xml` OR `xml-to-json`).

### Policy Configuration

`<base />`: execute the default policies that are defined at parent scopes. Provides the ability to enforce policy evaluation order.

```xml
<policies>
  <inbound>
    <!-- statements to be applied to the request go here -->
  </inbound>
  <backend>
    <!-- statements to be applied before the request is forwarded to
         the backend service go here -->
  </backend>
  <outbound>
    <!-- statements to be applied to the response go here -->
  </outbound>
  <on-error>
    <!-- statements to be applied if there is an error condition go here -->
  </on-error>
</policies>
```

**All times are in seconds!**  
**All sizes are in KB!**

### [Named values](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-properties?tabs=azure-cli)

Simple use case: marked as secret, are stored encrypted, and can be referenced in policies using {{_yourNamedValue_}}.
Add a named value: `Dashboard > API Management Services > `_your-service-name_` > Named values`

**Types:**

- Plain: Literal string or policy expression
- Secret: Literal string or policy expression that is encrypted by API Management
- [Key vault](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-properties): Identifier of a secret stored in an Azure key vault. After update in the key vault, a named value in API Management is updated within 4 hours. Requires managed identity. Configure either a _key vault access policy_ or _Azure RBAC access_ for an API Management managed identity. Key Vault Firewall requires _system-assigned managed identity_.

Add a secret:

```sh
az apim nv create --resource-group $resourceGroup \
    --display-name "named_value_01" --named-value-id named_value_01 \
    --secret true --service-name apim-hello-world --value test
```

To use a named value in a policy, place its display name inside a double pair of braces like `{{ContosoHeader}}`. If the value is an expression, it will be evaluated at runtime. However, named values cannot reference other named values, i.e., `{{MyNamedValue}}` cannot contain `{{AnotherNamedValue}}`.

### Policy Expressions

Can be used as attribute values or text values in any of the API Management policies. They can be a single C# statement enclosed in `@(expression)` or a multi-statement C# code block, enclosed in `@{expression}`, that returns a value.

Example `set-body`:

```xml
<set-body>
  @{
    var response = context.Response.Body.As<JObject>();
    foreach (var key in new [] {"minutely", "hourly", "daily", "flags"}) {
    response.Property (key).Remove ();
    }
  return response.ToString();
  }
</set-body>
```

### [Throttling](https://learn.microsoft.com/en-us/azure/api-management/api-management-sample-flexible-throttling)

Use `rate-limit-by-key` (limit number of requests) or `quota-by-key` (limit bandwidth and/or number of requests). Renewal period is in _seconds_, bandwidth size is in _KB_. Use `counter-key` to specify identity or IP.

When quota is exceeded, a `403 Forbidden` status is returned.

- Throttle by IP: `counter-key="@(context.Request.IpAddress)"`
- Throttle by Subscription key: `@(context.Subscription.Id)`
- Throttle by Identity: `counter-key="@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Subject)"`

### [Caching](https://learn.microsoft.com/en-us/azure/api-management/api-management-sample-cache-by-key)

Azure APIM has built-in support for HTTP response caching using the resource URL as the key. You can modify the key using request headers with the vary-by properties.

[In a common example in official doc, policies implemented as below:](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-cache)
- Get from cache (`cache-lookup`): inbound
- Store to cache (`cache-store`): outbound
- Others are mixed

#### Examples

##### Cache by header

```http
https://myapi.azure-api.net/me
Authorization: Bearer <access_token>
```

Endpoint is not unique, but the authorization header is for each user.

```xml
<cache-lookup>
    <vary-by-header>Authorization</vary-by-header>
</cache-lookup>
```

##### Cache by query parameter

```http
https://myapi.azure-api.net/samples?topic=apim&section=caching
```

Endpoint has two query parameters: `topic` and `section`. Use semicolon to separate.

```xml
<cache-lookup>
    <vary-by-query-parameter>topic;section</vary-by-query-parameter>
</cache-lookup>
```

##### Cache by url

```http
https://myapi.azure-api.net/items/123456
```

Leave `<vary-by-query-parameter>` empty so that caching is based on the full URL path (which is unique), not on query parameters.

```xml
<cache-lookup>
    <vary-by-query-parameter></vary-by-query-parameter>
</cache-lookup>
```

##### Fragment caching

When you want to add some information from external system to the current response, without fetching it every time. For example `/me/tasks` returns user's todos and profile, but the profile is stored at `/userprofile/{userid}`. To avoid fetching profile every time, the following rules must be implemented:

```xml
<policies>
    <inbound>
        <!-- Extract userId from JWT -->
        <set-variable
            name="enduserid"
            value="@(context.Request.Headers.GetValueOrDefault("Authorization","").Split(' ')[1].AsJwt()?.Subject)" />

        <!-- Data is supposed to be stored in userprofile (for example) -->
        <!-- Look for userprofile from cache -->
        <cache-lookup-value
            key="@("userprofile-" + context.Variables["enduserid"])"
            variable-name="userprofile" />

        <!-- If userprofile is not cached yet, send a request and store the response in cache -->
        <choose>
            <when condition="@(!context.Variables.ContainsKey("userprofile"))">
                <!-- Make HTTP request to get user profile -->
                <send-request
                    mode="new"
                    response-variable-name="userprofileresponse"
                    timeout="10"
                    ignore-error="true">

                      <!-- Build a URL that points to the profile for the current end-user -->
                      <set-url>@(new Uri(new Uri("https://apimairlineapi.azurewebsites.net/UserProfile/"),(string)context.Variables["enduserid"]).AbsoluteUri)</set-url>
                      <set-method>GET</set-method>
                </send-request>

                <!-- Store response body in context variable -->
                <set-variable
                  name="userprofile"
                  value="@(((IResponse)context.Variables["userprofileresponse"]).Body.As<string>())" />

                <!-- Store to cache -->
                <cache-store-value
                  key="@("userprofile-" + context.Variables["enduserid"])"
                  value="@((string)context.Variables["userprofile"])"
                  duration="100000" />
            </when>
        </choose>
        <base />
    </inbound>
    <outbound>
        <!-- Update response body with user profile-->
        <find-and-replace
              from='"$userprofile$"'
              to="@((string)context.Variables["userprofile"])" />
      <base />
    </outbound>
</policies>

```

## API Security

### [Via Entra ID](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-protect-backend-with-aad)

1. Register an application in Entra ID to represent the API (e.g. in Portal: `App registrations > New registration > ... > Register`).
2. Configure a JWT validation policy to pre-authorize requests (`validate-jwt`)

### [Via Managed Identities](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-managed-service-identity)

Use the `authentication-managed-identity` policy to authenticate with a service through managed identity. It gets an access token from Microsoft Entra ID and sets it in the `Authorization` header using the Bearer scheme. The token is cached until it expires. If no client-id is given, the system-assigned identity is used.

Example:

```xml
<authentication-managed-identity
    resource="azure-resource-you-want-to-access, e.g.: https://storage.azure.com"
    client-id="your-user-assigned-managed-identity-client-id"
    output-token-variable-name="accessToken"
    ignore-error="false" />
```

Use cases:

- Obtain a custom TLS/SSL certificate for the API Management instance from Azure Key Vault
- Store and manage named values from Azure Key Vault
- Authenticate to a backend by using a user-assigned identity
- Log events to an event hub

### Via Subscriptions

API Management lets you secure APIs using subscription keys. Developers have to include these keys in HTTP requests when accessing APIs. If not, API Management gateway rejects the requests. The subscription keys come from subscriptions, which developers can get without needing permission from API publishers. Apart from this, OAuth2.0, Client certificates, and IP allow listing are other security methods.

#### [Subscription Keys](https://learn.microsoft.com/en-us/azure/api-management/api-management-subscriptions)

A subscription key is a unique, automatically generated key included in client request headers or as a query string parameter. It's tied to a subscription which can have different scopes providing varied access levels. Subscriptions let you control permissions and policies minutely.

**📝 NOTE:** Rate limiting by subscription key is not available at the Consumption tier.
Three main subscription scopes:

- **All APIs**: Grants access to all APIs configured in the service.
- **Product**: Applies to a particular product (a collection of APIs) in API Management, each with different access rules, usage quotas, and terms of use.
- **Single API**: Access control limited to a specific API and its endpoints.

Keys need to be included in every request to a protected API. They can be regenerated if needed, such as if a key is leaked. Subscriptions have a primary and a secondary key to aid in regeneration without downtime. In products with enabled subscriptions, clients must provide a key when calling APIs. They can get a key via a subscription request.

##### Calling API with Subscription Key

The default header name is **Ocp-Apim-Subscription-Key**, and for the default query string is **subscription-key**. APIs can be tested using the developer portal or command-line tools like curl. Here are example curl commands using a header and a URL query string:

```sh
curl --header "Ocp-Apim-Subscription-Key: <key string>" https://<apim gateway>.azure-api.net/api/path
```

```sh
curl https://<apim gateway>.azure-api.net/api/path?subscription-key=<key string>
```

Failure to pass the key results in a **401 Access Denied** response.

### [API Security with Certificates](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-mutual-certificates-for-clients)

- `authentication-certificate` (_inbound_): Used by APIM to authenticate itself to the backend service.
- `validate-client-certificate` (_inbound_): Used by APIM to validate the client's certificate connecting to the APIM.

Configure access to key vault:

- Access policy: `Access Policies > + Create > Secret permissions > Permissions tab > Get and List ; Principal tab > principal > managed identity > Next > Review + create > Create`
- RBAC access: `Access control (IAM) > Add role assignment > Role tab > Key Vault Secrets User ; Members tab > Managed identity > + Select members > identity`

#### TLS Client Authentication

API gateways inspect client certificates for specific attributes, such as Certificate Authority (CA), Thumbprint, Subject, and Expiration Date. These can be combined for custom policies.

Certificates are signed to avoid tampering. Validate received certificates to ensure they're authentic. Trusted CAs or physically delivered self-signed certificates are ways to confirm authenticity.

In the _Consumption_ tier client certificates must be _manually enabled_ on the **Custom domains** page.

**📝 NOTE:** The _CA certificates_ blade in the APIM portal only applies to the managed gateway. Self-Hosted Gateways (SHGW) manage their own trust. To make an SHGW trust a client CA, you must upload the certificate to your APIM instance and then use the specific [`Gateway Certificate Authority - Create Or Update`](https://learn.microsoft.com/en-us/rest/api/apimanagement/gateway-certificate-authority/create-or-update?view=rest-apimanagement-2024-05-01&tabs=HTTP) API to explicitly link that certificate to the SHGW resource.

**📝 NOTE:** SHGW is designed to survive temporary disconnects from the control plane. It continues to operate using the last configuration it successfully downloaded (which is held in-memory). When local configuration backup is enabled, new pods can start using the saved local copy. Otherwise, any new pods that try to start during the outage cannot pull a configuration and will fail to initialize.

#### Check the thumbprint against certificates uploaded to API Management

Use `<when condition="$(...)">` policy with `<return-response>` and `<set-status code="403" reason="Invalid client certificate" />`:

```cs
// Client certificate : Hardcoded, not scalable
context.Request.Certificate == null || context.Request.Certificate.Thumbprint != "desired-thumbprint"

// Certificates uploaded to API Management : Verifying a managed list of pre-approved, trusted client certificates.
context.Request.Certificate == null || !context.Request.Certificate.Verify() || !context.Deployment.Certificates.Any(c => c.Value.Thumbprint == context.Request.Certificate.Thumbprint)

// Check the issuer and subject of a client certificate : You run your own CA and to trust all certs you issue.
context.Request.Certificate == null || context.Request.Certificate.Issuer != "trusted-issuer" || context.Request.Certificate.SubjectName.Name != "expected-subject-name"
```

## [Error handling](https://learn.microsoft.com/en-us/azure/api-management/api-management-error-handling-policies)

- Azure API Management uses a `ProxyError` object, accessed via `context.LastError`, for handling errors during request processing.
- Policies are divided into inbound, backend, outbound, and on-error sections. Processing jumps to the on-error section if an error occurs.
- If there's no on-error section, callers receive `400` or `500` HTTP response messages during an error.
- Predefined errors exist for built-in steps and policies, each with a source, condition, reason, and message.
- Custom behavior, like logging errors or creating new responses, can be configured in the on-error section.

## Versions and Revisions

- Use **Revisions** for non-breaking changes, allowing for testing and updates without affecting existing users. Users can access different revisions by using a different query string at the same endpoint. E.g. `https://apis.contoso.com/customers`**;rev=3**`/leads?customerId=123`
- Use **Versions** for breaking changes, requiring publishing and potentially requiring users to update their applications.

[Versioning schemes](https://learn.microsoft.com/en-us/azure/api-management/api-management-versions):

- Path-based versioning: `https://apis.contoso.com/products/v1` and `https://apis.contoso.com/products/v2`
- Header-based versioning: For example, custom header named `Api-Version,` and clients specify `v1` or `v2`
- Query string-based versioning: `https://apis.contoso.com/products?api-version=v1` and `https://apis.contoso.com/products?api-version=v2`

Use _Header-based versioning_ if the _URL has to stay the same_. Revisions and other types of versioning schemas require modified URL.

Creating separate gateways or web APIs would force users to access a different endpoint. A separate gateway provides complete isolation.

```sh
# Creating a release for an API inside an existing APIM
az apim api release create --resource-group $resourceGroup \
    --api-id demo-conference-api --api-revision 2 --service-name apim-hello-world \
    --notes 'Testing revisions. Added new "test" operation.'

# Deployment via ARM template
az group deployment create --resource-group $resourceGroup \
    --template-file ./apis.json \
    --parameters apiRevision="20191206" apiVersion="v1" serviceName=<serviceName> \
    apiVersionSetName=<versionSetName> apiName=<apiName> apiDisplayName=<displayName>
```

## [Integrating backend API with APIM](https://learn.microsoft.com/en-us/azure/api-management/import-and-publish)

Create OpenAPI documentation for the backend API, then import it into APIM. This enables integration with APIM and allows for automatic discovery of all endpoints. APIM becomes a facade for the backend API, providing customization without altering the backend API itself.

## [Monitoring](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor)

Azure API Management emits metrics every minute, providing near real-time visibility into the state and health of your APIs. The most frequently used metrics are 'Capacity' and 'Requests'. 'Capacity' tracks resource utilization to guide scaling, while 'Requests' measures API traffic.

## Working with API Management instance

Create new APIM:

```sh
az apim create --name MyAPIMInstance --resource-group $resourceGroup --location eastus --publisher-name "My Publisher" --publisher-email publisher@example.com --sku-name Developer
# or
New-AzApiManagement -ResourceGroupName RESOURCE_GROUP -Name NAME -Location LOCATION -Organization ORGANIZATION -AdminEmail ADMIN_EMAIL [-Sku SKU_NAME] [-Tags TAGS]
```

## [Policies reference](https://learn.microsoft.com/en-us/azure/api-management/api-management-policies)

**📝 NOTE:** The _rate-limit-by-key_ and _quota-by-key_ policies aren't available in the Consumption tier of Azure API Management.

| Name                                                                                                                                | Example                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sections                    |
| ----------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------- |
| Check HTTP header                                                                                                                   | `<check-header name="header name" failed-check-httpcode="code" failed-check-error-message="message" ignore-case="true \| false">`<br>&nbsp;&nbsp;`<value>Value1</value>`<br>&nbsp;&nbsp;`<value>Value2</value>`<br>`</check-header>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | The check passes if the header's value matches any one of the values in the list. Otherwise, the policy immediately terminates the request and sends a response to the caller with provided `failed-check-...` details.                                                                                                                                                                                                                                                                                                                                                                | inbound                     |
| Get authorization context                                                                                                           | `<get-authorization-context`<br>&nbsp;&nbsp;`provider-id="authorization provider id"`<br>&nbsp;&nbsp;`authorization-id="authorization id"`<br>&nbsp;&nbsp;`context-variable-name="variable name"`<br>&nbsp;&nbsp;`identity-type="managed \| jwt"`<br>&nbsp;&nbsp;`identity="JWT bearer token"`<br>&nbsp;&nbsp;`ignore-error="true \| false" />`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | `context-variable-name` is the name of the context variable to receive the Authorization object (`{accessToken: string, claims: Record<string, object>}`). Configure `identity-type=jwt` when the access policy for the authorization is assigned to a service principal. Only `/.default` scope (`app-only` token) is supported for the JWT.                                                                                                                                                                                                                                          | inbound                     |
| Restrict caller IPs                                                                                                                 | `<ip-filter action="allow \| forbid">`<br>&nbsp;&nbsp;`<address>address</address>`<br>&nbsp;&nbsp;`<address-range from="address" to="address" />`<br>`</ip-filter>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | At least one `address` or `address-range` element is required.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | inbound                     |
| Validate Microsoft Entra ID token                                                                                                   | `Simple token validation: <validate-azure-ad-token tenant-id="{{aad-tenant-id}}">`<br>&nbsp;&nbsp;`<client-application-ids>`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<application-id>{{aad-client-application-id}}</application-id>`<br>&nbsp;&nbsp;`</client-application-ids>`<br>`</validate-azure-ad-token>`<br>`Validate that audience and claim are correct: <validate-azure-ad-token tenant-id="{{aad-tenant-id}}" output-token-variable-name="jwt">`<br>&nbsp;&nbsp;`<client-application-ids>`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<application-id>{{aad-client-application-id}}</application-id>`<br>&nbsp;&nbsp;`</client-application-ids>`<br>&nbsp;&nbsp;`<audiences>`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<audience>@(context.Request.OriginalUrl.Host)</audience>`<br>&nbsp;&nbsp;`</audiences>`<br>&nbsp;&nbsp;`<required-claims>`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<claim name="ctry" match="any">`<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`<value>US</value>`<br>&nbsp;&nbsp;&nbsp;&nbsp;`</claim>`<br>&nbsp;&nbsp;`</required-claims>`<br>`</validate-azure-ad-token>`                                                        | To validate a JWT that was provided by another identity provider, use the generic `validate-jwt`. You can secure the whole API with Entra ID authentication by applying the policy on the API level, or you can apply it on the API operation level and use claims for more granular control.                                                                                                                                                                                                                                                                                          | inbound                     |
| Validate client certificate                                                                                                         | `<validate-client-certificate`<br>&nbsp;&nbsp;`validate-revocation="true \| false"`<br>&nbsp;&nbsp;`validate-trust="true \| false"`<br>&nbsp;&nbsp;`validate-not-before="true \| false"`<br>&nbsp;&nbsp;`validate-not-after="true \| false"`<br>&nbsp;&nbsp;`ignore-error="true \| false">`<br>&nbsp;&nbsp;`<identities>`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<identity `<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`thumbprint="certificate thumbprint"`<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`serial-number="certificate serial number"`<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`common-name="certificate common name"`<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`subject="certificate subject string"`<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`dns-name="certificate DNS name"`<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`issuer-subject="certificate issuer"`<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`issuer-thumbprint="certificate issuer thumbprint"`<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`issuer-certificate-id="certificate identifier" />`<br>&nbsp;&nbsp;`</identities>`<br>`</validate-client-certificate>` | All attributes are Optional (each has defaults), `identities` too                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | inbound                     |
| Control flow                                                                                                                        | `<choose>`<br>&nbsp;&nbsp;`<when condition="Boolean expression \| Boolean constant">`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<!— one or more policy statements to be applied if the above condition is true  -->`<br>&nbsp;&nbsp;`</when>`<br>&nbsp;&nbsp;`<when condition="Boolean expression \| Boolean constant">`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<!— one or more policy statements to be applied if the above condition is true  -->`<br>&nbsp;&nbsp;`</when>`<br>&nbsp;&nbsp;`<otherwise>`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<!— one or more policy statements to be applied if none of the above conditions are true  -->`<br>&nbsp;&nbsp;`</otherwise>`<br>`</choose>`                                                                                                                                                                                                                                                                                                                                                                                                                                                     | The choose policy must contain at least one `<when/>` element. The `<otherwise/>` element is optional.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Any                         |
| Limit concurrency                                                                                                                   | `<limit-concurrency key="expression" max-count="number">`<br>&nbsp;&nbsp;`<!— nested policy statements -->`<br>`</limit-concurrency>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Prevents enclosed policies from executing by more than the specified number of requests at any time. When that number is exceeded, new requests will fail immediately with `429 Too Many Requests`.<br>E.g. for given: `<limit-concurrency key="@((string)context.Variables["connectionId"])" max-count="10">`, requests with _connectionId="A"_ and _connectionId="B"_ => Total number of requests are allowed to be >=**20** at a certain point in time.                                                                                                                             | Any                         |
| Rate limit                                                                                                                          | `<rate-limit-by-key calls="number"`<br>&nbsp;&nbsp;`counter-key="key value" `<br>&nbsp;&nbsp;`renewal-period="seconds"`<br>`/>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Renewal period is in seconds. Raises `429 Too Many Requests`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | inbound                     |
| Quota                                                                                                                               | `<quota-by-key counter-key="key value" bandwidth="kilobytes" renewal-period="seconds" />`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Bandwidth is in KB, renewal period is in seconds                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | inbound                     |
| Emit custom metrics                                                                                                                 | `<emit-metric name="name of custom metric" value="value of custom metric" namespace="metric namespace">`<br>&nbsp;&nbsp;`<dimension name="dimension name" value="dimension value" />`<br>`</emit-metric>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sends custom metrics in the specified format to Application Insights. You can configure at most 10 custom dimensions for this policy. Counts toward the usage limits for custom metrics per region in a subscription.                                                                                                                                                                                                                                                                                                                                                                  | Any                         |
| Forward request                                                                                                                     | `<forward-request http-version="1 \| 2or1 \| 2" timeout="time in seconds" continue-timeout="time in seconds" follow-redirects="false \| true" buffer-request-body="false \| true" buffer-response="true \| false" fail-on-error-status-code="false \| true"/>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | By default, this policy is set at the global scope.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | backend                     |
| Log to event hub                                                                                                                    | `<log-to-eventhub logger-id="id of the logger entity" partition-id="index of the partition where messages are sent" partition-key="value used for partition assignment">`<br>`Expression returning a string to be logged`<br>`</log-to-eventhub>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | The policy is not affected by Application Insights sampling. All invocations of the policy will be logged. Max message size: 200 KB (otherwise truncated).                                                                                                                                                                                                                                                                                                                                                                                                                             | Any                         |
| Mock response                                                                                                                       | `<mock-response status-code="code" content-type="media type"/>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Cancels normal pipeline execution. It prioritizes response content examples, using schemas when available and generating sample responses (or no content is returned). Policy expressions can't be used in attribute values for this policy.                                                                                                                                                                                                                                                                                                                                           | inbound, outbound, on-error |
| Retry                                                                                                                               | `<retry`<br>&nbsp;&nbsp;`condition="Boolean expression or literal"`<br>&nbsp;&nbsp;`count="number of retry attempts"`<br>&nbsp;&nbsp;`interval="retry interval in seconds"`<br>&nbsp;&nbsp;`max-interval="maximum retry interval in seconds"`<br>&nbsp;&nbsp;`delta="retry interval delta in seconds"`<br>&nbsp;&nbsp;`first-fast-retry="boolean expression or literal">`<br>&nbsp;&nbsp;`<!-- One or more child policies. No restrictions. -->`<br>`</retry>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Executes its child policies once and then retries their execution until the `retry` condition becomes `false` or retry `count` is exhausted. When only the `interval` and `delta` are specified, the wait time between retries increases: `interval + (count - 1)*delta`.                                                                                                                                                                                                                                                                                                              | Any                         |
| Return response                                                                                                                     | `<return-response response-variable-name="existing context variable">`<br>&nbsp;&nbsp;`<set-status>...</set-status>`<br>&nbsp;&nbsp;`<set-header>...</set-header>`<br>&nbsp;&nbsp;`<set-body>...</set-body>`<br>`</return-response>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Pipeline cancelation, body removal, custom or default response return to caller. Context variable and policy statements modify response if both provided.                                                                                                                                                                                                                                                                                                                                                                                                                              | Any                         |
| [Send request](https://learn.microsoft.com/en-us/azure/api-management/send-request-policy)                                          | `<send-request mode="new \| copy" response-variable-name="" timeout="seconds" ignore-error="false \| true">`<br>&nbsp;&nbsp;`<set-url>request URL</set-url>`<br>&nbsp;&nbsp;`<set-method>...</set-method>`<br>&nbsp;&nbsp;`<set-header>...</set-header>`<br>&nbsp;&nbsp;`<set-body>...</set-body>`<br>&nbsp;&nbsp;`<authentication-certificate thumbprint="thumbprint" />`<br>&nbsp;&nbsp;`<proxy>...</proxy>`<br>`</send-request>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Default timeout: 60sec<br>In `set-method`, policy expressions aren't allowed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Inbound                     |
| Set HTTP proxy                                                                                                                      | `<proxy url="http://hostname-or-ip:port" username="username" password="password" />`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Only HTTP is supported between the gateway and the proxy. Basic and NTLM authentication only. `username` and `password` are not required.                                                                                                                                                                                                                                                                                                                                                                                                                                              | inbound                     |
| Set request method                                                                                                                  | `<set-method>HTTP method</set-method>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Policy expressions are allowed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | inbound, on-error           |
| Set Status Code                                                                                                                     | `<set-status code="HTTP status code" reason="description"/>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | -                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Any                         |
| Set Variable                                                                                                                        | `<set-variable name="variable name" value="Expression \| String literal" />`<br>`<set-variable name="IsMobile" value="@(context.Request.Headers.GetValueOrDefault("User-Agent","").Contains("iPad") \|\| context.Request.Headers.GetValueOrDefault("User-Agent","").Contains("iPhone"))" />`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | If the expression contains a literal it will be converted to a string.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Any                         |
| Authenticate with Basic                                                                                                             | `<authentication-basic username="username" password="password" />`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sets the HTTP Authorization header. Recommended using named values to provide credentials, with secrets protected in a key vault.                                                                                                                                                                                                                                                                                                                                                                                                                                                      | inbound                     |
| [Authenticate with managed identity](https://learn.microsoft.com/en-us/azure/api-management/authentication-managed-identity-policy) | `<authentication-managed-identity resource="resource" client-id="clientid of user-assigned identity" output-token-variable-name="token-variable" ignore-error="true\|false"/>`<br>`<authentication-managed-identity resource="AD_application_id" output-token-variable-name="msi-access-token" ignore-error="false" />`<br>`<!--Application (client) ID of your own Entra ID Application-->`<br>`<set-header name="Authorization" exists-action="override">`<br>&nbsp;&nbsp;`<value>@("Bearer " + (string)context.Variables["msi-access-token"])</value>`<br>`</set-header>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | After successfully obtaining the token, the policy will set the value of the token in the Authorization header using the Bearer scheme. Both system-assigned identity and any of the multiple user-assigned identities can be used to request a token.                                                                                                                                                                                                                                                                                                                                 | inbound                     |
| Get from cache                                                                                                                      | `<cache-lookup vary-by-developer="true \| false" vary-by-developer-groups="true \| false" caching-type="prefer-external \| external \| internal" downstream-caching-type="none \| private \| public" must-revalidate="true \| false" allow-private-response-caching="@(expression to evaluate)">`<br>&nbsp;&nbsp;`<vary-by-header>Accept</vary-by-header>`<br>&nbsp;&nbsp;`<vary-by-header>Accept-Charset</vary-by-header>`<br>&nbsp;&nbsp;`<vary-by-header>Authorization</vary-by-header>`<br>&nbsp;&nbsp;`<vary-by-header>header name</vary-by-header>`<br>&nbsp;&nbsp;`<vary-by-query-parameter>parameter name</vary-by-query-parameter>`<br>`</cache-lookup>`                                                                                                                                                                                                                                                                                                                                                                                                                                             | `vary-by-header` Add one or more of these elements to start caching responses per value of specified header, such as `Accept`, `Accept-Charset`, `Accept-Encoding`, `Accept-Language`, `Authorization`, `Expect`, `From`, `Host`, `If-Match`.<br>Similar behavior for `vary-by-...` attributes.                                                                                                                                                                                                                                                                                        | inbound                     |
| [Get value from cache](https://learn.microsoft.com/en-us/azure/api-management/cache-lookup-value-policy)                            | `<cache-lookup-value key="cache key value" default-value="value to use if cache lookup resulted in a miss" variable-name="name of a variable looked up value is assigned to" caching-type="prefer-external \| external \| internal" />`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | `caching-type`: `internal` to use the built-in API Management cache, `external` to use Redis.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Any                         |
| [Store to cache](https://learn.microsoft.com/en-us/azure/api-management/cache-store-value-policy)                                   | `<cache-store duration="seconds" cache-response="true \| false" />`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Use with `cache-lookup-value` in `inbound`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Any                         |
| Store value in cache                                                                                                                | `<cache-store-value key="cache key value" value="value to cache" duration="seconds" caching-type="prefer-external \| external \| internal" />`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | The operation is asynchronous. `caching-type`: `internal` to use the built-in API Management cache, `external` to use Redis.                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Any                         |
| ---                                                                                                                                 | ---                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | ---                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | ---                         |
| CORS                                                                                                                                | `<cors allow-credentials="false \| true" terminate-unmatched-request="true \| false">`<br>&nbsp;&nbsp;`<allowed-origins>`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<origin>origin uri</origin>`<br>&nbsp;&nbsp;`</allowed-origins>`<br>&nbsp;&nbsp;`<allowed-methods preflight-result-max-age="number of seconds">`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<method>HTTP verb</method>`<br>&nbsp;&nbsp;`</allowed-methods>`<br>&nbsp;&nbsp;`<allowed-headers>`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<header>header name</header>`<br>&nbsp;&nbsp;`</allowed-headers>`<br>&nbsp;&nbsp;`<expose-headers>`<br>&nbsp;&nbsp;&nbsp;&nbsp;`<header>header name</header>`<br>&nbsp;&nbsp;`</expose-headers>`<br>`</cors>`                                                                                                                                                                                                                                                                                                                                                                                                                               | Can be configured at any scope. The policy from the most specific scope is always applied.<br><br><b>For OPTIONS Preflight Requests:</b> When APIM receives a preflight OPTIONS request, it only evaluates the `cors` policy. All other policies in the inbound section are skipped.<br><br><b>For Actual Requests (e.g., GET, POST):</b> For the subsequent "actual" request (which follows a successful preflight), APIM skips the `cors` policy and executes all other inbound policies as configured.<br><br>You can only include one `cors` element within a single policy scope. | inbound                     |
| Find and replace string in body                                                                                                     | `<find-and-replace from="what to replace" to="replacement" />`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Policy expressions are allowed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Any                         |
| Set backend service                                                                                                                 | `<set-backend-service base-url="base URL of the backend service"  backend-id="name of the backend entity specifying base URL of the backend service" />`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Redirect an incoming request to a different backend than the one specified in the API settings for that operation.<br>`base-url` and `backend-id` are **mutual exclusive**.<br> Great for `choose`                                                                                                                                                                                                                                                                                                                                                                                     | inbound, backend            |
| [Set body](https://learn.microsoft.com/en-us/azure/api-management/set-body-policy)                                                  | `<set-body template="liquid" xsi-nil="blank \| null">new body value as text</set-body>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | `preserveContent` not needed when providing new body.<br> Inbound pipeline: no response yet, so no `preserveContent`.<br> Outbound pipeline: request already sent, so no `preserveContent`. <br>Exception if used in inbound GET with no body.                                                                                                                                                                                                                                                                                                                                         | inbound, outbound, backend  |
| Set header                                                                                                                          | `<set-header name="header name" exists-action="override \| skip \| append \| delete"><value>value</value></set-header>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | For multiple headers with the same name, add additional value elements.<br>The following headers can't be appended, overridden, or deleted: `Connection, Content-Length, Keep-Alive, Transfer-Encoding`                                                                                                                                                                                                                                                                                                                                                                                | Any                         |
| Rewrite URL                                                                                                                         | `<rewrite-uri template="/v2/US/hardware/{storenumber}&{ordernumber}?City=city&State=state" />`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Transforms human/browser-friendly URL into the URL format expected by the web service.<br>You can only add query string parameters using the policy. You can't add extra template path parameters in the rewrite URL.                                                                                                                                                                                                                                                                                                                                                                  | inbound                     |
| [Fire and forget](https://learn.microsoft.com/en-us/azure/api-management/send-one-way-request-policy)                               | `<send-one-way-request mode="new \| copy" timeout="time in seconds">`<br>`<set-url>request URL</set-url>`<br>`<set-method>...</set-method>`<br>`<set-header>...</set-header>`<br>`<set-body>...</set-body>`<br>`<authentication-certificate thumbprint="thumbprint" />`<br>`</send-one-way-request>`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sends an HTTP request but does not wait for a response, making it ideal for 'fire-and-forget' scenarios like logging without impacting client response time.                                                                                                                                                                                                                                                                                                                                                                                                                           | outbound                    |

**📝 NOTE:** As a best practice, include a base element at the beginning of each policy section to inherit policies from the parent scope. This ensures that any policies defined at a higher level are not inadvertently overridden or ignored.

**📝 NOTE:** APIM is designed to share a Redis cache without namespace collisions. To achieve this, it adds a prefix (based on the APIM instance ID) to all keys it writes. Therefore, a policy writing to key `user-123` will result in a key in Redis like `<apim-instance-id>_user-123`.

**📝 NOTE:** You CANNOT use Credential Manager to let APIM auto-sign client_assertions from an uploaded cert. You must manually implement the flow using a `send-request` policy to the token endpoint and sign the assertion yourself.
<br>

# AZ CLI

## Prerequisites

```sh
# Upgrade the Azure CLI to the latest version
az upgrade
```

### Extensions

```sh
 az extension add --name <name> --upgrade
```

- `containerapp`
- `storage-preview`

### Providers

```sh
# Only needed on subscriptions that haven't previously used it (takes some time for changes to propagate)
az provider register --namespace <name>

# Check status
az provider show --namespace <name> --query "registrationState"
```

- `Microsoft.App` (App Services - hosting APIs)
- `Microsoft.EventGrid`
- `Microsoft.CDN`
- `Microsoft.OperationalInsights` (telemetry)
- `Microsoft.OperationsManagement`

# [Azure App Configuration](https://learn.microsoft.com/en-us/azure/azure-app-configuration/)

Azure App Configuration provides a centralized way to manage application settings and feature flags, streamlining configuration across environments, regions, and deployment stages. It supports hierarchical keys and enables real-time updates without restarting the application.

Although it encrypts stored data, Azure App Configuration lacks the advanced security capabilities of Azure Key Vault—such as hardware-backed encryption, fine-grained access policies, and secret lifecycle management, which includes automatic secret rotation, expiration handling, and versioning with audit trails.

Configurations can be imported from or exported to external files, and separate configuration stores can be maintained for different environments like development, testing, and production.

## Azure App Configuration Overview

Azure App Configuration manages configuration data using key-value pairs.

### Azure App Configuration Reserved Characters

| Reserved Characters in Key Names                             | Reserved Characters in Queries (Filters)                                                        | Examples of Queries                                                                                        |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `%` - Cannot use the percent sign                            | `*` (Asterisk) - Wildcard to match one or more characters                                       | `App:Settings:*` matches all keys starting with `App:Settings:`                                            |
| `.` or `..` - Key names cannot be a single dot or double dot | `,` (Comma) - Separator to query for multiple keys or labels at once                            | `Key1,Key2` retrieves both `Key1` and `Key2`                                                               |
|                                                              | `\` (Backslash) - Escape character to search for keys that literally contain special characters | To find `MyKey,Special` use filter `MyKey\,Special`<br>To find `MyKey*Special` use filter `MyKey\*Special` |

#### **Keys**

- Keys are unique, case-sensitive identifiers.
- Use delimiters like / or : to create a logical hierarchy (e.g., AppName:Service1:ApiEndpoint), but note:
- Azure treats each key as a flat string—hierarchy is only a naming convention, not enforced.

#### **Labels**

- Creates different versions of the same key. The most common use is for environments (like "Dev", "Test", "Prod") or software versions (like "v1.0", "v1.1").

- A key's name _plus_ its label makes it unique. This lets you store different values for the same key.

| Key                    | Label | Value                         |
| ---------------------- | ----- | ----------------------------- |
| Api:DatabaseConnection | Test  | test-server-connection-string |
| Api:DatabaseConnection | Prod  | prod-server-connection-string |

These are treated as two completely separate settings.

- A setting with no label is called the **"null" label**. This is perfect for default values that all environments share. In a query, null labels can be found by searching for `\0`.
- Labels are hierachical logical-wise only, same approach as keys.

#### **Values**

- Values are Unicode strings, and can optionally include a content type to describe the data (e.g., application/json).
- This content type acts as metadata, aiding tools or services that consume the values.

### Configuration and Querying

```cs
// Load configuration
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(connectionString)
           // Select a subset of the configuration keys (start with `TestApp:` and have no label) from Azure App Configuration
           // To select all keys: KeyFilter.Any
           .Select("TestApp:*", LabelFilter.Null)
           // Configure to reload configuration if the registered sentinel key is modified
           .ConfigureRefresh(refreshOptions => refreshOptions.Register("TestApp:Settings:Sentinel", refreshAll: true));
});
...

// Query keys
AsyncPageable<ConfigurationSetting> settings = client.GetConfigurationSettingsAsync(new SettingSelector { KeyFilter = "AppName:*" });
await foreach (ConfigurationSetting setting in settings)
    Console.WriteLine($"Key: {setting.Key}, Value: {setting.Value}");
```

#### Chaining

When using multiple `.Select()`, if a key with the same name exists in both labels. the value from the last `.Select()` will be used.

```cs
// In this example, if a key like 'TestApp:Key' exists both with the 'dev' label and with no label,
// the value associated with the 'dev' label will be used. This is due to the order of the .Select() calls.
// If you reverse the order of the .Select() calls, the value associated with no label would take precedence.
.Select("TestApp:*", LabelFilter.Null)
.Select("TestApp:*", "dev");
```

## [Feature Management](https://learn.microsoft.com/en-us/azure/azure-app-configuration/howto-feature-filters)

- **Feature flag**: A binary variable (on/off) that controls the execution of an associated code block.
- **Feature manager**: A software package managing feature flags' lifecycle, providing additional functions like caching and updating flag states.
- **Filter**: A rule determining the state of a feature flag, based on user groups, device types, geographic location, or time windows.

A successful feature management system requires:

- An application using feature flags.
- A separate centralized, external storage service (or `Azure App Configuration` service) that holds all your feature flags and their current states (e.g., on, off, or specific targeting rules).

### Using Feature Flags in Code

Feature flags are used as Boolean state variables in conditional statements:

```cs
bool featureFlag = true; // static value
bool featureFlag = isBetaUser(); // evaluated value

if (featureFlag) { /* ... */ }
else { /* ... */ }
```

#### [Configure feature flags](https://learn.microsoft.com/en-us/azure/azure-app-configuration/quickstart-feature-flag-aspnet-core)

```cs
// Load configuration from Azure App Configuration
builder.Configuration.AddAzureAppConfiguration(options =>
{
    // By default if no parameter is passed (options.UseFeatureFlags()), it loads all feature flags with no label
    // The default refresh expiration of feature flags is 30 seconds
    options.UseFeatureFlags(featureFlagOptions =>
    {
        // Select feature flags from "TestApp" namespace, with label "dev"
        featureFlagOptions.Select("TestApp:*", "dev");
        // Cache the feature flag values for 5 minutes before checking Azure App Configuration again
        featureFlagOptions.CacheExpirationInterval = TimeSpan.FromMinutes(5);
    });
});

// Add feature management to the container of services.
builder.Services.AddFeatureManagement();
// below is explained later
builder.Services.AddFeatureFilter<TargetingFilter>();
```

### [Conditional feature flags](https://learn.microsoft.com/en-us/azure/azure-app-configuration/howto-feature-filters-aspnet-core)

Allows the feature flag to be enabled or disabled dynamically.

- `PercentageFilter` enables the feature for a certain percentage of requests or users.

  - Example: Roll out a new search algorithm to 10% of users, then gradually increase.
  - Often used for canary releases or A/B testing.

- `TimeWindowFilter` enables the feature only within a specific time range.

  - Example: Turn on a holiday banner from December 1st to December 31st automatically.
  - Useful for timed promotions or feature launches.

- `TargetingFilter` enables the feature for specific users or groups based on IDs, usernames, or claims (like an email domain).
  - Example: Show a beta feature only to internal testers or VIP customers.

#### [Enable staged rollout of features for targeted audiences with `TargetingFilter`](https://learn.microsoft.com/en-us/azure/azure-app-configuration/howto-targetingfilter-aspnet-core)

1. Implement `ITargetingContextAccessor`

```cs
public class HttpContextTargetingContextAccessor : ITargetingContextAccessor
{
    private const string TargetingContextLookup = "TestTargetingContextAccessor.TargetingContext";
    private readonly IHttpContextAccessor _httpContextAccessor;

    public HttpContextTargetingContextAccessor(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public ValueTask<TargetingContext> GetContextAsync()
    {
        if (httpContext.Items.TryGetValue(TargetingContextLookup, out object value))
            return new ValueTask<TargetingContext>((TargetingContext)value);

        // Example: `test@contoso.com` - User: `test`, Group(s): `contoso.com`
        List<string> groups = new List<string>();
        if (httpContext.User.Identity.Name != null)
            groups.Add(httpContext.User.Identity.Name.Split("@", StringSplitOptions.None)[1]);

        var targetingContext = new TargetingContext
        {
            UserId = httpContext.User.Identity.Name,
            Groups = groups
        };
        httpContext.Items[TargetingContextLookup] = targetingContext;
        return new ValueTask<TargetingContext>(targetingContext);
    }
}
```

1. Add `TargetingFilter`: `services.AddFeatureManagement().AddFeatureFilter<TargetingFilter>();`

1. Update the `ConfigureServices` method to add the `ITargetingContextAccessor` implementation: `services.AddSingleton<ITargetingContextAccessor, HttpContextTargetingContextAccessor>();`

1. `app > Feature Manager > (create feature flag) > (enable) > Edit > Use feature filter > Targeting filter >  Override by Groups and Override by Users`

### Declaring Feature Flags

A feature flag consists of a name and one or more filters determining if the feature is on (`true`). When multiple filters are used, they are evaluated in order until one enables the feature. If none do, the feature is off.

The feature manager supports _appsettings.json_ as a configuration source for feature flags:

```jsonc
{
  "FeatureManagement": {
    "FeatureA": true, // Feature on
    "FeatureB": false, // Feature off
    "FeatureC": {
      // Conditional feature flag. It uses a feature filter called Percentage to enable the feature for a random 50% of users.
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": { "Value": 50 }
        }
      ]
    }
  }
}
```

### Feature Flag Repository

Azure App Configuration serves as a centralized repository for feature flags, enabling externalizing all feature flags used in an application. This allows changing feature states without modifying and redeploying the application, no stop/start required - thanks to `Polling mechanism + Cache refresh`.

<br>

#### **Built-in Feature Filters**

| Filter                      | Purpose                                        | Auto-Added                      | Key Properties                                                                     |
| --------------------------- | ---------------------------------------------- | ------------------------------- | ---------------------------------------------------------------------------------- |
| `Microsoft.Percentage`      | Enable feature for % of users                  | Yes                             | `Value` (0-100)                                                                    |
| `Microsoft.TimeWindow`      | Enable during time window                      | Yes                             | `Start`, `End`, `Recurrence` (optional)                                            |
| `Microsoft.Targeting`       | Enable for specific users/groups (in web apps) | No (use `AddTargetingFilter()`) | `Audience` (Users, Groups, DefaultRolloutPercentage, Exclusion)                    |
| `ContextualTargetingFilter` | Targeting for console apps / non-web scenarios | Yes                             | Same as Targeting, but you must pass the `TargetingContext` manually in your code. |

#### **Recurrence Pattern Types**

| Pattern | Key Properties                                                                                            |
| ------- | --------------------------------------------------------------------------------------------------------- |
| Daily   | `Type: "Daily"`, `Interval` (optional, days between occurrences)                                          |
| Weekly  | `Type: "Weekly"`, `DaysOfWeek` (array), `Interval` (optional, weeks between), `FirstDayOfWeek` (optional) |

#### **Recurrence Range Types**

| Range    | Description         | Key Properties                            |
| -------- | ------------------- | ----------------------------------------- |
| NoEnd    | Repeat indefinitely | `Type: "NoEnd"`                           |
| EndDate  | Repeat until date   | `Type: "EndDate"`, `EndDate`              |
| Numbered | Repeat N times      | `Type: "Numbered"`, `NumberOfOccurrences` |

#### **Targeting Setup**

| Application Type | Implementation                                                     | Registration                                |
| ---------------- | ------------------------------------------------------------------ | ------------------------------------------- |
| Web App          | Use `ITargetingContextAccessor`                                    | `.WithTargeting()` or `.WithTargeting<T>()` |
| Console App      | Use `ContextualTargetingFilter` + pass `TargetingContext` manually | Standard feature management setup           |

#### **Targeting Evaluation Order**

Evaluation stops as soon as the first matching rule is found.

| Priority | Rule                | Condition                                                   | Result                                                     | Notes                                                                                              |
| -------- | ------------------- | ----------------------------------------------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| 1        | **Exclusion**       | User or any of his group is in `Exclusion` list             | **Disabled**                                               | Always checked first. Immediately disables the feature regardless of other rules.                  |
| 2        | **Users**           | User is in `Users` list                                     | **Enabled**                                                | Direct user targeting.                                                                             |
| 3        | **Groups**          | User is in any defined `Group`                              | **Enabled/Disabled** (based on group's rollout %)          | Uses percentage-based rollout per group. User may or may not be `Enabled` based on the percentage. |
| 4        | **Default Rollout** | User not in `Exclusion`, `Users`, or `Groups`               | **Enabled/Disabled** (based on `DefaultRolloutPercentage`) | Uses percentage-based enablement.                                                                  |
| 5        | **None Match**      | DefaultRolloutPercentage = 0% OR user falls outside rollout | **Disabled**                                               | Final fallback                                                                                     |

#### **[Variants Overview](https://learn.microsoft.com/en-us/azure/azure-app-configuration/feature-management-dotnet-reference#variants)**

| Concept            | Description                                                     |
| ------------------ | --------------------------------------------------------------- |
| Purpose            | A/B testing - multiple feature configurations                   |
| Variant Properties | `configuration_value` (string, number, bool, or object), `name` |
| Retrieve           | `await featureManager.GetVariantAsync("FeatureName")`           |

#### **[Variant Allocation Properties](https://learn.microsoft.com/en-us/azure/azure-app-configuration/feature-management-dotnet-reference#allocate-variants)**

| Property                | Description                                                                                                                                                                                |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `default_when_disabled` | Takes effect if the flag is disabled, done by setting the `Enabled` field to false, also known as "kill switch".                                                                           |
| `default_when_enabled`  | Takes effect if the flag is enabled but the allocation doesn't assign all percentiles. Any user placed in an unassigned percentile receives the `DefaultWhenEnabled` variant.              |
| `user`                  | Assign variant to specific users                                                                                                                                                           |
| `group`                 | Assign variant if user in group(s)                                                                                                                                                         |
| `percentile`            | Assign variant if user in % range (e.g., 0-10)                                                                                                                                             |
| `seed`                  | Without a seed, a user might see VariantA today and VariantB tomorrow. The seed makes the random assignment predictable.<br>Its value is trivial, better to give a self-descriptive value. |

#### **[Status Override (Variants)](https://learn.microsoft.com/en-us/azure/azure-app-configuration/feature-management-dotnet-reference#override-enabled-state-by-using-a-variant)**

During the call to IsEnabledAsync on a flag with variants, the feature manager checks whether the variant assigned to the user is configured to override the result.

| Value      | Effect                                             |
| ---------- | -------------------------------------------------- |
| `None`     | No override (default)                              |
| `Enabled`  | Evaluate feature as `enabled` when variant chosen  |
| `Disabled` | Evaluate feature as `disabled` when variant chosen |

#### **[Variant Service Provider](https://learn.microsoft.com/en-us/azure/azure-app-configuration/feature-management-dotnet-reference#variant-service-alias-attribute)**

| Component                    | Purpose                                                              |
| ---------------------------- | -------------------------------------------------------------------- |
| `IVariantServiceProvider<T>` | Get different service implementations per user based on variant      |
| Setup                        | `.WithVariantService<T>("FeatureName")`                              |
| Attribute                    | `[VariantServiceAlias("Name")]` - map implementation to variant name |

## Security

### [Using Customer-Managed Keys for Encryption](https://learn.microsoft.com/en-us/azure/azure-app-configuration/concept-customer-managed-keys)

A managed identity authenticates with Microsoft Entra ID and wraps the encryption key using Azure Key Vault. The wrapped key is stored and the unwrapped key is cached for an hour, then refreshed.

Prerequisites:

- _A Standard+ tier_ Azure App Configuration
- Azure Key Vault with enabled soft-delete and purge-protection
- An unexpired, enabled RSA or RSA-HSM key in the Key Vault with `wrap` and `unwrap` capabilities

After setup, assign a managed identity to the App Configuration and grant it `GET`, `WRAP`, and `UNWRAP` (permits decrypting previously wrapped keys) permissions in the Key Vault's access policy:

```bash
az keyvault set-policy --name <YourKeyVaultName> \
                       --object-id <ManagedIdentityObjectID> \
                       --key-permissions get wrapKey unwrapKey
```

**📝 NOTE:** To mitigate the possibility of the underlying managed key expiring, do both:

- Omit the key version when setting up encryption
- Enable _auto key rotation_ in Key Vault.

**📝 NOTE:** Sensitive information can't be decrypted if any of the following occurs:

- The store's identity loses permission to the encryption key
- The managed key is permanently deleted
- The managed key version expires

## [Private endpoint](https://learn.microsoft.com/en-us/azure/azure-app-configuration/concept-private-endpoint)

- Enables secure access to Azure App Configuration over a private link using an IP address from the VNet address space.
- Traffic stays on the Microsoft backbone network, preventing exposure to the public internet.
- Blocks public network access by default; can be re-enabled by following:<br>

```bash
  # To enable requests coming from public networks while private endpoint is enabled. When false, only requests made through Private Links are allowed.
  az appconfig update \
    --resource-group <resource-group-name> \
    --name <App-Configuration-store-name> \
    --enable-public-network true
```

- Uses same connection strings/auth; no app changes needed.

## Configure Key Vault

```cs
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(
        builder.Configuration["ConnectionStrings:AppConfig"])
            .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential()));
});
```

After the initialization, you can access the values of Key Vault references in the same way you access the values of regular App Configuration keys.

## Import / Export configuration

```bash
# Import all keys and feature flags from a file and apply "Test" label
az appconfig kv import \
  -n MyAppConfiguration \
  --label test \
  -s file \
  --path D:/abc.json \
  --format json

# Export all keys and feature flags with label "Test" to a json file
az appconfig kv export \
  -n MyAppConfiguration \
  --label test \
  -d file \
  --path D:/abc.json \
  --format json
```

## CLI

- [az appconfig kv import](https://learn.microsoft.com/en-us/cli/azure/appconfig/kv?view=azure-cli-latest#az-appconfig-kv-import)
- [az appconfig kv export](https://learn.microsoft.com/en-us/cli/azure/appconfig/kv?view=azure-cli-latest#az-appconfig-kv-export)
- [az appconfig identity](https://learn.microsoft.com/en-us/cli/azure/appconfig/identity?view=azure-cli-latest)

# [Azure App Service](https://docs.microsoft.com/en-us/azure/app-service/)

## [Azure App Service plans](https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans)

Tier Progression: Free → Shared → Basic → Standard → PremiumV(n) → IsolatedV(n)
### Features

- [Multi-tenant workers](https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans#:~:text=Description-,Shared%20compute,-Free%2C%20Shared): Free and Shared
- Custom DNS Name: Shared+
- [Dedicated workers](https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans#:~:text=and%20testing%20purposes.-,Dedicated%20compute,-Basic%2C%20Standard%2C%20Premium): Basic+
- Scalability (scaling out): Basic+
- TLS/SSL (HTTPS): Basic+
- Always On: Basic+
- Free managed certificate: Basic+
- Autoscale: Standard+
- Staging environments (deployment slots): Standard+
- Linux: Basic+ (Free tier supports for limited resource only)
- AppServiceFileAuditLogs: Premium+
- AppServiceAntivirusScanAuditLogs: Premium+
- Automatic scaling: PremiumV2+
- Single-tenant setup (App Service Environment - ASE): Isolated+
- [Virtual Networks (VNet Integration)](https://learn.microsoft.com/en-us/azure/app-service/overview-vnet-integration#:~:text=inbound%20private%20access.-,The%20virtual%20network%20integration%20feature,-%3A): Basic+ 
- Full Network Isolation: Isolated+
- Maximum scale-out: IsolatedV2+

The roles that handle incoming HTTP or HTTPS requests are called _front ends_. The roles that host the customer workload are called _workers_.

### Tiers

- **Shared compute**: **Free** (F1) and **Shared** (D1) tiers run apps on the same Azure VM with other customer apps, [sharing resources and limited CPU quotas](https://docs.azure.cn/en-us/azure-resource-manager/management/azure-subscription-service-limits#azure-app-service-limits:~:text=is%20999%20GB.-,CPU%20time,-(5%20minutes)). ⭐: _development_ and _testing_ only. The Free tier is $0, and the Shared tier is a fixed monthly cost per app that includes a limited CPU quota.

- **Dedicated compute**: **Basic** ⏺️ (B1), **Standard** (S1), **Premium** (P1V1), **PremiumV2** (P1V2), and **PremiumV3** (P1V3) tiers utilize dedicated Azure VMs. Apps within the same App Service plan share compute resources. Higher tiers provide more powerful VM instances and allow you to scale out to a higher maximum number of instances. Scaling out (_autoscale_) simply adds another VM running the same applications and services. You're charged based on the App Service Plan tier and the number of instances.

- **Isolated** (I1): The **Isolated** and **IsolatedV2** tiers run dedicated Azure VMs on _dedicated Azure Virtual Networks_. It provides network isolation on top of compute isolation to your apps. IsolatedV2 provides the _maximum scale-out capabilities_. Charging is [mostly](https://techcommunity.microsoft.com/blog/appsonazureblog/asev3-vs-asev2---difference-overview/3276225#:~:text=Total(ASPs)%3A200-,Pricing,-Pay%20for%20each) based on the _number of isolated workers that run your apps_.

**📝 NOTE:** App Service plans that have no apps associated with them still incur charges because they continue to reserve the configured VM instances.

All apps, deployment slots, WebJobs, backups, and diagnostic logs in an App Service plan share the _same VM instances_.

When to isolate an app into a new App Service plan:

- The app is resource-intensive.
- You want to scale the app independently from the other apps in the existing plan.
- The app needs resources in a different geographical region.

#### [Move App To Another Plan](https://learn.microsoft.com/en-us/azure/app-service/app-service-plan-manage#move-an-app-to-another-app-service-plan)

Source plan and destination plan must be in the same resource group, geographical region, same OS type, and supports the currently used features.
This can be done on Portal or [Powershell](https://docs.azure.cn/en-us/app-service/app-service-web-app-cloning#clone-an-existing-app), [it is not supported via CLI](https://learn.microsoft.com/en-us/answers/questions/659842/moving-an-azure-app-service-to-another-existing-ap#:~:text=Unfortunately%2C%20we%20can%27t%20change%20the%20App%20Service%20Plan%20through%20CLI%2C%20otherwise%2C%20it%20would%20have%20shown%20more%20information.).

If you attempt to [move](https://learn.microsoft.com/en-us/azure/app-service/app-service-plan-manage#:~:text=Important-,If%20you%20move%20an%20app,-from%20a%20higher) an app from a higher-tiered plan to a lower-tiered plan, e.g. from D1 to F1, it might fail due to unsupported features in the target plan.

**📝 NOTE:** For [moving to a different region](https://learn.microsoft.com/en-us/azure/app-service/app-service-plan-manage#move-an-app-to-a-different-region), you'd need to clone to a new app (can't directly move).

### [Scaling](https://learn.microsoft.com/en-us/azure/app-service/manage-automatic-scaling?tabs=azure-portal)

Settings affect all apps in your App Service plan.

#### Azure App Service Scaling Modes

| Scaling Mode | Description | Typical Tiers |
|--------------|-------------|---------------|
| **Manual** | You explicitly set the number of instances (VMs). Azure won't change it unless you do it yourself. | Basic+ |
| **Autoscale (Rule-based)** | Azure automatically scales out/in based on rules you define (e.g., CPU > 70% for 10 mins). | Standard+ |
| **Automatic (Platform-managed)** | Azure completely manages scaling for you — based on real-time HTTP traffic — no user-defined rules needed. | Premium V2+ |


  ```sh
  az appservice plan update --name $appServicePlanName --resource-group $resourceGroup \
      # enables automatic scaling
      --elastic-scale true --max-elastic-worker-count <YOUR_MAX_BURST> \
      # disable automatic scaling
      --elastic-scale false
  ```

Horizontal scaling: Adding/removing virtual machines.

- **Scale out** (increase VM instances): If _any_ of the rules are met
- **Scale in** (decrease VM instances): If _all_ rules are met

Vertical scaling: Scale up/down - when changing app service plan

[**Flapping**](https://learn.microsoft.com/en-us/azure/architecture/best-practices/auto-scaling?WT.mc_id=AZ-MVP-5004334#:~:text=Avoid%20flapping%20where,scale%2Din%20thresholds.): where scale-in and scale-out actions continually go back and forth.

### Continuous integration/deployment

Built-in CI/CD with Git (Azure DevOps, third-party, local), FTP, and container registries (ACR, third-party).

## [Deployment slots](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots)

All of the slots for a web app share the same App Service Plan and can share several configuration. However, [a slot can be considered as a separate VM instance within an app service plan.](https://learn.microsoft.com/en-us/answers/questions/19042/azure-app-service-shared-storage-between-deploymen#:~:text=A%20slot%20can%20be%20considered%20as%20a%20separate%20VM%20instance%20within%20your%20app%20service%20plan.) They have different host names (`my-app-staging.azurewebsites.net` vs `my-app.azurewebsites.net`).

[Best practices](https://learn.microsoft.com/en-us/azure/app-service/deploy-best-practices): Deploy to staging, then swap slots to warm up instances and eliminate downtime.

- **Swapped**: Settings that define the application's _behavior_. Includes connection strings, authentication settings, public certificates, path mappings, CDN, hybrid connections.
- **Not Swapped**: Settings that define the application's _environment and security_. They are less about the application itself and more about how it interacts with the external world. Examples: Private certificates, managed identities, publishing endpoints, diagnostic logs settings, CORS.

| [Settings that are swappable](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots?utm_source=chatgpt.com&tabs=cli#swap-with-preview-multiple-phase-swap:~:text=When%20you%20swap%20slots%2C%20these%20settings%20are%20swapped)                      | [Settings that are not swappable](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots?utm_source=chatgpt.com&tabs=cli#swap-with-preview-multiple-phase-swap:~:text=When%20you%20swap%20slots%2C%20these%20settings%20aren%27t%20swapped%3A)                            |
| ---------------------------------------------- | ------------------------------------------------------- |
| General settings: framework, arch, web sockets | Publishing endpoints                                    |
| App settings: authentication (can be disabled) | Custom domain names                                     |
| Connection strings (can be disabled)           | Non-public certificates and TLS/SSL settings            |
| Handler mappings                               | Scale settings                                          |
| Public certificates                            | WebJobs schedulers                                      |
| WebJobs content                                | IP restrictions                                         |
| Hybrid connections                             | Always On                                               |
| Azure Content Delivery Network                 | Diagnostic log settings                                 |
| Service endpoints                              | CORS                    |
| Path mappings                                  | Virtual network integration                             |
|                                                | Managed identities                                      |
|                                                | Settings that end with the suffix `\_EXTENSION_VERSION` |

**📝 NOTE:** `WEBSITE_OVERRIDE_PRESERVE_DEFAULT_STICKY_SLOT_SETTINGS`: Setting this in every slot of the app to “0” or “false” will change behavior of the settings listed above in "not swappable" to "swappable" , with the exceptions below:
- Managed Identities are never swapped.
- Those explicitly marked as `Deployment slot` can not be overridden.

**📝 NOTE:** **Hybrid Connections**: Lets your Azure App talk to your local server securely without changing firewall settings.

### [Route production traffic manually](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots?tabs=portal#route-production-traffic-manually)

`x-ms-routing-name=`: `self` for production slot, `staging` for staging slot.

Example: `<a href="<webappname>.azurewebsites.net/?x-ms-routing-name=self">Go back to production app</a> | <a href="<webappname>.azurewebsites.net/?x-ms-routing-name=staging">Go back to staging app</a>`

⏺️: All requests go to production. Traffic can be split; new slots start at 0% (no random transfers to other slots).

## [Configuration](https://learn.microsoft.com/en-us/azure/app-service/configure-common?tabs=cli)

App settings are always encrypted when stored (encrypted-at-rest).

For Linux apps and custom containers, App Service passes app settings to the container using the `--env` flag to set the environment variable in the container.

App Settings and Connection Strings are set at app startup and **trigger a restart when changed**. They override settings in `Web.config` or `appsettings.json`.

- **Always On**: Keeps app loaded; off by default. When off, app unloads after 20 mins of inactivity. Needed for Application Insights Profiler, continuous WebJobs or WebJobs triggered by a CRON expression.
- **ARR affinity (`sticky sessions`)**: In a multi-instance deployment (scaled out to run on multiple VM instances), ensure that the client is routed to the same instance for the life of the session. Disabling ARR affinity often improves performance in stateless apps.

### App Settings

Configuration data is hierarchical (settings can have sub-settings). In Linux, nested setting name like `ApplicationInsights:InstrumentationKey` needs to be configured as `ApplicationInsights__InstrumentationKey` Dots (`.`) will be replaced with `_`.

```cs
// az webapp config appsettings set --settings MySetting="<value>" MyParentSetting__MySubsetting="<value>" ...
string mySettingValue = Configuration["MySetting"];
string myParentSettingValue = Configuration["MyParentSetting:MySubSetting"]; // same as "MyParentSetting__MySubSetting"
```

```jsonc
// Save: az webapp config appsettings list --name $appName --resource-group $resourceGroup > settings.json

// settings.json
[
  { "name": "key1", "value": "value1", "slotSetting": false },
  { "name": "key2", "value": "value2" }
  // ...
]

// Load: az webapp config appsettings set --resource-group $resourceGroup --name $appName --settings @settings.json
```

### [Source app settings](https://learn.microsoft.com/en-us/azure/app-service/app-service-key-vault-references?tabs=azure-cli#source-app-settings-from-key-vault)

Key vault: Prerequisites: Grant your app access to a key vault to a managed identity. Instead of storing "my-secret-value", you store `@Microsoft.KeyVault(...)` which tells Azure to fetch the real value from Key Vault at runtime:

- `@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/mysecret/)`
- `@Microsoft.KeyVault(VaultName=myvault;SecretName=mysecret)`

[App Configuration](https://learn.microsoft.com/en-us/azure/app-service/app-service-configuration-references#reference-syntax): `@Microsoft.AppConfiguration(Endpoint=https://myAppConfigStore.azconfig.io; Key=myAppConfigKey; Label=myKeysLabel)`

### [Connection strings](https://learn.microsoft.com/en-us/azure/app-service/configure-common?tabs=portal#:~:text=At%20runtime%2C%20connection%20strings%20are%20available%20as%20environment%20variables%2C%20prefixed%20with%20the%20following%20connection%20types%3A)

Connection strings are prefixed with connection type at runtime. Similar to how they're set in the `Web.config` under `<connectionStrings>`.

```cs
// az webapp config connection-string set --connection-string-type SQLServer --settings MyDb="Server=myserver;Database=mydb;User Id=myuser;Password=mypassword;" ...
string myConnectionString = Configuration.GetConnectionString("MyDb");
string myConnectionStringEnv = Environment.GetEnvironmentVariable("SQLCONNSTR_MyDb"); // Same as above
```

```jsonc
// Save: az webapp config connection-string list --name $appName --resource-group $resourceGroup > conn-settings.json

// conn-settings.json
[
  {
    "name": "name-1",
    "value": "conn-string-1",
    "type": "SQLServer",
    "slotSetting": false
  },
  {
    "name": "name-2",
    "value": "conn-string-2",
    "type": "PostgreSQL"
  }
  // ...
]

// Load: az webapp config connection-string set --resource-group $resourceGroup --name $appName --settings @conn-settings.json
```

### [General Settings](https://learn.microsoft.com/en-us/cli/azure/webapp/config?view=azure-cli-latest#az-webapp-config-set)

```sh
az webapp config set --name $appName --resource-group $resourceGroup \
    --use-32bit-worker-process [true|false] \
    --web-sockets-enabled [true|false] \
    --always-on [true|false] \
    --http20-enabled \
    --auto-heal-enabled [true|false] \
    --remote-debugging-enabled [true|false] \ # automatically turned off after 48 hours, in case forgotten enabled
    --number-of-workers

az webapp config appsettings set --name $appName --resource-group $resourceGroup /
    --settings \
        WEBSITES_PORT=8000 \
        PRE_BUILD_COMMAND="echo foo && scripts/prebuild.sh" \
        POST_BUILD_COMMAND="echo foo && scripts/postbuild.sh" \
        PROJECT="<project-name>/<project-name>.csproj" \ # Deploy multi-project solutions
        ASPNETCORE_ENVIRONMENT="Development" \
        # Custom environment variables
        DB_HOST="myownserver.mysql.database.azure.com"
```
**📝 NOTE:** Updating or removing application settings will cause an app recycle. [Also, cleaner syntax is possible using `.json`](https://learn.microsoft.com/en-us/cli/azure/webapp/config/appsettings?view=azure-cli-latest#az-webapp-config-appsettings-set:~:text=Resource%20Id%20Arguments-,%2D%2Dsettings,more%20information%20on%20file%20format%20and%20editing%20app%20settings%20in%20bulk.,-%2D%2Dslot%20%2Ds).

**📝 NOTE:** `https://<app-name>.scm.azurewebsites.net/Env` displays all environment variables configured in your Azure App Service, allowing you to verify that settings like connection strings and app configurations are correctly set and accessible to your application. Configuration settings are **excluded**.

### [Handler Mappings](https://learn.microsoft.com/en-us/azure/app-service/configure-common?tabs=portal#configure-handler-mappings)

Lets you add custom script processors to handle requests for specific file extensions.

- **Extension**: The file extension you want to handle, such as _\*.php_ or _handler.fcgi_.
- **Script processor**: The absolute path of the script processor. Requests to files that match the file extension are processed by the script processor. Use the path `D:\home\site\wwwroot` to refer to your app's root directory.
- **Arguments**: Optional command-line arguments for the script processor.

### [Map a URL path to a directory](https://learn.microsoft.com/en-us/azure/app-service/configure-common?tabs=cli#map-a-url-path-to-a-directory)
Lets you control which folder serves which URL path - useful for monorepos, static sites, or custom folder structures.
```jsonc
// json.txt
[
  {
    "physicalPath"':' "site\\wwwroot\\public", // serve app from /public instead of root (site\\wwwroot)
    "preloadEnabled"':' false,
    "virtualDirectories"':' null,
    "virtualPath"':' "/" // any path can be mapped
  }
]
```
```bash
az resource update --resource-group <group-name> --resource-type Microsoft.Web/sites/config --name <app-name>/config/web --set properties.virtualApplications="@json.txt"
```

The feature of mapping a virtual directory to a physical path is available only on Windows apps.

### [Local Cache](https://learn.microsoft.com/en-us/azure/app-service/overview-local-cache)

Not supported for function apps or containerized App Service apps. To enable: `WEBSITE_LOCAL_CACHE_OPTION = Always`

## [Persistence](https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container?tabs=debian&pivots=container-linux#use-persistent-shared-storage)

**Persistent storage disabled:** Files written to `/home` are lost on restart or across `instances` (scaled-out app).

**Persistent storage enabled:** All `/home` writes persist and are shared across all instances, with persistent files overwriting container defaults on startup.

**Exception:** `/home/LogFiles` always persists when `Application Logging (Filesystem)` option, regardless of the persistent storage setting.

By default, persistent storage is disabled on Linux custom containers. To enable it, set the `WEBSITES_ENABLE_APP_SERVICE_STORAGE=true`.

### [Mount Azure Storage as a local share in App Service](https://learn.microsoft.com/en-us/azure/app-service/configure-connect-to-azure-storage)

#### Key notes:
- Supports Azure Files (read/write) and Azure Blobs (read-only for Linux, N/A for Windows).
- App backups don't include storage mounts.
- To avoid latency issues, place the app and the Azure Storage account in the same region.
- Adding, editing, or deleting a storage mount causes the app to restart.
- Ensure removal of corresponding storage mount configuration in the app if you delete an Azure Storage account, container, or share.
- Don't use mounts for local databases (e.g. SQLite) or apps require file locks.

Mount: `az webapp config storage-account add --custom-id <custom-id> --storage-type AzureFiles --share-name <share-name> --account-name <storage-account-name> --access-key "<access-key>" --mount-path <mount-path-directory> ...`  
Check: `az webapp config storage-account list ...`

### Limitations

- Don't map to `/`, `/home`, or `/tmp` to avoid issues.
- Storage firewall support via service and private endpoints only.
- No FTP/FTPS for custom-mounted storage.
- Azure Storage billed separately from App Service.

### Best Practices

- Place app and storage in the same Azure region.
- Avoid regenerating access key.

## Deploying apps (gist)

1. Create resource group: `az group create`
1. Create App Service plan: `az appservice plan create --location $location`
1. Create web app: `az webapp create --runtime "DOTNET|6.0"`
1. (optinal) Use managed identity for ACR:
   - Assign managed identity to the web app
   - Assign `AcrPull` role: `az role assignment create --assignee $principalId --scope $registry_resource_id --role "AcrPull"`
   - Set generic config to `{acrUseManagedIdentityCreds:true}` for system identity and `{acrUserManagedIdentityID:id}` for user identity.  
      `az webapp config set --generic-configurations '{"acrUserManagedIdentityID": "<client-id>"}'`
1. (optional) Create deployment slot (staging) (Standard+): `az webapp deployment slot create`
1. Deploy app (add `--slot staging` to use deployment slot):
   - Git: `az webapp deployment source config --repo-url $gitrepo --branch master --manual-integration`
   - Docker: `az webapp config container set --docker-custom-image-name`
   - Compose (skip step 3): `az webapp create --multicontainer-config-type compose --multicontainer-config-file $dockerComposeFile`
   - Local ZIP file: `az webapp deploy --src-path "path/to/zip"`
   - Remote ZIP file: `az webapp deploy --src-url "<url>"`
1. (optional) Set some settings: `az webapp config appsettings set --settings` (ex: `DEPLOYMENT_BRANCH='main'` for git, `SCM_DO_BUILD_DURING_DEPLOYMENT=true` for build automation)
1. (optional) Swap slots: `az webapp deployment slot swap --slot staging`

### ARM Templates

In JSON format.

`az group export --name $resourceGroup` - create ARM template

`az group deployment export --name $resourceGroup --deployment-name $deployment` - create ARM template for specific deploy

`az deployment group create --resource-group $resourceGroup --template-file $armTemplateJsonFile` - create deployment at resource group

## Security

### [Authentication](https://learn.microsoft.com/en-us/azure/app-service/overview-authentication-authorization)

Enabling this feature will automatically redirect all requests to HTTPS. You can either restrict access to authenticated users or allow anonymous requests. Built-in token store for managing tokens.

In Windows (non-container), the authentication/authorization module runs as a native IIS module in the same sandbox as your application (shares the same runtime, memory space, and security context). When enabled, every incoming HTTP request passes through it before your application handles it.

#### [Service Identity](https://learn.microsoft.com/en-us/azure/app-service/overview-managed-identity)

Each deployment slot / app has it's own managed identity configuration.

#### REST endpoint reference

An app with a managed identity defines two environment variables to make an endpoint available. This endpoint can be used to request tokens for accessing other Azure services

- `IDENTITY_ENDPOINT` endpoint from which apps can request tokens.
- `IDENTITY_HEADER` - (uuid) used to help mitigate server-side request forgery (SSRF) attacks.

Endpoint parameters:

- Required: `resource`, `api-version`, `X-IDENTITY-HEADER` (header)
- Optional: `client_id`, `principal_id`, `mi_res_id`

For user-assigned identities, include one of the optional properties; without it, a system-assigned identity token is requested.

Example:

```http
GET {IDENTITY_ENDPOINT}?resource=https://vault.azure.net&api-version=2019-08-01&client_id=XXX
X-IDENTITY-HEADER: {IDENTITY_HEADER}
```

#### Authentication flows

OAuth enables apps to access resources via user permissions, bypassing the need for credentials. Azure App Service manages this through its authentication module, which handles sessions and tokens. It can authenticate requests and redirect unauthenticated users (login page or 401). Tokens are stored in a token store when enabled. Note: An Access Rule is required.

- **Server-directed** (no SDK): handled by App Service, for browser apps
- **Client-directed** (SDK): handled by the app, for non-browser apps

| Step                            | Server-directed                                               | Client-directed                                                                              |
| ------------------------------- | ------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Sign user in                    | Redirects client to `/.auth/login/aad` (MS Identity Platform) | Client code signs user in directly with provider's SDK and receives an authentication token. |
| Post-authentication             | Provider redirects client to `/.auth/login/aad/callback`      | Client code posts token from provider to `/.auth/login/aad` for validation.                  |
| Establish authenticated session | App Service adds authenticated cookie to response             | App Service returns its own authentication token to client code                              |
| Serve authenticated content     | Client includes authentication cookie in subsequent requests  | Client code presents authentication token in `X-ZUMO-AUTH` header                            |

Access another app: Add header `Authorization: Bearer ${req.headers['x-ms-token-aad-access-token']}`

##### Access user claims in app code

```cs
private class ClientPrincipalClaim
{
    [JsonPropertyName("typ")] public string Type { get; set; }
    [JsonPropertyName("val")] public string Value { get; set; }
}

private class ClientPrincipal
{
    [JsonPropertyName("auth_typ")] public string IdentityProvider { get; set; }
    [JsonPropertyName("name_typ")] public string NameClaimType { get; set; }
    [JsonPropertyName("role_typ")] public string RoleClaimType { get; set; }
    [JsonPropertyName("claims")] public IEnumerable<ClientPrincipalClaim> Claims { get; set; }
}

public static ClaimsPrincipal Parse(HttpRequest req)
{
    var principal = new ClientPrincipal();

    if (req.Headers.TryGetValue("x-ms-client-principal", out var header))
    {
        var json = Encoding.UTF8.GetString(Convert.FromBase64String(header[0]));
        principal = JsonSerializer.Deserialize<ClientPrincipal>(json, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
    }

    // Code can now iterate through `principal.Claims` for validation
    // or converts it to a `ClaimsPrincipal` for later use in the request pipeline.

    var identity = new ClaimsIdentity(principal.IdentityProvider, principal.NameClaimType, principal.RoleClaimType);
    identity.AddClaims(principal.Claims.Select(c => new Claim(c.Type, c.Value)));

    return new ClaimsPrincipal(identity);
}
```

### [Certificates](https://learn.microsoft.com/en-us/azure/app-service/configure-ssl-certificate?tabs=apex)

A certificate is accessible to all apps in the same resource group and region combination.

- **Free Managed Certificate**: Auto renewed every 6 months, no wildcard certificates or private DNS, can't be exported (**cannot be used in other apps**), not supported in ASE.
- **App Service Certificate**: A private certificate that is managed by Azure. Automated certificate management, renewal and export options.
- **Using Key Vault**: Store private certificates (same requerenments) in Key Vault. Automatic renewal, except for non-integrated certificates (`az keyvault certificate create ...`, default policy: `az keyvault certificate get-default-policy`)
- **Uploading a Private Certificate**: Requires a password-protected PFX file encrypted with triple DES, with 2048-bit private key and all intermediate/root certificates in the chain.
- **Uploading a Public Certificate**: For accessing remote resources.

[Make certificate accessible](https://learn.microsoft.com/en-us/azure/app-service/configure-ssl-certificate-in-code): `az webapp config appsettings set --settings WEBSITE_LOAD_CERTIFICATES=<comma-separated-certificate-thumbprints>`, then use `X509Store.Certificates.Find(X509FindType.FindByThumbprint, "certificate-thumbprint", true)` to load it.

#### [TLS mutual authentication](https://learn.microsoft.com/en-us/azure/app-service/app-service-web-configure-tls-mutual-auth?tabs=azurecli)

Requires Basic+ plan; set from `Configuration > General Settings`.

TLS termination is handled by frontend load balancer. When enabling client certificates (`az webapp update --set clientCertEnabled=true ...`), `X-ARR-ClientCert` header is added. Accessing client certificate: `HttpRequest.ClientCertificate`:

For NodeJs, client certificate is accessed through request header: `req.get('X-ARR-ClientCert');`

```cs
// Forward the client certificate from the frontend load balancer
services.AddCertificateForwarding(options => { options.CertificateHeader = "X-ARR-ClientCert"; });

// Adds certificate-based authentication to the application.
services.AddAuthentication(CertificateAuthenticationDefaults.AuthenticationScheme).AddCertificate();
```

### [CORS](https://learn.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-rest-api)

For apps: `az webapp cors add --allowed-origins $website ...`

For storage: `az storage cors add --services blob --methods GET POST --origins $website --allowed-headers '*' --exposed-headers '*' --max-age 200 ...`

_To enable the sending of credentials like cookies or authentication tokens in your app_, the browser may require the `ACCESS-CONTROL-ALLOW-CREDENTIALS` header in the response: `az resource update --set properties.cors.supportCredentials=true --namespace Microsoft.Web --resource-type config --parent sites/$appName ...`

## Networking

- **Deployment Types**

  - Multi-tenant setup where your application shares resources with other applications.
  - Single-tenant setup, called App Service Environment (ASE), where your application gets its own dedicated resources within your Azure virtual network.

- [**Networking Features**](https://learn.microsoft.com/en-us/azure/app-service/networking-features): Manage both incoming (inbound) and outgoing (outbound) network traffic.

  | Feature                                      | Type     | Use Cases                                                                                 |
  | -------------------------------------------- | -------- | ----------------------------------------------------------------------------------------- |
  | App-assigned address                         | Inbound  | Support IP-based SSL for your app; Support a dedicated inbound address for your app       |
  | Access restrictions                          | Inbound  | Restrict access to your app from a set of well-defined IP addresses                       |
  | Service endpoints/Private endpoints          | Inbound  | Restrict access to your Azure Service Resources to only your virtual network              |
  | Hybrid Connections                           | Outbound | Access an on-premises system or service securely (from Azure to On-Premises)              |
  | Gateway-required virtual network integration | Outbound | Access Azure or on-premises resources via ExpressRoute or VPN (two way Azure-On-Premises) |
  | Virtual network integration                  | Outbound | Access Azure network resources                                                            |

  Hybrid Connections: from Azure to On-Premises; Gateway: two way Azure-On-Premises.

**Ingress** - manages external access to the services running in a container. It allows you to define how external traffic should be routed to the services within your containerized application. Set "external" to allow public traffic. Ingress configurations typically specify rules for directing HTTP and HTTPS traffic to specific services based on factors like the request path or host header.

- **Default Networking Behavior**: Free and Shared plans use multi-tenant workers, meaning your application shares resources with others. Plans from Basic and above use dedicated workers, meaning your application gets its own resources. If you have a Standard App Service plan, all the apps in that plan run on the same worker.

- **Outbound Addresses**: When your application needs to make a call to an external service, it uses an outbound IP address. This address is shared among all applications running on the same type of worker VM.

  - To find the current outbound IP addresses: `az webapp show --query outboundIpAddresses ...`
  - To find all possible outbound IP addresses: `az webapp show --query possibleOutboundIpAddresses ...`

## [Diagnostics](https://learn.microsoft.com/en-us/azure/app-service/troubleshoot-diagnostic-logs)

| Type                    | Platform       | Location                                           | Notes                                                                                                                              |
| ----------------------- | -------------- | -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Application logging     | Windows, Linux | App Service file system and/or Azure Storage blobs | Useful for debugging issues (bugs or unexpected behavior) within application code.                                                 |
| Web server logging      | Windows        | App Service file system or Azure Storage blobs     | Raw HTTP request data. Useful for diagnosing issues related to connectivity, HTTP errors (`404`), and server-level issues (`5xx`). |
| Detailed Error Messages | Windows        | App Service file system                            | Copies of the .htm error pages that would have been sent to the client browser.                                                    |
| Failed request tracing  | Windows        | App Service file system                            |                                                                                                                                    |
| Deployment logging      | Windows, Linux | App Service file system                            | Logs for when you publish content to an app.                                                                                       |

The _App Service file system_ option is for temporary debugging purposes, and turns itself off in 12 hours.  
_The Blob_ option is for long-term logging, includes additional information. .Net apps only.

`az webapp log config --application-logging {azureblobstorage, filesystem, off} --name MyWebapp --resource-group $resourceGroup`

Accessing log files:

- Linux/custom containers: `https://$appName.scm.azurewebsites.net/api/logs/docker/zip`. The ZIP file contains console output logs for both the docker host and the docker container.
- Windows apps: `https://$appName.scm.azurewebsites.net/api/dump`

`AppServiceFileAuditLogs` and `AppServiceAntivirusScanAuditLogs` log types are available only for Premium+.

`AllMetrics` settings are collected by agents on to the App Service and report the usage of host resources. These are items like CPU usage, memory usage, and disk I/O used.

### Stream logs

To stream logs in the Azure portal, navigate to your app and select **Log stream**.

Logs written to .txt, .log, or .htm files in `/home/LogFiles` (or `D:\home\LogFiles` for Windows apps) . Note, some logs may appear out of order due to buffering.

CLI: `az webapp log tail ...`

```sh
# Stream HTTP logs
az webapp log tail --provider http --name $app --resource-group $resourceGroup

# Stream errors
az webapp log tail --filter Error --name $app --resource-group $resourceGroup # filter by word Error
az webapp log tail --only-show-errors --name $app --resource-group $resourceGroup
```

### [Monitoring apps](https://learn.microsoft.com/en-us/azure/app-service/web-sites-monitor)

Metrics: CPU Percentage, Memory Percentage, Data In, Data Out - used across all instances of the plan (**not a single app!**).

Example: `Metric: CPU Percentage; Resource: <AppServicePlanName>`

```sh
az monitor metrics list --resource $app_service_plan_resource_id --metric "Percentage CPU" --time-grain PT1M --output table
```

CPU Time is valuable for apps on Free or Shared plans, where quotas are set by app's CPU minutes usage.  
The CPU percentage is valuable for apps on Basic+, providing insights into usage across scalable instances.

### [Health Checks](https://learn.microsoft.com/en-us/azure/app-service/monitor-instances-health-check?tabs=dotnet)

Health Check pings the specified path every minute. If an instance fails to respond with a valid status code after 10 requests, it's marked unhealthy and removed from the load balancer. If it recovers, it's returned to the load balancer. If it stays unhealthy for an hour, it's replaced (only for Basic+).

For private endpoints check if `x-ms-auth-internal-token` request header equals the hashed value of `WEBSITE_AUTH_ENCRYPTION_KEY` environment variable. You should first use features such as IP restrictions, client certificates, or a Virtual Network to restrict application access.

Configure path: `az webapp config set --health-check-path <Path> --resource-group $resourceGroup --name $webApp`

### [Application Insights Profiler](https://learn.microsoft.com/en-us/azure/azure-monitor/profiler/profiler)

Requires the **Always on** setting is _enabled_.

## Deploying apps (full)

```sh
let "randomIdentifier=$RANDOM*$RANDOM"
location="East US"
resourceGroup="app-service-rg-$randomIdentifier"
tag="deploy-github.sh"
appServicePlan="app-service-plan-$randomIdentifier"
webapp="web-app-$randomIdentifier"
gitrepo="https://github.com/Azure-Samples/dotnet-core-sample"

az group create --name $resourceGroup --location "$location" --tag $tag

az appservice plan create --name $appServicePlan --resource-group $resourceGroup --location $location # --sku B1
# az appservice plan create --name $appServicePlan --resource-group $resourceGroup --sku S1 --is-linux

az webapp create --name $webapp --plan $appServicePlan --runtime "DOTNET|6.0" --resource-group $resourceGroup

# https://learn.microsoft.com/en-us/azure/app-service/scripts/cli-deploy-github
github_deployment() {
    echo "Deploying from GitHub"
    az webapp deployment source config --name $webapp --repo-url $gitrepo --branch master --manual-integration --resource-group $resourceGroup

    # Change deploiment branch to "main"
    # az webapp config appsettings set --name $webapp --settings DEPLOYMENT_BRANCH='main' --resource-group $resourceGroup
}

# https://learn.microsoft.com/en-us/azure/app-service/scripts/cli-deploy-staging-environment
# Use it to avoid locking files
staging_deployment() {
    # Deployment slots require Standard tier, default is Basic (B1)
    az appservice plan update --name $appServicePlan --sku S1 --resource-group $resourceGroup

    echo "Creating a deployment slot"
    az webapp deployment slot create --name $webapp --slot staging --resource-group $resourceGroup

    echo "Deploying to Staging Slot"
    az webapp deployment source config --name $webapp --resource-group $resourceGroup \
      --slot staging \
      --repo-url $gitrepo \
      --branch master --manual-integration \


    echo "Swapping staging slot into production"
    az webapp deployment slot swap --slot staging --name $webapp --resource-group $resourceGroup
}

# https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container?tabs=debian&pivots=container-linux#change-the-docker-image-of-a-custom-container
docker_deployment() {
    # (Optional) Use managed identity: https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container?tabs=debian&pivots=container-linux#change-the-docker-image-of-a-custom-container
    ## Enable the system-assigned managed identity for the web app
    az webapp identity assign --name $webapp --resource-group $resourceGroup
    ## Grant the managed identity permission to access the container registry
    az role assignment create --assignee $principalId --scope $registry_resource_id --role "AcrPull"
    ## Configure your app to use the system managed identity to pull from Azure Container Registry
    az webapp config set --generic-configurations '{"acrUseManagedIdentityCreds": true}' --name $webapp --resource-group $resourceGroup
    ## (OR) Set the user-assigned managed identity ID for your app
    az webapp config set --generic-configurations '{"acrUserManagedIdentityID": "$principalId"}' --name $webapp --resource-group $resourceGroup

    echo "Deploying from DockerHub" # Custom container
    az webapp config container set --name $webapp --resource-group $resourceGroup \
      --docker-custom-image-name <docker-hub-repo>/<image> \
      # Private registry: https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container?tabs=debian&pivots=container-linux#use-an-image-from-a-private-registry
      --docker-registry-server-url <private-repo-url> \
      --docker-registry-server-user <username> \
      --docker-registry-server-password <password>

    # NOTE: Another version of it, using
    # az webapp create --deployment-container-image-name <registry-name>.azurecr.io/$image:$tag
    # https://learn.microsoft.com/en-us/azure/app-service/tutorial-custom-container
}

# https://learn.microsoft.com/en-us/azure/app-service/tutorial-multi-container-app
compose_deployment() {
    echo "Creating webapp with Docker Compose configuration"
    $dockerComposeFile=docker-compose-wordpress.yml
    # Note that az webapp create is different
    az webapp create --resource-group $resourceGroup --plan $appServicePlan --name wordpressApp --multicontainer-config-type compose --multicontainer-config-file $dockerComposeFile

    echo "Setup database"
    az mysql server create --resource-group $resourceGroup --name wordpressDb  --location $location --admin-user adminuser --admin-password letmein --sku-name B_Gen5_1 --version 5.7
    az mysql db create --resource-group $resourceGroup --server-name <mysql-server-name> --name wordpress

    echo "Setting app settings for WordPress"
    az webapp config appsettings set \
      --settings WORDPRESS_DB_HOST="<mysql-server-name>.mysql.database.azure.com" WORDPRESS_DB_USER="adminuser" WORDPRESS_DB_PASSWORD="letmein" WORDPRESS_DB_NAME="wordpress" MYSQL_SSL_CA="BaltimoreCyberTrustroot.crt.pem" \
      --resource-group $resourceGroup \
      --name wordpressApp
}

# https://learn.microsoft.com/en-us/azure/app-service/deploy-zip?tabs=cli
# uses the same Kudu service that powers continuous integration-based deployments
zip_archive() {
  az webapp deploy --src-path "path/to/zip" --name $webapp --resource-group $resourceGroup
  # Zip from url
  # az webapp deploy --src-url "https://storagesample.blob.core.windows.net/sample-container/myapp.zip?sv=2021-10-01&sb&sig=slk22f3UrS823n4kSh8Skjpa7Naj4CG3" --name $webapp --resource-group $resourceGroup

  # (Optional) Enable build automation
  # az webapp config appsettings set --settings SCM_DO_BUILD_DURING_DEPLOYMENT=true --name $webapp --resource-group $resourceGroup
}
```

## CLI

| Command                                                                                                                                  | Brief Explanation                                         | Example                                                                                                                                |
| ---------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| [az appservice plan](https://learn.microsoft.com/en-us/cli/azure/appservice/plan?view=azure-cli-latest)                                  | Manage App Service Plans.                                 | `az appservice plan create --name MyPlan --resource-group MyResourceGroup --sku FREE`                                                  |
| [az appservice plan update](https://learn.microsoft.com/en-us/cli/azure/appservice/plan?view=azure-cli-latest#az-appservice-plan-update) | Update an App Service Plan.                               | `az appservice plan update --name MyPlan --sku STANDARD`                                                                               |
| [az webapp](https://learn.microsoft.com/en-us/cli/azure/webapp?view=azure-cli-latest)                                                    | Manage web apps.                                          | `az webapp list`                                                                                                                       |
| [az webapp create](https://learn.microsoft.com/en-us/cli/azure/webapp?view=azure-cli-latest#az-webapp-create)                            | Create a web app.                                         | `az webapp create --name MyApp --plan MyPlan --resource-group MyResourceGroup`                                                         |
| [az webapp deployment](https://learn.microsoft.com/en-us/cli/azure/webapp/deployment?view=azure-cli-latest)                              | Manage web app deployments.                               | `az webapp deployment list-publishing-profiles --name MyApp`                                                                           |
| [az webapp config](https://learn.microsoft.com/en-us/cli/azure/webapp/config?view=azure-cli-latest)                                      | Manage web app configurations.                            | `az webapp config set --name MyApp --ftps-state AllAllowed`                                                                            |
| [az webapp config appsettings](https://learn.microsoft.com/en-us/cli/azure/webapp/config/appsettings?view=azure-cli-latest)              | Manage web app appsettings.                               | `az webapp config appsettings set --name MyApp --settings KEY=VALUE`                                                                   |
| [az webapp config connection-string](https://learn.microsoft.com/en-us/cli/azure/webapp/config/connection-string?view=azure-cli-latest)  | Manage web app connection strings.                        | `az webapp config connection-string set --name MyApp --connection-string-type SQLAzure --settings NAME=CONNECTION_STRING`              |
| [az webapp cors](https://learn.microsoft.com/en-us/cli/azure/webapp/cors?view=azure-cli-latest)                                          | Manage Cross-Origin Resource Sharing (CORS) for web apps. | `az webapp cors add --name MyApp --allowed-origins 'https://example.com'`                                                              |
| [az webapp show](https://learn.microsoft.com/en-us/cli/azure/webapp?view=azure-cli-latest#az-webapp-show)                                | Get details of a web app.                                 | `az webapp show --name MyApp --resource-group MyResourceGroup`                                                                         |
| [az webapp identity](https://learn.microsoft.com/en-us/cli/azure/webapp/identity?view=azure-cli-latest)                                  | Manage web app's managed service identity.                | `az webapp identity assign --name MyApp --resource-group MyResourceGroup`                                                              |
| [az identity](https://learn.microsoft.com/en-us/cli/azure/identity?view=azure-cli-latest)                                                | Manage Managed Service Identities (MSI).                  | `az identity create --resource-group MyResourceGroup --name MyIdentity`                                                                |
| [az webapp log](https://learn.microsoft.com/en-us/cli/azure/webapp/log?view=azure-cli-latest)                                            | Manage web app logs.                                      | `az webapp log tail --name MyApp --resource-group MyResourceGroup`                                                                     |
| [az webapp log config](https://learn.microsoft.com/en-us/cli/azure/webapp/log?view=azure-cli-latest#az-webapp-log-config)                | Configure logging for a web app.                          | `az webapp log config --name MyWebApp --resource-group MyResourceGroup --web-server-logging \| --docker-container-logging  filesystem` |
| [az resource update](https://learn.microsoft.com/en-us/cli/azure/resource?view=azure-cli-latest#az-resource-update)                      | Update a resource.                                        | `az resource update --ids RESOURCE_ID --set properties.key=value`                                                                      |
| [az webapp config storage-account](https://learn.microsoft.com/en-us/cli/azure/webapp/config/storage-account?view=azure-cli-latest)      | Manage web app's Azure Storage account configurations.    | `az webapp config storage-account update --name MyApp --custom-id CustomId --storage-type AzureBlob --account-name MyStorageAccount`   |
| [az webapp list-runtimes](https://learn.microsoft.com/en-us/cli/azure/webapp?view=azure-cli-latest#az-webapp-list-runtimes)              | List available runtime stacks.                            | `az webapp list-runtimes --linux`                                                                                                      |
| [az monitor metrics](https://learn.microsoft.com/en-us/cli/azure/monitor/metrics?view=azure-cli-latest)                                  | Manage metrics.                                           | `az monitor metrics list --resource RESOURCE_ID --metric-names "Percentage CPU"`                                                       |

# [Azure Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)

Application Insights, an extension of Azure Monitor, is a comprehensive Application Performance Monitoring (APM) tool. It provides unique features that help monitor applications throughout their lifecycle, from development and testing to production.

**📝 NOTE:** Application Insights automatically captures Session Id, so no need to manually capture it.

| Feature                                         | Description                                                                                          |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Live Metrics                                    | Real-time monitoring without affecting the host. For immediate insights during critical deployments. |
| Availability (Synthetic Transaction Monitoring) | Tests endpoints for availability and responsiveness. To ensure uptime and SLAs are met.              |
| GitHub or Azure DevOps integration              | Create GitHub or Azure DevOps work items in the context of Application Insights data.                |
| Usage                                           | Tracks popular features and user interactions.                                                       |
| Smart Detection                                 | Automatic failure and anomaly detection through proactive telemetry analysis.                        |
| Application Map                                 | Top-down view of app architecture with health indicators.                                            |
| Distributed Tracing                             | Search and visualize an end-to-end flow of a given execution or transaction.                         |

Application Insights monitors various aspects of your application's performance and health. It collects Metrics and Telemetry data, including:

- **Request rates, response times, and failure rates**: Identify popular pages, peak usage times, and user locations. Monitor page performance and detect resourcing issues during high request loads.
- **Dependency rates, response times, and failure rates**: Check if external services are causing slowdowns.
- **Exceptions**: Analyze aggregated statistics or specific instances, examining stack traces and related requests. Reports both server and browser exceptions.
- **Page views and load performance**: Information reported by users' browsers.
- **AJAX calls**: Monitor rates, response times, and failure rates for AJAX calls from web pages.
- **User and session counts**: Keep track of user and session numbers.
- **Performance counters**: Monitor CPU, memory, and network usage on Windows or Linux server machines.
- **Host diagnostics**: Gather diagnostic information from Docker or Azure.
- **Diagnostic trace logs**: Correlate trace events with requests by collecting logs from your app.
- **Custom events and metrics**: Create your own events and metrics in the client or server code to track specific business events, such as items sold or games won.

Here are various methods to start monitoring and analyzing your app's performance:

- **Run time**: Use Application Insights with your web app on the server, which is great for already deployed apps and doesn't require code updates.
- **Development time**: Add Application Insights into your code. This allows for customized telemetry collection and more extensive data collection.
- **Web page instrumentation**: Track page views, AJAX, and other client-side activities.
- **Mobile app analysis**: Use Visual Studio App Center to study mobile app usage.
- **Availability tests**: Regularly ping your website from our servers to test availability.

## Metrics

**Log-based metrics**: Offers thorough data analysis and diagnostics. ⭐: you need complete set of events ❌: high-volume apps that require sampling / filtering.

**Standard metrics** are time-series data **pre-aggregated** by either SDK (version does not affect accuracy) or backend (better accuracy), optimized for fast queries. ⭐: dashboards and real-time alerts, use cases _requiring sampling or filtering_.

You can toggle between these metrics types using the metrics explorer's namespace selector.

[Supported Metrics](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/metrics-index)

### [Sampling](https://learn.microsoft.com/en-us/azure/azure-monitor/app/sampling)

Reduces data traffic and costs while keeping analysis accurate. Helps avoid data limits and makes diagnostics easier. High sampling rates (> 60%) can affect log-based accuracy. Pre-aggregated metrics in SDKs solve this issue, but too much filtering can miss alerts.

- **Adaptive sampling**: On by default, adjusts data volume to stay within set limits. Used in Azure Functions.
- **Fixed-rate sampling**: You set the rate manually. ⭐: syncing client and server data to investigations of related events.
- **Ingestion sampling**: Discards some data at the service endpoint to stay within monthly limits. Doesn't reduce app traffic. Use if you hit monthly limits, or get too much data, or using older SDK.

For web apps, to group custom events, use the same `OperationId` value.

#### Configuring sampling

```cs
var builder = TelemetryConfiguration.Active.DefaultTelemetrySink.TelemetryProcessorChainBuilder;

// Enable AdaptiveSampling so as to keep overall telemetry volume to 5 items per second.
builder.UseAdaptiveSampling(maxTelemetryItemsPerSecond:5);

// Fixed rate sampling
builder.UseSampling(10.0); // percentage

// If you have other telemetry processors:
builder.Use((next) => new AnotherProcessor(next));
```

## [Custom events and metrics](https://learn.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics#getmetric)

Use [`GetMetric()`](https://learn.microsoft.com/en-us/azure/azure-monitor/app/get-metric) instead of `TrackMetric()`. `GetMetric()` handles **pre-aggregation**, reducing costs and performance issues associated with raw telemetry. It avoids sampling, ensuring reliable alerts. Tracking metrics at a granular level can lead to increased costs, network traffic, and throttling risks. `GetMetric()` solves these concerns by sending summarized data every minute.

```cs
TelemetryConfiguration configuration = TelemetryConfiguration.CreateDefault();
configuration.InstrumentationKey = "your-instrumentation-key-here";
var telemetry = new TelemetryClient(configuration);

// Set properties such as UserId and DeviceId to identify the machine.
// This information is attached to all events that the instance sends.
telemetry.Context.User.Id = "...";
telemetry.Context.Device.Id = "...";

// Monitors usage patterns and sends data to Custom Events for search.
// It names events and includes string properties and numeric metrics.
telemetry.TrackEvent("WinGame");

// GetMetric: capture locally pre-aggregated metrics for .NET and .NET Core applications

// TrackMetric: not the preferred method for sending metrics, but can be used if you're implementing your own pre-aggregation logic
var sample = new MetricTelemetry();
sample.Name = "queueLength";
sample.Sum = 42.3;
telemetry.TrackMetric(sample);

// Track page views at more or different times
telemetry.TrackPageView("GameReviewPage");

// Track the response times and success rates of calls to an external piece of code
var success = false;
var startTime = DateTime.UtcNow;
var timer = System.Diagnostics.Stopwatch.StartNew();

try
{
    // Some code that might throw an exception
    success = true;
}
catch (Exception ex)
{
    // Send exceptions to Application Insights
    telemetry.TrackException(ex);

    // Log exceptions to a diagnostic trace listener (Trace.aspx).
    Trace.TraceError(ex.Message);
}
finally
{
    timer.Stop();
    // TrackDependency: Tracks the performance of external dependencies not automatically collected by the SDK.
    // Use it to measure response times for databases, or external services.
    // Send data to Dependency Tracking in Application Insights
    telemetry.TrackDependency("DependencyType", "myDependency", "myCall", startTime, timer.Elapsed, success);
}

// Diagnose problems by sending a "breadcrumb trail" to Application Insights
// Lets you send longer data such as POST information.
telemetry.TrackTrace("Some message", SeverityLevel.Warning);

// Event log: use ILogger or a class inheriting EventSource.

// Send data immediately, rather than waiting for the next fixed-interval sending
telemetry.Flush();
```

Read more: [Dependency tracking in Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/asp-net-dependencies)

## [Usage analysis](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-overview)

- [User, session, and event analysis](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-segmentation)

  - **Users tool**: Counts unique app users per browser/machine.
  - **Sessions tool**: Tracks feature usage per session; resets after 30min inactivity or 24hr use.
  - **Events tool**: Measures page views and custom events like clicks.

- [Funnels](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-funnels): For linear, step-by-step processes. Track how users move through different stages of your web application, for example how many users go from the home page to creating a ticket. Use funnels to identify where users may stop or leave your app, helping you understand its effective areas and where improvements are needed.
- [User Flows](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-flows): For understanding complex, branching user behavior. Helps you analyze how users navigate between pages and features of your web app. It can answer questions like where users go after visiting a page, where they leave your site, or if they repeat the same action many times.
- [Cohorts](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-cohorts): Group and analyze sets of users, sessions, events, or operations that have something in common. For example, you might make a cohort of users who all tried a new feature.
- [Impact](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-impact): Helps you understand how different factors like load times and user properties influence conversion rates in various parts of your app.
- [Retention](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-retention): Helps you understand how many users come back to your app and how often they engage with specific tasks or goals. For example, if you have a game site, you can see how many users return after winning or losing a game.

## [Monitor an app (Instrumentation)](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-overview?tabs=aspnetcore)

- **Auto instrumentation**: Telemetry collection through configuration without modifying the application's code or configuring instrumentation.
- **Manual Instrumentation**: Coding against the **Application Insights** or **OpenTelemetry** API. Supports **Entra ID** and **Complex Tracing** (collect data that is not available in Application Insights)

## [Availability test](https://learn.microsoft.com/en-us/azure/azure-monitor/app/troubleshoot-availability)

Up to 100 tests per Application Insights resource.

- [URL ping test (classic - to be retired September 2026)](https://learn.microsoft.com/en-us/azure/azure-monitor/app/monitor-web-app-availability): Check endpoint response and measure performance. Customize success criteria with advanced features like parsing dependent requests and retries. It relies on public internet DNS; ensure public domain name servers resolve all test domain names. Use custom **TrackAvailability** tests otherwise.
- [Standard test](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability-standard-tests): Similar to URL ping, this single request test covers SSL certificate validity, proactive lifetime check, HTTP request verb (`GET`, `HEAD`, or `POST`), custom headers, and associated data.
- [Custom TrackAvailability test](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability-azure-functions): Use [TrackAvailability()](https://learn.microsoft.com/en-us/dotnet/api/microsoft.applicationinsights.telemetryclient.trackavailability) method to send results to Application Insights. Ideal for `multi-request` or `authentication` test scenarios. (Note: Multi-step test are the legacy version; To create multi-step tests, use Visual Studio)

Example: Create an alert that will notify you via email if the web app becomes unresponsive:

`Portal > Application Insights resource > Availability > Add Test option > Rules (Alerts) > set action group for availability alert > Configure notifications (email, SMS)`

## [Alert Processing Rules](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-processing-rules)

They allow you to suppress or modify notifications WITHOUT disabling the alert rules themselves. Alerts continue to fire and get logged (maintaining audit trail).

## [Troubleshoot app performance by using Application Map](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-map)

Application Map visualizes application topology and how components interact via HTTP dependencies. It's built from telemetry sent by the Application Insights SDK or OpenTelemetry.

### Key Requirements

- Use a **single Application Insights resource** (not just the same subscription) to see all components on a single map.
- Components are grouped by their `cloud_RoleName` value. To **customize component names**, override the it using:
  - Classic SDK: `cloud_RoleName`
  - OpenTelemetry: `service.name` + `service.namespace`

### Containerized Services with OpenTelemetry

In microservices or containerized environments, use **OpenTelemetry Resource attributes** to define component identity.

| OTel Attribute        | Maps To in App Insights    | Example              |
| --------------------- | -------------------------- | -------------------- |
| `service.name`        | Logical service name       | `orders-api`         |
| `service.namespace`   | Component grouping         | `ecommerce-platform` |
| `service.instance.id` | Unique service instance ID | `instance-01`        |

These values are translated internally by Azure Monitor:

- `cloud_RoleName` = `service.namespace`.`service.name`
- `cloud_RoleInstance` = `service.instance.id` (if provided)

### [Application Map Integration with OpenTelemetry](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-configuration)

To properly render all services in Application Map:

- All components **must send telemetry to the same Application Insights resource**.
- Each component **must set a distinct `cloud_RoleName`** (via `service.name` and `service.namespace`).

## Monitor a local web API by using Application Insights

```cs
// This forces HTTPS
builder.Services.AddApplicationInsightsTelemetry();
builder.Services.AddServiceProfiler();
```

appsettings.json:

```jsonc
"ApplicationInsights": {
  // Needed to send telemetry data to Application Insights
  "InstrumentationKey": "instrumentation-key"
}
```

Trust local certificates: `dotnet dev-certs https --trust`

## Azure Monitor

- Azure Monitor: Infrastructure and multi-resource monitoring, including hybrid and multi-cloud environments.
- Application Insights: Application-level monitoring (application performance management - APM), especially for web apps and services.

Application Insights data can also be viewed in Azure Monitor for a centralized experience.

### [Activity Log](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/activity-log)

Records subscription-level events, such as modifications to resources or starting a virtual machine.

**Diagnostic Settings**: Allows sending the activity log to different locations:

- **Log Analytics workspace**: Utilize log queries for deep insights (_Kusto queries_) and complex alerting. By default events are retained for 90 days, but you can create a diagnostic setting for longer retention.
- **Azure Storage account**: For audit, static analysis, or backup. Less expensive, and logs can be kept there indefinitely.
- **Azure Event Hubs**: Stream data to external systems such as third-party SIEMs and other Log Analytics solutions.

**📝 NOTE:** `az monitor activity-log` _cannot_ display data from Application Insight telemetry!  
**📝 NOTE:** `KQL` golden rule: filter early, filter often. Can reduce data processed by 90%+ if errors are rare

## Configuring

### Connection string

From env var `APPLICATIONINSIGHTS_CONNECTION_STRING`. Controls where telemetry is sent.

## Metric Source Scaling Rule

### Service Bus Queue

- **Message Count**: The number of messages currently in the queue.
- **Active Message Count**: The number of active messages in the queue.
- **Dead-letter Message Count**: The number of messages that have been moved to the dead-letter queue.
- **Scheduled Message Count**: The number of messages that are scheduled to appear in the queue at a future time.
- **Transfer Message Count**: The number of messages transferred to another queue or topic.
- **Transfer Dead-letter Message Count**: The number of messages transferred to the dead-letter queue for another queue or topic.

### Azure Blob Storage

- **Blob Count**: Number of blobs in a container.
- **Blob Size**: Total size of blobs in a container.
- **Egress**: Data egress rate.
- **Ingress**: Data ingress rate.

### Azure Event Hub

- **Incoming Messages**: Number of messages received.
- **Outgoing Messages**: Number of messages sent.
- **Capture Backlog**: Number of messages waiting to be captured.

## [OpenTelemetry](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable)

### Custom Metrics

| OTel Instrument            | Azure Aggregation Type    |
| -------------------------- | ------------------------- |
| `Counter` / `AsyncCounter` | Sum                       |
| `UpDownCounter`            | Sum                       |
| `Histogram`                | Min, Max, Avg, Sum, Count |

Histogram - closest to `GetMetric()` from classic SDK

#### Examples

```cs
var meter = new Meter("MyApp.Metrics");

var histogram = meter.CreateHistogram<long>("response_time_ms");
// Record a few random sale prices for apples and lemons, with different colors.
histogram.Record(rand.Next(1, 1000), new("name", "apple"), new("color", "red"));

var counter = meter.CreateCounter<long>("MyFruitCounter");
// Record the number of fruits sold, grouped by name and color.
counter.Add(1, new("name", "apple"), new("color", "red"));
```

### [Span Mapping to Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-add-modify)

Azure Monitor uses the type of **OpenTelemetry span** to determine how to classify telemetry in Application Insights. This classification happens **automatically** when OpenTelemetry telemetry is exported to Application Insights.

#### Span Kind → Application Insights Type

| OpenTelemetry Span Kind | Application Insights Equivalent     | Description                                    |
| ----------------------- | ----------------------------------- | ---------------------------------------------- |
| `Server`                | **Request**                         | Represents an incoming HTTP request            |
| `Client`                | **Dependency**                      | Represents an outgoing call to a service/API   |
| `Producer`              | **Dependency**                      | Used for messaging or queuing operations       |
| `Consumer`              | **Request**                         | When receiving a message                       |
| `Internal`              | **Custom Event / Custom Operation** | Custom telemetry (non-HTTP or system-internal) |

Practical Examples:

- You call a third-party REST API → OpenTelemetry uses a `Client Span` → Application Insights logs a **Dependency**
- Your API receives a request → `Server Span` → logged as a **Request**
- Your app posts to Azure Service Bus → `Producer Span` → logged as a **Dependency**
- Your background worker consumes from a queue → `Consumer Span` → logged as a **Request**

### [Filtering](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-filter)

- Remove noise (e.g. health checks)
- Drop sensitive data (PII, credentials)
- Improve performance by excluding low-value traces

```cs
// Via Instrumentation
builder.Services.AddOpenTelemetry().UseAzureMonitor().WithTracing(builder => builder.AddSqlClientInstrumentation(options => {
  options.SetDbStatementForStoredProcedure = false;
}));

// Via custom span processor
public class ActivityFilteringProcessor : BaseProcessor<Activity>
{
    public override void OnStart(Activity activity)
    {
        // Drop all internal spans
        if (activity.Kind == ActivityKind.Internal)
            activity.IsAllDataRequested = false;
    }
}
```

### [Pricing of Alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/cost-usage)

| Alert Type                      | Common Non-Free Tiers                                              | Free Tier                        |
| ------------------------------- | ------------------------------------------------------------------ | -------------------------------- |
| Activity Log                    | Free                                                               | Free                             |
| Service Health                  | Free                                                               | Free                             |
| Resource Health                 | Free                                                               | Free                             |
| Metric Alerts                   | Per time-series monitored (per month)                              | Yes (e.g., first 10 time-series) |
| Log Alerts                      | Per alert rule, based on evaluation frequency (1-min, 5-min, etc.) | Not free                         |
| Notifications - Email / Webhook | Per 1,000+ notifications                                           | Yes (generous monthly allowance) |
| Notifications - SMS / Voice     | Per notification                                                   | Not free (pay-per-use)           |

### [Azure Monitor retention periods](n/a)

| Item                                                          |             Default retention |                   Maximum retention |
| ------------------------------------------------------------- | ----------------------------: | ----------------------------------: |
| Log Analytics workspace (most tables)                         |                       30 days |                            730 days |
| Some Log Analytics tables (platform defaults such as metrics) |                       90 days |                            730 days |
| Per-table retention override                                  | Inherits workspace by default |                      Up to 730 days |
| Archived long-term storage                                    |                   Not default | Indefinite when archived to storage |
| Activity log Azure platform                                   |             Typically 90 days |   Can be retained longer via export |

<br>

# Storage Account

- Update to v2: `storage account > Settings > Configuration > Account kind > Upgrade`
- Creating a new policy: `storage account > Data management > Lifecycle management > List view > Add rule > fill out the Action set form fields > add an optional filter`
- Redundancy: `storage account > Data management > Redundancy`
- Increase DNS zone quota: `Azure Portal > Quotas > Storage > <subscription> > <region> > Request increase`
- Initialize failover: `storage account > Settings > Geo-replication > Prepare for failover`
- Toggle Static website: `storage account > Static Website`
- Static website address: `Settings > Endpoints > Static website`
- Get Account Access Key: `storage account > Security + networking > Access keys`
- Create encryption scope: `Azure portal >  Security + networking > Encryption > Encryption Scopes tab > Add` (can be selected under "Advanced Settings" for new blobs/containers)

## Security

- RBAC: `Azure Portal > Access control (IAM)`
- Configure the target resource to allow access from your app or function: `Azure Portal <Settings > Access policies > Add Access Policy`

# [Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction)

Endpoint: `https://<storage-account>.blob.core.windows.net/<container>/<blob>`

- Serving images or documents directly to a browser.
- Storing files for distributed access.
- Streaming video and audio.
- Writing to log files.
- Storing data for backup and restore, disaster recovery, and archiving.
- Storing data for analysis by an on-premises or Azure-hosted service.

## Types of resources

![Diagram showing the relationship between a storage account, containers, and blobs](https://learn.microsoft.com/en-us/azure/storage/blobs/media/storage-blobs-introduction/blob1.png)

### [Storage Account](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview)

```sh
az storage account create
    --name # valid DNS name, 3-24 chars
    --resource-group

    # Pricing tiers (<Type>_<Redundancy>)
    # Changing type: Copy to another account.
    # Changing redundancy: Instantly applied
    [--sku {Standard_GRS, Standard_GZRS, Standard_LRS, Standard_RAGRS, Standard_ZRS, Standard_RAGZRS, Premium_LRS, Premium_ZRS}]
    # Type 🧊:
    # - Standard: ⏺️⭐
    # - Premium: ⚡💲 (SSD). ⭐: using smaller objects
    # Redundancy:
    # - LRS: 🏷️, ❌: 🙋‍♂️.
    #   ⭐: your application can reconstruct lost data, requires regional replication (perhaps due to governance reasons), or uses Azure unmanaged disks.
    # - ZRS: Data write operations are confirmed successful once all the available zones have received the data. This even includes zones that are temporarily unavailable.
    #   ⭐: 🙋‍♂️, regional data replication, Azure Files workloads.
    # - GRS: LRS + async copy to a secondary region.
    # - GZRS: ZRS + async copy to a secondary region. 🦺
    # Read Access (RA): 🙋‍♂️ Allow read-only from `https://{accountName}-secondary.<url>`
    # Failover: manually initialized, swaps primary and secondary regions.
    # - C#: BlobClientOptions.GeoRedundantSecondaryUri (will not attempt again if 404).
    # - Alt: Copy data.
    # - ❌: Azure Files, BlockBlobStorage

    [--access-tier {Cool, Hot, Premium}] # Premium is inherited by SKU

    [--kind {BlobStorage, BlockBlobStorage, FileStorage, Storage, StorageV2}]
    # - BlobStorage: Simple blob-only scenarios.
    # - BlockBlobStorage: ⚡💎
    # - FileStorage: High-scale or high IOPS file shares. 💎
    # - Storage (General-purpose v1): Legacy. ⭐: classic deployment model or 🏋🏿 apps
    # - StorageV2: ⏺️⭐

    [--dns-endpoint-type {AzureDnsZone, Standard}] # Requires storage-preview extension
    # In one subscription, you can have accounts with both
    # - Standard: 250 accounts (500 with quota increase)
    # - AzureDnsZone: 5000 accounts
    # https://<storage-account>.z[00-50].<storage-service>.core.windows.net
    # Retrieve endpoints: GET https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Storage/storageAccounts/{accountName}?api-version=2022-09-01

    [--enable-hierarchical-namespace {false, true}]
    # Filesystem semantics. StorageV2 only. ❌ failover
```

Redundancy:

| LRS                                                                                                                   | ZRS                                                                                                                | GRS                                                                                                               | GZRS                                                                                                                    |
| --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| ![LRS](https://learn.microsoft.com/en-us/azure/storage/common/media/storage-redundancy/locally-redundant-storage.png) | ![ZRS](https://learn.microsoft.com/en-us/azure/storage/common/media/storage-redundancy/zone-redundant-storage.png) | ![GRS](https://learn.microsoft.com/en-us/azure/storage/common/media/storage-redundancy/geo-redundant-storage.png) | ![GZRS](https://learn.microsoft.com/en-us/azure/storage/common/media/storage-redundancy/geo-zone-redundant-storage.png) |

Pricing: ⏫ is not charged.

### Container

Organizes a set of blobs, similar to a directory in a file system. A storage account can include an unlimited number of containers, and a container can store an unlimited number of blobs.

```sh
az storage container create
    --name # Valid lowercase DNS name (3-63) with no double dashes
    [--resource-group]
    [--metadata]
    [--public-access {blob, container, off}]
```

### [Blob](https://learn.microsoft.com/en-us/rest/api/storageservices/understanding-block-blobs--append-blobs--and-page-blobs)

Each version of the blob has a unique tag, called an `ETag` that allows to only change a specific instance of the blob. Set type: `--type {block, append, page}`

- **Block blobs**: Store text and binary data in individual blocks, with a capacity of up to 190.7 TiB. `PUT <url>?comp=block&blockid=id`

- [**Append Blobs**](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-append): ⭐: append operations (ex: logging data from virtual machines). `PUT <url>?comp=appendblock`. `Add` permission is only applicable to this type of blob.

  ```cs
  AppendBlobClient appendBlobClient = containerClient.GetAppendBlobClient(logBlobName);
  // Read appendBlobClient.AppendBlobMaxAppendBlockBytes bytes of data
  await appendBlobClient.AppendBlockAsync(memoryStream);
  ```

- [**Page blobs**](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-pageblob-overview): Stores random access files up to 8 TiB. ⭐: VHD files, disks for Azure virtual machines, or databseses. `PUT <url>?comp=page`

  ![A diagram showing the two separate write options.](https://learn.microsoft.com/en-us/azure/storage/blobs/media/storage-blob-pageblob-overview/storage-blob-pageblob-overview-figure2.png)

  ```cs
  var pageBlobClient = blobContainerClient.GetPageBlobClient("0s4.vhd");
  pageBlobClient.Create(16 * 1024 * 1024 * 1024 /* 1GB */); // create an empty page blob of a specified size
  pageBlobClient.Resize(32 * 1024 * 1024 * 1024); // resize a page blob
  pageBlobClient.UploadPages(dataStream, startingOffset); // write pages to a page blob
  var pageBlob = pageBlobClient.Download(new HttpRange(bufferOffset, rangeSize)); // read pages from a page blob
  ```

## [Access tiers](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)

```sh
# Account level
az storage account create --access-tier {Cool, Hot, Premium} # Premium is inherited by SKU

# Individual blob level
az storage blob set-tier
    --tier {Archive, Cool, Hot}

    [--rehydrate-priority {Standard, High}] # For archived objects < 10GB, 1-15 hours
    # Can be checked by `x-ms-rehydrate-priority` header.

    [--type {block, page}] # append is considered always "hot"
```

- Hot: Frequently accessed or modified data. ⚡💲
- Not-hot: Infrequently accessed or modified data (ex: short-term data backup and disaster recovery). Penalty for early removal of data. 🏷️
- Archive: Only for individual blob blocks. 🐌. _Offline_ (requires rehydration to be accessed, at least an hour). To access data, either [copy](https://learn.microsoft.com/en-us/azure/storage/blobs/archive-rehydrate-overview#copy-an-archived-blob-to-an-online-tier) or [change](https://learn.microsoft.com/en-us/azure/storage/blobs/archive-rehydrate-overview#change-a-blobs-access-tier-to-an-online-tier) data to online tier. Rehydration copy to a different account is supported if the other account is within the same region. Destination cannot be at archive tier. ❌: \*ZRS redundancy.
- Non-archive: _Online_ (can be accessed at any time). ❌: append blobs. ⚡

Min retention period (in days): 30 (Cool), 90 (Cold), Archive (180). To avoid penalty, choose a tier with less than required days.

Changing a blob's tier leaves its last modified time untouched. If a lifecycle policy is active, using **Set Blob Tier** to rehydrate may cause the blob to return to the archive tier if the last modified time exceeds the policy's threshold.

## [Lifecycle policies](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)

`General Purpose v2` only.  
❌: `$logs` or `$web` containers

- Transition blobs to a cooler storage tier for 🏷️
- Delete blobs at the end of their lifecycles
- Define rules to be run once per day at the storage account level
- Apply rules to containers or a subset of blobs (using prefixes as filters)

```sh
az storage account management-policy create \
    #--account-name "<storage-account>" \
    #--resource-group $resourceGroup
    --policy @policy.json \
```

```ts
type RelesType = {
  rules: [
    {
      enabled: boolean;
      name: string;
      type: "Lifecycle";
      definition: {
        actions: {
          // NOTE: Delete is the only action available for all blob types; snapshots cannot auto set to hot
          version?: RuleAction;
          /* blobBlock */ baseBlob?: RuleAction;
          snapshopt?: Omit<RuleAction, "enableAutoTierToHotFromCool">;
          appendBlob?: { delete: ActionRunCondition }; // only one lifecycle policy
        };
        filters?: {
          blobTypes: Array<"appendBlob" | "blockBlob">;
          // A prefix string must start with a container name.
          // To match the container or blob name exactly, include the trailing forward slash ('/'), e.g., 'sample-container/' or 'sample-container/blob1/'
          // To match the container or blob name pattern (wildcard), omit the trailing forward slash, e.g., 'sample-container' or 'sample-container/blob1'
          prefixMatch?: string[];
          // Each rule can define up to 10 blob index tag conditions.
          // example, if you want to match all blobs with `Project = Contoso``: `{"name": "Project","op": "==","value": "Contoso"}``
          // https://learn.microsoft.com/en-us/azure/storage/blobs/storage-manage-find-blobs?tabs=azure-portal
          blobIndexMatch?: Record<string, string>;
        };
      };
    }
  ];
};

type RuleAction = {
  tierToCool?: ActionRunCondition;
  tierToArchive?: {
    daysAfterModificationGreaterThan: number;
    daysAfterLastTierChangeGreaterThan: number;
  };
  enableAutoTierToHotFromCool?: ActionRunCondition;
  delete?: ActionRunCondition;
};

type ActionRunCondition = {
  daysAfterModificationGreaterThan: number;
  daysAfterCreationGreaterThan: number;
  daysAfterLastAccessTimeGreaterThan: number; // requires last access time tracking
};
```

It takes _up to 24 hours to go into effect_. Then it could take additional _up to 24 hours_ for some actions to run for the first time.

Data stored in a premium block blob storage account cannot be tiered to Hot, Cool, or Archive using Set Blob Tier or using Azure Blob Storage lifecycle management.

If you define more than one action on the same blob, lifecycle management applies the least 💲 action to the blob: `delete < tierToArchive < tierToCool`.

❌: partial updates

Access time tracking: when is enabled (`az storage account blob-service-properties update --enable-last-access-tracking true`), a lifecycle management policy can include an action based on the time that the blob was last accessed with a read (tracks only the first in the past 24 hours) or write operation. 💲

## Transient error handling (retry strategy)

```cs
var options = new BlobClientOptions();
options.Retry.MaxRetries = 10;
opions.Retry.Delay = TimeSpan.FromSeconds(20);
var client = new BlobClient(new Uri("..."), options);
```

## Data Protection

| Feature            | [Snapshots](https://learn.microsoft.com/en-us/azure/storage/blobs/snapshots-overview) | [Versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-overview) |
| ------------------ | ------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Creation           | Manually                                                                              | Automatically (when enbled)                                                             |
| Immutability       | Read-only once created.                                                               | Previous versions are read-only.                                                        |
| URI                | DateTime value appended to base blob URI.                                             | Unique version ID for each version.                                                     |
| Flexibility        | More manual control, suitable for point-in-time backups.                              | Easier to manage, better for frequent changes.                                          |
| Tiers              | Not in Archive.                                                                       | All.                                                                                    |
| Deletion           | Must be deleted explicitly or with the base blob.                                     | Automatically managed; older versions can be deleted based on policies.                 |
| Soft Delete Impact | Soft-deleted along with the base blob; can be recovered during retention period.      | Current version becomes a previous version, and there's no longer a current version     |

```cs
BlobSnapshotInfo snapshotInfo = await blobClient.CreateSnapshotAsync();
// If you attempt to delete a blob that has snapshots, the operation will fail unless you explicitly specify that you also want to delete the snapshots
await blobClient.DeleteIfExistsAsync(DeleteSnapshotsOption.IncludeSnapshots);
```

## [Leasing](https://learn.microsoft.com/en-us/rest/api/storageservices/lease-blob)

Provides temporary exclusive write access to a Blob for a certain client with a lease key. Modifying the lease also requires that key (else `412 – Precondition failed` error)

```sh
leaseId=$(az storage blob lease acquire --lease-duration 60 --output tsv ...)
az storage blob lease {renew, change, release, break}
```

```cs
// Acquire a lease on the blob
string proposedLeaseId = Guid.NewGuid().ToString();
var leaseClient = blobClient.GetBlobLeaseClient(proposedLeaseId);
var lease = leaseClient.Acquire(TimeSpan.FromSeconds(15));
// leaseClient.Renew();
// leaseClient.Change(newLeaseId); // change the ID of an existing lease
// leaseClient.Release();
// leaseClient.Break(); // end the lease, but ensure that another client can't acquire a new lease until the current lease period has expired

// proposedLeaseId can now be passed as option in order to work with the blob
BlobUploadOptions uploadOptions = new BlobUploadOptions { Conditions = new BlobRequestConditions { LeaseId = proposedLeaseId } };
using (var stream = new MemoryStream(Encoding.UTF8.GetBytes("New content")));
await blobClient.UploadAsync(stream, uploadOptions);

BlobRequestConditions conditions = new BlobRequestConditions { LeaseId = proposedLeaseId };
await blobClient.SetMetadataAsync(newMetadata, conditions);
```

```http
PUT https://myaccount.blob.core.windows.net/mycontainer/myblob?comp=lease

Request Headers:
x-ms-version: 2015-02-21
x-ms-lease-action: acquire
x-ms-lease-duration: -1 # In seconds. -1 is infinite
x-ms-proposed-lease-id: 1f812371-a41d-49e6-b123-f4b542e851c5
x-ms-date: <date>
...

# Working with leased blob
PUT https://myaccount.blob.core.windows.net/mycontainer/myblob?comp=metadata

Request Headers:
x-ms-meta-name:string-value
x-ms-lease-id:[lease_id]
```

## Encryption

Data in Azure Storage is encrypted and decrypted transparently using 256-bit AES encryption (similar to BitLocker). Enforced for all tiers. Object metadata is also encrypted.

- **Microsoft Keys**: 🙂 All operations handled by Azure, supporting all services. Keys are stored by Microsoft, who also handles rotation and access.
- **Customer-Managed Keys**: Handled by Azure but you have more control. Supports some services, stored in Azure Key Vault. You handle key rotation and both you and Microsoft can access.
  ![Diagram showing how customer-managed keys work in Azure Storage ](https://learn.microsoft.com/en-us/azure/storage/common/media/customer-managed-keys-overview/encryption-customer-managed-keys-diagram.png)
- **Customer-Provided Keys**: Even more control, mainly for Blob storage. Can be stored in Azure or elsewhere, and you're responsible for key rotation. Only you can access.

### [Storage Account encryption](https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption)

```sh
az storage account <create/update>
    [--encryption-key-source {Microsoft.Keyvault, Microsoft.Storage}]
    [--encryption-services {blob, file, queue, table}] # queue / table with customer-managed keys = 💲
    [--encryption-key-type-for-queue {Account, Service}]
    [--encryption-key-type-for-table {Account, Service}]
    # When using Microsoft.Keyvault:
    #   [--encryption-key-name]
    #   [--encryption-key-vault] # URL
    #   [--encryption-key-version]

    # 🧊 Optionally encrypt infrastructure with separate Microsoft managed key. StorageV2 or BlockBlobStorage only.
    [--require-infrastructure-encryption {false, true}] # false
```

### [Container / Blob encryption (Encryption Scope)](https://learn.microsoft.com/en-us/azure/storage/blobs/encryption-scope-overview)

```sh
az storage account encryption-scope create
    --account-name
    --name "<scope-name>"
    [--key-source {Microsoft.KeyVault, Microsoft.Storage}] # Same rules like encryption at account level
    [--key-uri] # For KeyVault
    [--require-infrastructure-encryption {false, true}] # Inherited from storage account level, if set

# Optional
az storage container create
    --default-encryption-scope "<scope-name>"
    --prevent-encryption-scope-override true # force all blobs in a container to use the container's default scope

az storage <type> <operation>
    --encryption-scope "<scope-name>" # if not set, inherited from container or storage account
    # EncryptionScope property for BlobOptions in C#
```

Encryption makes Access tier 🧊.

Using disabled encryption scope will result in `403 Forbidden`.

### Client Side Encryption

```cs
// Your key and key resolver instances, either through Azure Key Vault SDK or an external implementation.
IKeyEncryptionKey key;
IKeyEncryptionKeyResolver keyResolver;

// Create the encryption options to be used for upload and download.
var encryptionOptions = new ClientSideEncryptionOptions(ClientSideEncryptionVersion.V2_0)
{
    KeyEncryptionKey = key,
    KeyResolver = keyResolver,
    // String value that the client library will use when calling IKeyEncryptionKey.WrapKey()
    KeyWrapAlgorithm = "some algorithm name"
};

// Set the encryption options on the client options.
var options = new SpecializedBlobClientOptions() { ClientSideEncryption = encryptionOptions };
// pass options to BlobServiceClient instance
```

## Properties and metadata

Metadata name/value pairs are valid HTTP headers, and so should adhere to all restrictions governing HTTP headers. Metadata names must be valid HTTP header names and valid C# identifiers, may contain only ASCII characters, and should be treated as case-insensitive.

### [Manage container properties and metadata by using .NET](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-container-properties-metadata)

Blob containers support system properties and user-defined metadata, in addition to the data they contain. Metadata values containing non-ASCII characters should be Base64-encoded or URL-encoded.

- **System properties**: System properties exist on each Blob storage resource. Some of them can be read or set, while others are read-only. Under the covers, some system properties correspond to certain standard HTTP headers.
- **User-defined metadata**: For your own purposes only, and do not affect how the resource behaves.

```cs
/// <exception cref="RequestFailedException">Thrown if a failure occurs.</exception>
Task<Azure.Response<BlobContainerProperties>> GetPropertiesAsync (BlobRequestConditions conditions = default, CancellationToken cancellationToken = default);

/// <exception cref="RequestFailedException">Thrown if a failure occurs.</exception>
Task<Azure.Response<BlobContainerInfo>> SetMetadataAsync (IDictionary<string,string> metadata, BlobRequestConditions conditions = default, CancellationToken cancellationToken = default);
```

If two or more metadata headers with the same name are submitted for a resource, Blob storage comma-separates and concatenates the two values and returns HTTP response code `200 (OK)` (Note: Rest cannot do that!)

Example:

```cs
// Fetch some container properties and write out their values.
var properties = await container.GetPropertiesAsync();
Console.WriteLine($"Properties for container {container.Uri}");
Console.WriteLine($"Public access level: {properties.Value.PublicAccess}");
Console.WriteLine($"Last modified time in UTC: {properties.Value.LastModified}");

var metadata = new Dictionary<string, string>() { { "docType", "textDocuments" }, { "category", "guidance" } };
var containerInfo = await container.SetMetadataAsync(metadata); // ETag, LastModified
```

### [Manage blob properties and metadata with .NET](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-properties-metadata)

```cs
public BlobServiceClient GetBlobServiceClient(string accountName)
{
    BlobServiceClient client = new(
        new Uri($"https://{accountName}.blob.core.windows.net"),
        new DefaultAzureCredential());

    return client;
}

...........

public static async Task SetBlobPropertiesAsync(BlobClient blob)
{
    Console.WriteLine("Setting blob properties...");

    try
    {
        // Get the existing properties
        BlobProperties properties = await blob.GetPropertiesAsync();

        BlobHttpHeaders headers = new BlobHttpHeaders
        {
            // Set the MIME ContentType every time the properties 
            // are updated or the field will be cleared
            ContentType = "text/plain",
            ContentLanguage = "en-us",

            // Populate remaining headers with 
            // the pre-existing properties
            CacheControl = properties.CacheControl,
            ContentDisposition = properties.ContentDisposition,
            ContentEncoding = properties.ContentEncoding,
            ContentHash = properties.ContentHash
        };

        // Set the blob's properties.
        await blob.SetHttpHeadersAsync(headers);
    }
    catch (RequestFailedException e)
    {
        Console.WriteLine($"HTTP error code {e.Status}: {e.ErrorCode}");
        Console.WriteLine(e.Message);
        Console.ReadLine();
    }
}

...........

private static async Task GetBlobPropertiesAsync(BlobClient blob)
{
    try
    {
        // Get the blob properties
        BlobProperties properties = await blob.GetPropertiesAsync();

        // Display some of the blob's property values
        Console.WriteLine($" ContentLanguage: {properties.ContentLanguage}");
        Console.WriteLine($" ContentType: {properties.ContentType}");
        Console.WriteLine($" CreatedOn: {properties.CreatedOn}");
        Console.WriteLine($" LastModified: {properties.LastModified}");
    }
    catch (RequestFailedException e)
    {
        Console.WriteLine($"HTTP error code {e.Status}: {e.ErrorCode}");
        Console.WriteLine(e.Message);
        Console.ReadLine();
    }
}
```

### [Set and retrieve properties and metadata for blob resources by using REST](https://learn.microsoft.com/en-us/rest/api/storageservices/setting-and-retrieving-properties-and-metadata-for-blob-resources)

Endpoint templates:

- Container: `https://<storage-account>.blob.core.windows.net/<container>?restype=container`
- Blob: `https://<storage-account>.blob.core.windows.net/<container>/<blob>?comp=metadata`

Retrieving metadata: `GET` or `HEAD` (example: `GET https://<storage-account>.blob.core.windows.net/<container>/<blob>?comp=metadata`)  
Setting metadata: `PUT` (example: `PUT https://myaccount.blob.core.windows.net/mycontainer?comp=metadata?restype=container`)

The format for the header is: `x-ms-meta-name:string-value`.

If two or more metadata headers with the same name are submitted for a resource, the Blob service returns status code `400 (Bad Request)`.

The total size of all metadata pairs can be up to 8KB in size.

Partial updates are not supported: setting metadata on a resource overwrites any existing metadata values for that resource.

### Standard Properties for Containers and Blobs

`ETag` and `Last-Modified` are common for containers and blobs.

For HTTP names start with `x-ms-meta`.

Containers:

- ETag (`x-ms-meta-etag`)
- Last-Modified (`x-ms-meta-last-modified`)

Blobs:

- ETag (`x-ms-meta-etag`)
- Last-Modified (`x-ms-meta-last-modified`)
- Content-Length (`x-ms-meta-content-length`)
- Content-Type (`x-ms-meta-content-type`)
- Content-MD5 (`x-ms-meta-content-md5`)
- Content-Encoding (`x-ms-meta-content-encoding`)
- Content-Language (`x-ms-meta-content-language`)
- Cache-Control (`x-ms-meta-cache-control`)
- Origin (`x-ms-meta-origin`)
- Range (`x-ms-meta-range`)

### Access conditions

```cs
BlobProperties properties = await blobClient.GetPropertiesAsync();

BlobRequestConditions conditions = new BlobRequestConditions
{
    IfMatch = properties.Value.ETag, // Limit requests to resources that have not be modified since they were last fetched.
    IfModifiedSince = DateTimeOffset.UtcNow.AddHours(-1), // Limit requests to resources modified since this time.
    IfNoneMatch = new Azure.ETag("some-etag-value"), // Limit requests to resources that do not match the ETag.
    IfUnmodifiedSince = DateTimeOffset.UtcNow.AddHours(-2), // Limit requests to resources that have remained unmodified.
    LeaseId = "some-lease-id", // Limit requests to resources with an active lease matching this Id.
    TagConditions = "tagKey = 'tagValue'" // Optional SQL statement to apply to the Tags of the Blob.
};

BlobUploadOptions options = new BlobUploadOptions
{
    Metadata = new Dictionary<string, string> { { "key", "value" } },
    Conditions = conditions
};

// Upload blob only if mathcing conditions
await blobClient.UploadAsync(BinaryData.FromString("data"), options);
```

```sh
az storage blob <operation>
    # ETag value, or the wildcard character (*)
    [--if-match]
    [--if-none-match]

    [--if-modified-since]
    [--if-unmodified-since]

    # SQL clause to check tags
    [--tags-condition]
```

## [Authorization](https://learn.microsoft.com/en-us/azure/storage/common/authorize-data-access)

- [Shared Key (storage account key)](https://learn.microsoft.com/en-us/rest/api/storageservices/authorize-with-shared-key) (`StorageSharedKeyCredential`): A secret password that gives full access to your Azure Storage account. ⭐: programmatic access (ex: data migration)
- [Microsoft Entra ID](https://learn.microsoft.com/en-us/azure/storage/blobs/authorize-access-azure-active-directory) (`DefaultAzureCredential`): Identity-based, role-based authorization with advanced security. ⭐: fine-grained enterprise access control.
- App credentials (`ClientSecretCredential`): App needs to be registered first.

### [Use OAuth access tokens for authentication](https://learn.microsoft.com/en-us/rest/api/storageservices/authorize-with-azure-active-directory#use-oauth-access-tokens-for-authentication)

- **Delegation Scope**: Use `user_impersonation` to allow applications to perform actions permitted by the signed-in user.
- **Resource ID**: Use `https://storage.azure.com/` to request tokens.

### [Manage access rights with RBAC](https://learn.microsoft.com/en-us/rest/api/storageservices/authorize-with-azure-active-directory#manage-access-rights-with-rbac)

In addition to `Storage Blob Data [Standard Role]` there also is `Storage Blob Delegator` for getting user delegation key.

Permissions for Blob service operations: `Microsoft.Storage/storageAccounts/blobServices/<op>` for top level operations, sub `containers/<op>` and `containers/blobs/<op>` for fine grained control. `<op>` can be `read`, `write`, `delete`, `filter/action` (find blobs, blob level only).

### [Anonymous public read access](https://learn.microsoft.com/en-us/azure/storage/blobs/anonymous-read-access-prevent?tabs=azure-cli)

Anonymous public access to your data is always prohibited by default. If the storage account doesn't allow public access, then no container within it can have public access, regardless of its own settings. If public access is allowed at the storage account level, then a container's access depends on its own settings (**Public access: Container**, or **Public access: Blob**).

## Working with blobs

### AZ CLI

```sh
az storage blob <command>
    # Authenticate:
    ## By Storage Account Key
    --account-key # az storage account keys list -g $resourcegroup -n $accountname --query '[0].value' -o tsv
    ## By Entra ID Login
    --auth-mode login # Use credentials from az login
    ## By Connection String
    --connection-string
    ## By SAS token
    --sas-token

    # Select target blob
    ## By name
    [--blob-endpoint] # https://<storage-account>.blob.core.windows.net
    [--account-name] # When using storage account key or a SAS token
    --container-name
    --name # Case sensitive, cannot end with dot (.) or dash (-)
    ## By URL
    --blob-url "https://<storage-account>.blob.core.windows.net/<container>/<blob>?<SAS>" # Use <SAS> only if unauthenticated.

    # <command>
    ## upload
    --file "/path/to/file" # for file uploads
    --data "some data" # for text uploads
    ## copy start-batch
    ## use -source-<prop> and --destination-<prop>

# Example
az storage blob upload --file /path/to/file --container mycontainer --name MyBlob
az storage container list --account-name $storageaccountname # get containers
```

### [C#](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-dotnet)

```cs
// Authorization
//// Storage Account Key
TokenCredential credential = new StorageSharedKeyCredential(accountName, "<account-key>");
//// Entra ID Login
TokenCredential credential = new DefaultAzureCredential();
//// App registration
TokenCredential credential = new ClientSecretCredential("<tenant-id>", "<client-id>", "<client-secret>");

// Select account
var blobServiceClient = new BlobServiceClient(new Uri($"https://${accountName}.blob.core.windows.net"), credential);

// Enumerate containers
await foreach (BlobContainerItem container in blobServiceClient.GetBlobContainersAsync()) {}

// Get container
// Note: BlobContainerClient allows you to manipulate both Azure Storage containers and their blobs
BlobContainerClient containerClient = blobServiceClient.GetBlobContainerClient(containerName);
// BlobContainerClient containerClient = await blobServiceClient.CreateBlobContainerAsync(containerName);
// BlobContainerClient containerClient = new BlobContainerClient(new Uri($"https://{accountName}.blob.core.windows.net/{containerName}"), credential, clientOptions);

BlobClient blobClient = containerClient.GetBlobClient(fileName);
// Getting a blob reference does not make any calls to Azure Storage, it simply creates an object locally that can work with a stored blob.

await blobClient.UploadAsync(localFilePath, true);

// List all blobs in the container
await foreach (var blobItem in containerClient.GetBlobsAsync()) {}

// Download the blob's contents and save it to a file
await blobClient.DownloadToAsync(downloadFilePath);
// Alternative version:
BlobDownloadInfo download = await blobClient.DownloadAsync();
using (FileStream downloadFileStream = File.OpenWrite(downloadFilePath));
await download.Content.CopyToAsync(downloadFileStream);

// Copy from one container to another, without downloading locally
BlobContainerClient sourceContainer = blobServiceClient.GetBlobContainerClient("sourcecontainer");
BlobClient sourceBlob = sourceContainer.GetBlobClient("sourceblob.txt");
BlobContainerClient targetContainer = blobServiceClient.GetBlobContainerClient("targetcontainer");
BlobClient targetBlob = targetContainer.GetBlobClient("targetblob.txt");
await targetBlob.StartCopyFromUriAsync(sourceBlob.Uri);
```

### HTTP

`PUT`: upload file, create container.

Headers:

- Optional: `x-ms-date`, `x-ms-version`
- `Authorization`:
  - Storage Account Key: `[Storage_Account_Key]`
  - Shared key: `SharedKey [your_account]:[signature]`
  - Entra ID: `Bearer [access_token]`

SAS GET param may be used instead of `Authorization` header.

```http
GET https://<account>.blob.core.windows.net/?comp=list # list containers
PUT https://<account>.blob.core.windows.net/<container>?restype=container # create container
GET https://<account>.blob.core.windows.net/<container>?restype=container&comp=list # list blobs in container
PUT https://<account>.blob.core.windows.net/<container>/<blob> # upload file
GET https://<account>.blob.core.windows.net/<container>/<blob> # download file
```

### [Object replication](https://learn.microsoft.com/en-us/azure/storage/blobs/object-replication-overview)

Asynchronously copies block blobs from a source to a destination account. To use this feature, you'll need to enable **Change Feed** and **Blob Versioning**, and it's compatible only with **general-purpose v2** and **premium block blob accounts**. However, it doesn't support append blobs, page blobs, or custom encryption keys.

The process involves 2 main steps:

1. Creating a replication policy.
1. Setting rules to specify which blobs to replicate.

### [AzCopy](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10)

```sh
# Move local data to blob storage
azcopy copy "C:\local\path" "https://<storage-account>.blob.core.windows.net/<container>/<sas-token>" --recursive=true

# Syncronize (avoid copying existing files again, when running multiple times)
azcopy sync \
  "https://<source-storage-account>.blob.core.windows.net/<container>/<sas-token>" \
  "https://<destination-storage-account>.blob.core.windows.net/<container>/<sas-token>" \
  --recursive=true
```

Destination needs `Microsoft.Storage/storageAccounts/blobServices/containers/blobs/add/action` permission when adding new lob to the destination.  
If copying existing blob, source needs `Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read` permission.

You can also use [AZ CLI](https://learn.microsoft.com/en-us/cli/azure/storage/blob/copy?view=azure-cli-latest), [Powershell](https://learn.microsoft.com/en-us/powershell/module/az.storage/start-azstorageblobcopy?view=azps-10.2.0&viewFallbackFrom=azps-4.6.0), or [SDK](https://learn.microsoft.com/en-us/dotnet/api/microsoft.azure.storage.blob.cloudblockblob.startcopyasync?view=azure-dotnet).

When copying, for destination you need SAS or OAuth authentication (Azure Files only supports SAS).

## [Network Access Rules](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security?tabs=azure-portal)

By default, Azure Storage Accounts accept connections from clients on any network.

Steps to Change the Default Network Access Rule:

- Disable All Public Network Access
- Enable Access from Selected Networks
- Apply Network Rules

# [Compute Solutions](https://learn.microsoft.com/en-us/azure/container-apps/compare-options)

| Service                   | Suitable Scenarios                                                          | Distinctive Features                                                                                  | Specific Criteria                                   |
| ------------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| Azure Container Apps      | Building serverless microservices based on containers                       | General-purpose containers, Kubernetes-style apps, event-driven architectures, long-running processes | No need for direct access to native Kubernetes APIs |
| Azure App Service         | Fully managed hosting for web applications, including websites and web APIs | Optimized for web applications, integrated with other Azure services                                  |                                                     |
| Azure Container Instances | Lower-level "building block" option compared to Container Apps              | No scale, load balancing, and certificates                                                            | Less "opinionated" building block                   |
| Azure Kubernetes Service  | Fully managed Kubernetes option in Azure                                    | Direct access to the Kubernetes API, runs any Kubernetes workload                                     |                                                     |
| Azure Functions           | Running event-driven applications using the functions programming model     | Short-lived functions deployed as either code or containers                                           |                                                     |

- **Azure Container Apps** vs. **AKS**: If you require access to the Kubernetes APIs and control plane, use AKS. If you want to build Kubernetes-style applications without needing direct access to all native Kubernetes APIs, Container Apps is preferable.
- **Azure Container Instances** vs. **Container Apps**: ACI is a more basic building block without application-specific concepts like scale and load balancing, while Container Apps provides these features.
- **Azure Functions** vs. **Container Apps**: Both are suitable for event-driven applications, but Functions is optimized for short-lived functions, while Container Apps is more general-purpose.

## Comparison by feature

| Feature                  | Container Apps | Container Instances | App Services                | Kubernetes Service (AKS)    | Functions                      |
| ------------------------ | -------------- | ------------------- | --------------------------- | --------------------------- | ------------------------------ |
| **Scaling**              | Auto-scaling   | Manual scaling only | Auto-scaling                | Cluster Autoscaler (say no) | Consumption-based auto-scaling |
| **State Management**     | Stateless      | Stateless           | Both stateless and stateful | Both stateless and stateful | Stateless                      |
| **Resource Isolation**   | Shared         | Dedicated           | Shared/Dedicated            | Dedicated                   | Shared                         |
| **Cost**                 | Pay-as-you-go  | Pay-as-you-go       | Fixed + Scale               | Cluster costs + Node costs  | Pay-as-you-go or fixed         |
| **Multi-Region Support** | No             | No                  | Yes                         | Yes                         | Yes                            |

**📝 NOTE:** Functions is not a containerized solution.

# [Containers](https://learn.microsoft.com/en-us/azure/containers/)

## [Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/)

Endpoint: `<registry>.azurecr.io/<repository>/<image-or-artifact>:<tag>`

`<repository>` is also known as `<namepsace>`. It allows sharing a single registry across multiple groups within your organization. Can be multiple levels deep. Optional.

## [Working with Azure Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli)

```sh
# Login to manage resources
az login

# Create a resource group
az group create --name $resourceGroup --location eastus

# Create Azure Container Registry
## https://learn.microsoft.com/en-us/azure/container-registry/container-registry-skus
## --sku {Basic,Standard,Premium} # 10, 100, 500GB; 💎: Concurrent operations, High volumes (⚡), Customer-Managed Key, Content trust for image tag signing, Private link
## Throttling: May happen if you exceed the registry's limits, causing temporary `HTTP 429` errors and requiring retry logic or reducing the request rate.
##
## [--default-action {Allow, Deny}] # 💎: Default action when no rules apply
##
## https://learn.microsoft.com/en-us/azure/container-registry/zone-redundancy
## [--zone-redundancy {Disabled, Enabled}] # 💎: Min 3 separate zones in each enabled region. The environment must include a virtual network (VNET) with an available subnet.
az acr create --resource-group $resourceGroup --name $registryName --sku Standard # ⭐: Production
# NOTE: High numbers of repositories and tags can impact the performance. Periodically delete unused.

# ACR Login: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication
## - Interactive: Individual Entra ID login, Admin Account
## - Unattended = Headless = Non-Interactive: Entra ID Service Principal, Managed Identity for Azure Resources
## Roles: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-roles?tabs=azure-cli
##
## 1) Individual login with Entra ID: Interactive push/pull by developers, testers.
## az login - provides the token. It has to be renewed every 3 hours
az acr login --name "$registryName" # Token must be renewed every 3 hours.
##
## 2) Entra ID Service Principal: Unattended push/pull in CI/CD pipelines
### Create service principal
#### Method 1: Short version that will setup and return appId and password in JSON format
az ad sp create-for-rbac --name $ServicePrincipalName --role AcrPush,AcrPull,AcrDelete --scopes /subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.ContainerRegistry/registries/$registryName
#### Method 2: Create a service principal and configure roles separately
az ad sp create --id $ServicePrincipalName
az role assignment create --assignee $appId --role AcrPush,AcrPull,AcrDelete --scope /subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.ContainerRegistry/registries/$registryName
az ad sp credential reset --name $appId # for method 2 password is not explicitly created, so we need to create (reset) it
#### Note: Password expires in 1 year.
az acr login --name $registryName --username $appId --password $password
##
## 3) Managed identities
az role assignment create --assignee $managedIdentityId --scope $registryName --role AcrPush,AcrPull,AcrDelete
## Now container instances / apps must use that managed identity to access this ACR (pull or push images)
##
## 4) Admin User: ❌ NOT RECOMMENDED. Individual identity is recommended for users and service principals for headless scenarios.
az acr update -n $registryName --admin-enabled true # this is disabled by default
docker login $registryName.azurecr.io

# Tasks: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview
## [--platform Linux {Linux, Windows}] # Linux supports all architectures (ex: Linux/arm), Windows: only amd64 (ex: Windows/amd64) - arch is optional
##
## - Quick task
az acr build --registry $registryName --image $imageName:$tag . # docker build, docker push
az acr run --registry $registryName --cmd '$registryName/$repository/$imageName:$tag' /dev/null # Run image (last param is source location, optional for non-image building tasks)
##
## - Automatically Triggered Task
### [--<operation>-trigger-enabled true] # CI on commit or pull-request
### [--schedule] # CRON schedule (⭐: OS/framework patching): https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-scheduled
az acr task create --name ciTask --registry $registryName --image $imageName:{{.Run.ID}} --context https://github.com/myuser/myrepo.git --file Dockerfile --git-access-token $GIT_ACCESS_TOKEN
az acr task create --name cmdTask --registry $registryName --cmd mcr.microsoft.com/hello-world --context /dev/null
### az acr task run --name mytask --registry $registryName # manually run task
##
## - Multi-step Task: granular control (build, push, when, cmd defined as steps) - https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-reference-yaml
### NOTE: --file is used for both multi-step task and Dockerfile
az acr run --file multi-step.yaml https://github.com/Azure-Samples/acr-tasks.git
az acr task create --file multi-step.yaml --name ciTask --registry $registryName --image $imageName:{{.Run.ID}} --context https://github.com/myuser/myrepo.git --git-access-token $GIT_ACCESS_TOKEN
```

Push and run a Docker image:

```sh
# Push image to registry
## Pull 'hello-world' image from Microsoft's Registry
docker pull mcr.microsoft.com/hello-world
## Tag the image
docker tag mcr.microsoft.com/hello-world $registryName.azurecr.io/$repository/$imageName:$tag
## Push image
docker push $registryName.azurecr.io/$repository/$imageName:$tag

# Run image from registry
docker run $registryName.azurecr.io/$repository/$imageName:$tag
# Alt: az acr run --registry $registryName --cmd '$registryName/$repository/$imageName:$tag' /dev/null
```

List container images and tags:

```sh
az acr repository list --name $registryName --output table
az acr repository show-tags --name $registryName --repository $repository --output table
```

## [Container Instances](https://learn.microsoft.com/en-us/azure/container-instances/)

Enables the deployment of Docker containers (up to 15 GB) without provisioning virtual machines.

**Does not support scaling! Use Container Apps for that!**

NB: If a container group restarts, its IP might change. Avoid using hardcoded IP addresses. For a stable public IP, consider [using Application Gateway](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-application-gateway).

## Working with Azure Container Instances

```sh
# Login to manage resources
az login

# Create a resource group
az group create --name $resourceGroup --location eastus

# (Optional)

# Deployment
##
## NOTE: If using managed identities with ACR, you'll also need --asign-identity param
## or az container identity assign --identities $identityName --resource-group $resourceGroup --name $containerName
##
## From image - simple scenarios
###
### Azure File share: https://learn.microsoft.com/en-us/azure/container-instances/container-instances-volume-azure-files
### Can only be mounted to Linux containers running as root!
### --os-type Linux
### --azure-file-volume-account-name # Azure File Share requires existing storage account and account key
### --azure-file-volume-account-key
### --azure-file-volume-mount-path
### --azure-file-volume-share-name
### NOTE: No direct integration Blob Storage because it lacks SMB support
###
### Public DNS name (must be unique) - accessible from $dnsLabel.<region>.azurecontainer.io
### --dns-name-label $dnsLabel
### --ip-address public
###
### [--restart-policy {Always, Never, OnFailure}] # Default: Always. Never if you only want to run once. Status when stopped: Terminated
###
### Environment variables: https://learn.microsoft.com/en-us/azure/container-instances/container-instances-environment-variables
#### NOTE: Format can be 'key'='value', key=value, 'key=value'
### --environment-variables # ex: 'PUBLIC_ENV_VAR'='my-exposed-value'
### --secure-environment-variables # ex: 'SECRET_ENV_VAR'='my-secret-value' - not visible in your container's properties
###
### Mount secret volumes: https://learn.microsoft.com/en-us/azure/container-instances/container-instances-volume-secret
### --secrets mysecret1="My first secret FOO" mysecret2="My second secret BAR"
### --secrets-mount-path /mnt/secrets
### NB: Restricted to Linux containers
### NOTE: This creates mysecret1 and mysecret2 files in /mnt/secrets with value the content of the secret
###
az container create --name $containerName --image $imageName:$tag --resource-group $resourceGroup
##
## From YAML file - deployment includes only container instances
### Same options as from simple deployment, but in a YAML file. Includes container groups.
az container create --name $containerName --file deploy.yml --resource-group $resourceGroup
##
## ARM template - deploy additional Azure service resources (for example, an Azure Files share)
### No example, but it's good to know this fact

# Verify container is running
az container show --name $containerName --resource-group $resourceGroup --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" --out table
```

YAML deployment (`deploy.yml`) (see CLI example above for reference):

```yml
apiVersion: "2019-12-01"
location: eastus
name: containerName
properties:
  # Container groups: https://learn.microsoft.com/en-us/azure/container-instances/container-instances-container-groups
  # Containers use a single host machine, sharing lifecycle, resources, network (share an external IP, ports. DNS), and storage volumes
  # For Windows containers, only single-instance deployment are allowed (NOTE: Here we use two!)
  # The resources allocated for the host are sum of all resources requested (In this case: 2 CPUs and 2.5 GB RAM)
  containers:
    - name: helloworld
      properties:
        environmentVariables:
          - name: "PUBLIC_ENV_VAR"
            value: "my-exposed-value"

          - name: "SECRET_ENV_VAR"
            secureValue: "my-secret-value"
        image: mcr.microsoft.com/hello-world
        ports:
          - port: 443
        resources:
          requests:
            cpu: 1.0
            memoryInGB: 1
        volumeMounts:
          - mountPath: /mnt/secrets
            name: secretvolume
    - name: hellofiles
      properties:
        environmentVariables: []
        image: mcr.microsoft.com/azuredocs/aci-hellofiles
        ports:
          - port: 80
        resources:
          requests:
            cpu: 1.0
            memoryInGB: 1.5
        volumeMounts:
          - mountPath: /aci/logs/
            name: filesharevolume
  osType: Linux # or Windows (for single containers)
  restartPolicy: Always
  ipAddress:
    type: Public
    ports:
      - port: 443
      - port: 80
    dnsNameLabel: containerName
  volumes:
    - name: filesharevolume
      azureFile:
        sharename: acishare
        storageAccountName: <Storage account name>
        storageAccountKey: <Storage account key>
    - name: secretvolume
      secret:
        # NB: The secret values must be Base64-encoded!
        mysecret1: TXkgZmlyc3Qgc2VjcmV0IEZPTwo= # "My first secret FOO"
        mysecret2: TXkgc2Vjb25kIHNlY3JldCBCQVIK # "My second secret BAR"
tags: {}
type: Microsoft.ContainerInstance/containerGroups
```

### Note for Azure File Shares

**Azure Container Instances does not support direct integration Blob Storage** because it lacks SMB support.

To use Azure File Share, you need to:

- Create the storage account
- Create a file share
- From storage account, you need Storage account name, Share name, and Storage account key.

### Container Instances Diagnostics and Logging

`az container attach` Connects your local console to a container's output and error streams in **real time** (example: to debug startup issue).

`az container logs` Displays logs (when no real time monitoring is needed)

## [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/)

Fully managed (no need to manage other Azure infrastructure) environment. Common use cases:

- Deploying API endpoints
- Hosting background processing applications
- Handling event-driven processing
- Running microservices

Limitations:

- Cannot run privileged containers (no root).
- linux/amd64 container images are required.
- State doesn't persist inside a container due to regular restarts. Use external caches for in-memory cache requirements.

A webhook can be used to notify Azure Container Apps when a new image has been pushed to the ACR, triggering automatic deployment of the updated container.

### Criteria for using multiple environments

- Different virtual networks: VNet is scoped at the environment level.
- Different Log Analytics workspaces: Diagnostics settings are tied to the environment.
- Different regional deployments (i.e. apps in different Azure regions): An environment is region-bound.
- Different managed identities for environment-level operations (rare): Only if using environment-scope identity.

### [Auth](https://learn.microsoft.com/en-us/azure/container-apps/authentication)

The platform's authentication and authorization middleware component runs as a _sidecar_ container on each application replica, screening all incoming HTTPS (ensure `allowInsecure` is _disabled_ in ingress config) requests before they reach your application. [See more](./App%20Service%20Web%20Apps.md). It requires any identity provider and specified provider within app settings.

### [Managed identities](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity)

Main topic: [Managed Identities](./Managed%20Identities.md)

Not supported in scaling rules.

### [Scaling](https://learn.microsoft.com/en-us/azure/container-apps/scale-app)

Scaling is driven by three different categories of triggers:

- HTTP: Based on the number of concurrent HTTP requests to your revision.
- TCP: Based on the number of concurrent TCP connections to your revision.
- Custom: Based on CPU, memory (_cannot scale to 0_), or supported event-driven data sources such as:
  - Azure Service Bus
  - Azure Event Hubs
  - Apache Kafka
  - Redis

As a container app revision scales out, new instances of the revision are created on-demand. These instances are known as replicas (default: 0-10). Adding or editing scaling rules creates a new revision of the container app. In "multiple revisions" mode, adding a new scale trigger creates a new revision of your application but your old revision remains available with the old scale rules.

Example:

```sh
az containerapp create \
 # ...
 --min-replicas 0 \
 --max-replicas 5 \

 #  HTTP Scaling Rule
 --scale-rule-name http-rule-name \
 --scale-rule-type http \
 --scale-rule-http-concurrency 100

 # TCP Scaling Rule
 --scale-rule-name tcp-rule-name \
 --scale-rule-type tcp \
 --scale-rule-tcp-concurrency 100

 # Custom Scaling rule
 ## Note we use --secrets to define the connection string and reuse it by secret name in --scale-rule-auth
  --secrets "connection-string-secret=<SERVICE_BUS_CONNECTION_STRING>" \
 --scale-rule-auth "connection=connection-string-secret"
 --scale-rule-name servicebus-rule-name \
 --scale-rule-type azure-servicebus \
 --scale-rule-metadata "queueName=my-queue" "namespace=service-bus-namespace" "messageCount=5"
```

Without a scale rule, the default (HTTP, 0-10 replicas) applies to your app. Create a rule or set minReplicas to 1+ if ingress is disabled. Without minReplicas or a custom rule, your app can scale to zero and won't start.

### [Revisions](https://learn.microsoft.com/en-us/azure/container-apps/revisions)

Immutable snapshots of a container app version. The first revision is auto-created upon deployment, new are created on revision scope changes. Up to 100 revisions can be stored for history. Multiple revisions can run at once, with HTTP traffic split among them.

- **Single revision mode**: keeps the old revision active until the new one is ready.
- **Multiple revision mode**: you control revision lifecycle and traffic distribution (via ingress), with traffic only switching to the latest revision when it's ready.

Revision Labels: direct traffic to specific revisions. A label provides a unique URL that you can use to route traffic to the revision that the label is assigned.

Scopes:

- Revision-scope changes via `az containerapp update` trigger a new revision when you deploy your app. Trigger: changing [properties.template](https://learn.microsoft.com/en-us/azure/container-apps/azure-resource-manager-api-spec?tabs=arm-template#propertiestemplate). Example: version suffix, container config, scaling rules. The changes don't affect other revisions.
- Application-scope changes are globally applied to all revisions. A new revision isn't created Trigger: changing [properties.configuration](https://learn.microsoft.com/en-us/azure/container-apps/azure-resource-manager-api-spec?tabs=arm-template#propertiesconfiguration). Example: secrets, revision mode, ingress, credentials, DARP settings.

### [Secrets](https://learn.microsoft.com/en-us/azure/container-apps/manage-secrets)

Secrets are scoped to an application (`az containerapp create`), outside of any specific revision of an application. Once secrets are defined at the application level, secured values are available to container apps. Adding, removing, or changing secrets doesn't generate new revisions. Apps need to be restarted to reflect updates.

Defining secrets: `--secrets "queue-connection-string=<CONNECTION_STRING>"`

Secrets from Key Vault: `--secrets "kv-connection-string=keyvaultref:<KEY_VAULT_SECRET_URI>,identityref:<USER_ASSIGNED_IDENTITY_ID>"`

Mounting Secrets in a Volume (secret name is filename, secret value is content): `--secret-volume-mount "/mnt/secrets"`

Referencing Secrets in Environment Variables (`secretref:`): `--env-vars "QueueName=myqueue" "ConnectionString=secretref:queue-connection-string"`

### [Logging](https://learn.microsoft.com/en-us/azure/container-apps/log-monitoring)

- System Logs (at the container app level)
- Console Logs (from the `stderr` and `stdout` messages inside container app)

#### Query Log with Log Analytics

```sh
# ContainerAppConsoleLogs_CL or ContainerAppSystemLogs_CL
az monitor log-analytics query --workspace $WORKSPACE_CUSTOMER_ID --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == 'album-api' | project Time=TimeGenerated, AppName=ContainerAppName_s, Revision=RevisionName_s, Container=ContainerName_s, Message=Log_s, LogLevel_s | take 5" --out table
```

### [Deployment](https://learn.microsoft.com/en-us/azure/container-apps/get-started)

- `az containerapp create` - Creates a new container app in Azure with specific configurations (CPU, memory, environment variables, etc).
- `az containerapp up` - Quicker and more automated deployment process, ideal for development or testing.

If you anticipate needing more control or specific configurations in the future, `az containerapp create` might be the more suitable choice. If simplicity and speed are the primary concerns, `az containerapp up` might be preferred.

```sh
# https://learn.microsoft.com/en-us/azure/container-apps/containerapp-up

# Upgrade Azure CLI version on the workstation
az upgrade

# Add and upgrade the containerapp extension for managing containerized services
az extension add --name containerapp --upgrade

# Login to Azure
az login

# Register providers for Azure App Services (for hosting APIs) and Azure Operational Insights (for telemetry)
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights

# Create an environment 'prod' in Azure Container Apps
az containerapp env create --resource-group $resourceGroup --name prod

# Deploy the API service to the 'prod' environment, using the source code from a repository
# https://learn.microsoft.com/en-us/azure/container-apps/quickstart-code-to-cloud
function deploy_repo() {
  az containerapp up \
    --name MyAPI \
    --resource-group $resourceGroup \
    --location eastus \
    --environment prod \
    --context-path ./src \
    --repo myuser/myrepo \
    --ingress 'external'

  # Display the Fully Qualified Domain Name (FQDN) of the app after it's deployed. This is the URL you would use to access your application.
  az containerapp show --name MyAPI --resource-group $resourceGroup --query properties.configuration.ingress.fqdn
}

# Deploy a containerized application in Azure Container Apps, using an existing public Docker image
# https://learn.microsoft.com/en-us/azure/container-apps/get-started
function deploy_image() {
  az containerapp up \
    --name MyContainerApp \
    --resource-group $resourceGroup \
    --environment prod \
    --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
    --target-port 80 \
    --ingress 'external' \ # allows the application to be accessible from the internet.
    # Display the Fully Qualified Domain Name (FQDN) of the app after it's deployed. This is the URL you would use to access your application.
    --query properties.configuration.ingress.fqdn

    # Alt: Deploy from a Docker Image in Azure Container Registry (ACR)
    # --image myAcr.azurecr.io/myimage:latest \
    # --registry-username myAcrUsername \
    # --registry-password myAcrPassword \
}
```

### [Disaster and Recovery](https://learn.microsoft.com/en-us/azure/container-apps/disaster-recovery?tabs=azure-cli)

In the event of a full region outage, you have two strategies:

- **Manual recovery**:
  - Manually deploy to a new region
  - Wait for the region to recover, and then manually redeploy all environments and apps.
- **Resilient recovery**: Deploy your container apps in advance to multiple regions. Use a traffic management service (ex: Azure Front Door) to direct requests to your main region. In case of an outage, reroute traffic from the affected region.

### [Dapr integration with Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/dapr-overview)

Dapr is activated per container app. Its APIs are accessible via a Dapr sidecar using HTTP or gRPC. Dapr's modular design allows shared or app-specific components, which can connect to external services and securely access configuration metadata. By default Dapr-enabled container apps load the full set of deployed components. To load components only for the right apps, application scopes are used.

Enable Dapr: `az containerapp create --dapr-enabled ...`

Main APIs provided by Dapr:

- [**Service-to-service invocation**](https://docs.dapr.io/developing-applications/building-blocks/service-invocation/service-invocation-overview/): Enables secure, direct service calls.
- [**State management**](https://docs.dapr.io/developing-applications/building-blocks/state-management/state-management-overview/): Manages transactions and CRUD operations.
- [**Pub/sub**](https://docs.dapr.io/developing-applications/building-blocks/pubsub/pubsub-overview): Facilitates communication between container apps via message broker. For event-driven architecture.
- [**Bindings**](https://docs.dapr.io/developing-applications/building-blocks/bindings/bindings-overview/): Communicate with external systems.
- [**Actors**](https://docs.dapr.io/developing-applications/building-blocks/actors/actors-overview/): Supports scalable, message-driven units of work.
- [**Observability**](https://learn.microsoft.com/en-us/azure/container-apps/observability): Sends tracing information to an Application Insights backend.
- [**Secrets**](https://docs.dapr.io/developing-applications/building-blocks/secrets/secrets-overview/): Accesses secrets or references secure values in Dapr components.

## Docker

- `dotnet/core/sdk` - build an ASP.NET app
- `dotnet/core/aspnet` - run an ASP.NET app

```Dockerfile
 # Sets the working directory to `/app`, which is where app files will be copied.
WORKDIR /app
# Copies the contents of the published app to the container's working directory (`/app` in this case).
COPY bin/Release/net6.0/publish/ .
```

### [Multi-stage](https://docs.docker.com/build/building/multi-stage/)

- Compile Stage:
  - Choose a base image suitable for compiling the code.
  - Set the working directory.
  - Copy the source code.
  - Compile the code.
- Runtime Stage:
  - Choose a base image suitable for running the application.
  - Copy compiled binaries or artifacts from the compile stage.
  - Set the command to run the application.

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.0 AS build
WORKDIR /app

# copy csproj and restore as distinct layers
COPY *.sln .
COPY aspnetapp/*.csproj ./aspnetapp/
RUN dotnet restore

# copy everything else and build app
COPY aspnetapp/. ./aspnetapp/
WORKDIR /app/aspnetapp
RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/core/aspnet:3.0 AS runtime
WORKDIR /app
COPY --from=build /app/aspnetapp/out ./
ENTRYPOINT ["dotnet", "aspnetapp.dll"]
```

[Serving both secure and non-secure web traffic](https://learn.microsoft.com/en-us/visualstudio/containers/container-tools?view=vs-2019#dockerfile-overview):

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

# copy csproj and restore as distinct layers
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build
WORKDIR /src
COPY ["WebApplication1/WebApplication1.csproj", "WebApplication1/"]
RUN dotnet restore "WebApplication1/WebApplication1.csproj"

# copy everything else and build app
COPY . .
WORKDIR "/src/WebApplication1"
RUN dotnet build "WebApplication1.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "WebApplication1.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "WebApplication1.dll"]
```

## CLI

| Command                                                                                                                                                    | Brief Explanation                                                                   | Example                                                                                                |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| [az acr login](https://learn.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az-acr-login)                                                         | Authenticate with an ACR.                                                           | `az acr login --name MyRegistry`                                                                       |
| [az acr create](https://learn.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az-acr-create)                                                       | Create a new ACR.                                                                   | `az acr create --resource-group $resourceGroup --name MyRegistry --sku Basic`                          |
| [az acr update](https://learn.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az-acr-update)                                                       | Update properties of an ACR.                                                        | `az acr update --name MyRegistry --tags key=value`                                                     |
| [az acr build](https://learn.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az-acr-build)                                                         | Build a container image in ACR.                                                     | `az acr build --image MyImage:tag --registry MyRegistry .`                                             |
| [az acr task create](https://learn.microsoft.com/en-us/cli/azure/acr/task?view=azure-cli-latest#az-acr-task-create)                                        | Create a task for an ACR.                                                           | `az acr task create --registry MyRegistry --name MyTask --image MyImage:tag --context /path/to/source` |
| [az acr repository](https://learn.microsoft.com/en-us/cli/azure/acr/repository?view=azure-cli-latest)                                                      | Manage repositories (image storage) in ACR.                                         | `az acr repository show-tags --name MyRegistry --repository MyImage`                                   |
| [`az acr repository list`](https://learn.microsoft.com/en-us/cli/azure/acr/repository?view=azure-cli-latest#az-acr-repository-list)                        | List repositories / Verify an image has been pushed to ACR                          | `az acr repository list --name MyRegistry`                                                             |
| [az acr run](https://learn.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az-acr-run)                                                             | Queue a run to stream logs for an ACR.                                              | `az acr run --registry MyRegistry --cmd '$Registry/myimage' /dev/null`                                 |
| [az acr show](https://learn.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az-acr-show)                                                           | Get the details of an ACR.                                                          | `az acr show --name MyRegistry --query "loginServer"`                                                  |
| [az container create](https://learn.microsoft.com/en-us/cli/azure/container?view=azure-cli-latest#az-container-create)                                     | Create a container group in ACI (deploy an image).                                  | `az container create --name MyContainer --image myimage:latest`                                        |
| [az container attach](https://learn.microsoft.com/en-us/cli/azure/container?view=azure-cli-latest#az-container-attach)                                     | Attach local standard output and error streams to a container in a container group. | `az container attach --name MyContainer --resource-group $resourceGroup`                               |
| [az container show](https://learn.microsoft.com/en-us/cli/azure/container?view=azure-cli-latest#az-container-show)                                         | Get the details of a container group.                                               | `az container show --name MyContainer --resource-group $resourceGroup`                                 |
| [az container logs](https://learn.microsoft.com/en-us/cli/azure/container?view=azure-cli-latest#az-container-logs)                                         | Fetch the logs for a container in a container group.                                | `az container logs --name MyContainer --resource-group $resourceGroup`                                 |
| [az containerapp create](https://learn.microsoft.com/en-us/cli/azure/containerapp?view=azure-cli-latest#az-containerapp-create)                            | Create a Container App.                                                             | `az containerapp create --name MyContainerApp --resource-group $resourceGroup --image myimage:latest`  |
| [az containerapp up](https://learn.microsoft.com/en-us/cli/azure/containerapp?view=azure-cli-latest#az-containerapp-up)                                    | Create or update a Container App and associated resources.                          | `az containerapp up --name MyContainerApp`                                                             |
| [az containerapp env create](https://learn.microsoft.com/en-us/cli/azure/containerapp/env?view=azure-cli-latest#az-containerapp-env-create)                | Create an environment for a Container App.                                          | `az containerapp env create --name MyEnvironment --resource-group $resourceGroup`                      |
| [az containerapp show](https://learn.microsoft.com/en-us/cli/azure/containerapp?view=azure-cli-latest#az-containerapp-show)                                | Show details of a Container App.                                                    | `az containerapp show --name MyContainerApp --resource-group $resourceGroup`                           |
| [az containerapp identity assign](https://learn.microsoft.com/en-us/cli/azure/containerapp/identity?view=azure-cli-latest#az-containerapp-identity-assign) | Assign managed identities to a Container App.                                       | `az containerapp identity assign --name MyContainerApp --identities [system]`                          |
| [az upgrade](https://learn.microsoft.com/en-us/cli/azure/reference-index?view=azure-cli-latest#az-upgrade)                                                 | Upgrade Azure CLI and extensions.                                                   | `az upgrade`                                                                                           |
| [az identity create](https://learn.microsoft.com/en-us/cli/azure/identity?view=azure-cli-latest#az-identity-create)                                        | Create a managed identity.                                                          | `az identity create --name MyManagedIdentity --resource-group $resourceGroup`                          |
| [az role assignment create](https://learn.microsoft.com/en-us/cli/azure/role/assignment?view=azure-cli-latest#az-role-assignment-create)                   | Create a new role assignment for a user, group, or service principal.               | `az role assignment create --assignee john.doe@domain.com --role Reader`                               |
| [az ad sp](https://learn.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest)                                                                        | Manage Microsoft Entra ID service principals.                                       | `az ad sp create-for-rbac --name MyServicePrincipal`                                                   |
| [az monitor log-analytics query](https://learn.microsoft.com/en-us/cli/azure/monitor/log-analytics?view=azure-cli-latest#az-monitor-log-analytics-query)   | Query a Log Analytics workspace.                                                    | `az monitor log-analytics query --workspace MyWorkspace --query 'MyQuery'`                             |
| [az acr import](https://learn.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az-acr-import)                                                       | Import an image to an ACR from another registry.                                    | `az acr import --name MyRegistry --source myregistry.azurecr.io/myimage:tag`                           |
| [az containerapp revision list](https://learn.microsoft.com/en-us/cli/azure/containerapp?view=azure-cli-latest#az-containerapp-revision-list)              | List the revisions of a Container App.                                              | `az containerapp revision list --name MyContainerApp --resource-group $resourceGroup`                  |

# [Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/introduction)

`https://<account-name>.documents.azure.com:443/` (SQL)

- **Global Distribution:** ⚡ and 99.999% availability for multi-region databases.
- **Data Models:** Supports Key-Value, Document, Column-family, and Graph formats.
- **Scalability:** Scales throughput and storage elastically as needed.
- **Consistency:** Choose trade-off between data accuracy and performance.
- **SLAs:** Covers latency, throughput, consistency, and availability.

## [Use Cases](https://docs.microsoft.com/en-us/azure/cosmos-db/use-cases)

- IoT: Handles massive real-time data and traffic spikes.
- Retail: Offers low-latency data access worldwide.
- Gaming: Scales for millions of players with synchronized game states.
- Analytics: Balances performance and consistency for real-time analysis.
- Personalization: query user activity data to drive real-time, personalized experiences

## [Core Components](https://docs.microsoft.com/en-us/azure/cosmos-db/databases-containers-items)

![Image showing the hierarchy of Azure Cosmos DB entities: Database accounts are at the top, Databases are grouped under accounts, Containers are grouped under databases.](https://learn.microsoft.com/en-us/training/wwl-azure/explore-azure-cosmos-db/media/cosmos-entities.png)

- [**Cosmos DB Account**](https://docs.microsoft.com/en-us/azure/cosmos-db/manage-account): Manages databases like storage accounts manage containers. Free tier availability is limited to one per subscription.
- **Databases**: Serves as a _namespace_, managing containers, users, permissions, and consistency levels.
- **Containers**: Data is stored on one or more servers, called partitions. Partition keys are used for efficient data distribution. Containers are schema-agnostic.
- **Items**: Smallest read/write data units. Schema-agnostic and API-dependent, they can represent documents, rows, nodes, or edges. Automatically indexed in containers without explicit management.

**📝 NOTE:** Partition keys always start with `/` (ex: `/partitionkey`). Min length for all names is 3 chars, alphanumeric.

### [Time to live (TTL)](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/time-to-live)

Auto-deletes items after a set time (seconds) from last modification. Configurable at container or item level; item settings take precedence. Deletion consumes spare RUs; low RUs cause deletion delays but expired data is not shown in queries. No container-level TTL or a value of -1 prevents auto-expiration unless item-specific TTL exists. Use `DefaultTimeToLive` in `ContainerProperties` to set TTL in C#.

### Examples

[Azure CLI](https://learn.microsoft.com/en-us/cli/azure/cosmosdb?view=azure-cli-latest):

```sh
# Create a Cosmos DB account
az cosmosdb create --name $account --kind GlobalDocumentDB ...

# Create a database
az cosmosdb sql database create --account-name $account --name $database

# Create a container
az cosmosdb sql container create --account-name $account --database-name $database --name $container --partition-key-path "/mypartitionkey"

# Create an item
az cosmosdb sql container item create --account-name $account --database-name $database --container-name $container --value "{\"id\": \"1\", \"mypartitionkey\": \"mypartitionvalue\", \"description\": \"mydescription\"}"
```

[Azure Cosmos DB .NET SDK](https://learn.microsoft.com/en-us/dotnet/api/microsoft.azure.cosmos?view=azure-dotnet):

```cs
var cosmosClient = new CosmosClient("<connection-string>"); // credentials or/and options

// Create a database
Database database = await cosmosClient.CreateDatabaseIfNotExistsAsync("<database>");
// NOTE: You can use DatabaseResponse instead of Database if you need information like status codes or headers.

// database = await database.ReadAsync(); // Re-fetch data

// Create a container
Container container = await database.CreateContainerIfNotExistsAsync(id: "<container>", partitionKeyPath: "/mypartitionkey", throughput: number);

ItemResponse<dynamic> response = await container.ReadItemAsync<dynamic>(id, new PartitionKey(partitionKey));

// Create an item (note that mypartitionkey doesn't contain leading slash (/))
dynamic testItem = new { id = "1", mypartitionkey = "mypartitionvalue", description = "mydescription" };
await container.CreateItemAsync(testItem, new PartitionKey(testItem.mypartitionkey));

// NOTE: This example is for another container
QueryDefinition query = new QueryDefinition(
    "select * from sales s where s.AccountNumber = @AccountInput ")
    .WithParameter("@AccountInput", "Account1");

string query = $@"
  SELECT VALUE products
  FROM models
  JOIN products in models.Products
  WHERE products.id = '{id}'
";
FeedIterator<SalesOrder> rs = container.GetItemQueryIterator<SalesOrder>(
    query,
    // Optional:
    // requestOptions: new QueryRequestOptions()
    // {
    //     PartitionKey = new PartitionKey("Account1"),
    //     MaxItemCount = 1
    // }
);
while (rs.HasMoreResults)
{
    Model next = await iterator.ReadNextAsync();
    matches.AddRange(next);
}
```

## [Consistency Levels](https://docs.microsoft.com/en-us/azure/cosmos-db/consistency-levels)

Azure Cosmos DB provides a balanced approach to consistency, availability, and latency. Its globally linearizable consistency (serving requests concurrently) levels remain unaffected by regional settings, operation origins, number of associated regions, or whether your account has one or multiple write regions. 'Read consistency' specifically denotes a singular read operation, scoped within a particular partition-key range or logical partition, initiated by either a remote client or a stored procedure.

![Image showing data consistency as a spectrum.](https://learn.microsoft.com/en-us/training/wwl-azure/explore-azure-cosmos-db/media/five-consistency-levels.png)

- **Strong** - Every reader sees the latest data right away, but it may be slower (higher latency) and less available. ⭐: serious stuff like bank transactions where accuracy is vital. Clients will never receive a partial write.
- **Bounded staleness** - Readers may see slightly old data, but there's a limit on how old (determined by number of operations K or the time interval T for staleness - in single-region accounts, the minimum values are 10 write operations or 5 seconds, while in multi-region accounts, they are 100,000 write operations or 300 seconds). Good for things like game leaderboards where a small delay is okay.
- **Session** (⏺️, can be configured) - Within a single user's session, the data is always up-to-date. Great for scenarios like a personal shopping cart on a website.
- **Consistent prefix** - Readers might be a bit behind, but they always see things in order. Update operations made as a batch within a transaction are always visible together. Good for situations where sequence matters, like following a chain of events.
- **Eventual** - Readers might see things out of order (non-consistent) or slightly old, but it eventually catches up. ⭐: when speed and availability are more important than immediate consistency, like a social media like counter. Lowest latency and highest availability.

```sh
# Get the default consistency level
az cosmosdb show --name $database--query defaultConsistencyLevel ...

# Set the default consistency level
az cosmosdb update --name $database--default-consistency-level "BoundedStaleness" ...
```

```cs
ConsistencyLevel defaultConsistencyLevel = cosmosClient.ConsistencyLevel;
cosmosClient.ConsistencyLevel = ConsistencyLevel.BoundedStaleness;
```

## [Partitioning in Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/partitioning-overview)

Improves⚡. Enables independent scaling of storage and throughput.

- **Partition Key**: A property in data used to distribute data across partitions. Critical for performance and scalability. Type: string only.
- **Logical Partitions**: Sets of items with the same partition key. Enables efficient queries and transactions. Max storage: 20 GB; Max throughput: 10,000 RU/s.
- **Physical Partitions**: Internal resources hosting logical partitions. Cosmos DB auto-splits and merges these for workload balance. Max throughput: 10,000 RU/s.
- **Partition Sets**: Groups of physical partitions. Split into subsets for ⚡.

### [Partitioning Key Selection](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/model-partition-example#identify-the-main-access-patterns)

Use a key that appears often in your queries' `WHERE` clause and has many unique (distinct) values to avoid _'hot' partitions_ (overused, or uneven partition sizes) that could become a performance bottleneck. For instance, in multi-tenant apps, using the tenant ID as the key improves read/write speeds and data isolation. Validate your choice through Azure SDKs and monitor metrics like data size, throughput, and cost for any needed adjustments.

### [Synthetic partition key](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/synthetic-partition-keys)

When no property has many distinct values:

- Concatenate multiple properties of an item
- Use a partition key with a random suffix
- Use a partition key with pre-calculated suffixes

## [Request Units (RUs)](https://learn.microsoft.com/en-us/azure/cosmos-db/request-units)

RUs measure the cost of operations like read, write, and query. The cost varies based on factors like item size and query complexity. If an operation costs more RUs than you have, it's rate-limited until more RUs are available. Typically, reading a 1-KB document costs 1 RU.

![Image showing how database operations consume request units.](https://learn.microsoft.com/en-us/training/wwl-azure/explore-azure-cosmos-db/media/request-units.png)

### [Provisioned Throughput](https://learn.microsoft.com/en-us/azure/cosmos-db/set-throughput)

🧊. You set the number of Request Units (RUs) per second your application needs in multiples of 100. Minimum value: 400.

- **Container-level throughput** (**Dedicated**): Reserves a set amount of processing power (consistent ⚡) for _one container_ (guaranteed by [SLA](https://www.azure.cn/en-us/support/sla/cosmos-db/)). ⭐: 🏋🏿 workloads, large data volumes.
- **Database-level throughput** (**Shared**): _Multiple containers in the same database_ share processing power. Does not effect containers with dedicated throughput. 🏷️. ⭐: small to large workloads with light/variable traffic, multitenant applications.
- [**Serverless mode**](https://learn.microsoft.com/en-us/azure/cosmos-db/serverless): No need to set throughput in Azure Cosmos DB. You're billed for the RUs used by database operations at the end of the billing cycle.
- [**Autoscale Throughput**](https://learn.microsoft.com/en-us/azure/cosmos-db/provision-throughput-autoscale): Dynamically adjusts provisioned RUs based on current usage, applicable to both single containers and databases. It scales between 10% and 100% of a set maximum (100 to 1000 RU/s), optimizing for unpredictable traffic while maintaining high performance and scale SLAs.

  ```sh
  az cosmosdb sql container create --throughput-type autoscale --max-throughput 4000
  ```

  ```cs
  await this.cosmosDatabase.CreateContainerAsync(new ContainerProperties(id, partitionKeyPath), ThroughputProperties.CreateAutoscaleThroughput(1000));
  ```

## [API Models](https://learn.microsoft.com/en-us/azure/cosmos-db/choose-api)

All API models return JSON formatted objects, regardless of the API used.

- **API for NoSQL** - A JSON-based, document-centric API that provides SQL querying capabilities. ⭐: web, mobile, and gaming applications, and anything that requires handling complex _hierarchical data_ with a _schemaless_ design.
- **API for MongoDB** - stores data in a document structure, via BSON format.
- **API for PostgreSQL** - Stores data either on a single node, or distributed in a multi-node configuration.
- **API for Apache Cassandra** - Supports a column-oriented schema, aligns with Apache Cassandra's distributed, scalable NoSQL philosophy, and is wire protocol compatible with it.
- **API for Table** - A _key-value_ based API that offers compatibility with Azure Table Storage, overcoming its limitations. ⭐: applications that need a simple schemaless _key-value_ store. Only supports OLTP scenarios.
- **API for Apache Gremlin** - _Graph-based_ API for highly _interconnected_ datasets. ⭐: handling dynamic, complexly related data that exceeds the capabilities of relational databases.

Using these APIs, you can emulate various database technologies, modeling real-world data via documents, key-value, graph, and column-family models, minus the management and scaling overheads.

## [Stored procedures](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-use-stored-procedures-triggers-udfs?tabs=dotnet-sdk-v3)

Azure Cosmos DB allows transactional execution of JavaScript in the form of stored procedures, triggers, and user-defined functions (UDFs). Each needs to be registered prior to calling. They are created and managed in Azure Portal.

```js
// Gist
var context = getContext(); // root for container and response

var container = context.getCollection(); // for modifying container (collection)
// container.createDocument(container.getSelfLink(), documentToCreate, callback);
// container.queryDocuments(container.getSelfLink(), filterQueryString, updateMetadataCallback);
// container.replaceDocument(metadataItem._self, metadataItem, callback);

var response = context.getResponse(); // getting and setting current item
// response.getBody();
// request.setBody(itemToCreate);
```

- **Stored Procedures**: JavaScript functions registered to a collection (container), enabling CRUD and query tasks on its documents. They run within a time limit and support transactions via "continuation tokens". Functions like `collection.createDocument()` return a `Boolean` indicating operation success.

  ```js
  var helloWorldStoredProc = {
    id: "helloWorld",
    serverScript: function () {
      // context provides access to all operations that can be performed in Azure Cosmos DB
      // and access to the request and response objects
      var context = getContext();
      var response = context.getResponse();

      response.setBody("Hello, World");
    },
  };
  ```

  ```js
  var createDocumentStoredProc = {
    id: "createMyDocument",
    // This stored procedure creates a new item in the Azure Cosmos container
    body: function createMyDocument(documentToCreate) {
      var context = getContext();
      var container = context.getCollection();

      // Async 'createDocument' operation, depends on JavaScript callbacks
      // returns true if creation was successful
      var accepted = container.createDocument(
        container.getSelfLink(),
        documentToCreate,
        // Callback function with error and created document parameters
        function (err, documentCreated) {
          // Handle or throw error inside the callback
          if (err) throw new Error("Error" + err.message);
          // Return the id of the newly created document
          context.getResponse().setBody(documentCreated.id);
        }
      );

      // If the document creation was not accepted, return
      if (!accepted) return;
    },
  };
  ```

  ```js
  // Parameter is always string!
  function sample(arr) {
    if (typeof arr === "string") arr = JSON.parse(arr);

    arr.forEach(function (a) {
      // do something here
      console.log(a);
    });
  }
  ```

- **Triggers**: Pretriggers and post-triggers operate before and after a database item modification, respectively. They aren't automatically executed, and must be registered and specified for each operation where execution is required.

  - **Pre-triggers**: Can't have any input parameters. Validates properties of an item that is being created, modifies properties (ex: add a timestamp of an item to be created).
  - **Post-triggers**: Runs as part of the same transaction for the underlying item itself. Modifies properties (ex: update metadata of newly created item).

  ```js
  // Pretrigger
  function validateToDoItemTimestamp() {
    var context = getContext();
    var request = context.getRequest();

    // item to be created in the current operation
    var itemToCreate = request.getBody();

    // validate properties
    if (!("timestamp" in itemToCreate)) {
      var ts = new Date();
      itemToCreate["timestamp"] = ts.getTime();
    }

    // update the item that will be created
    request.setBody(itemToCreate);
  }

  // Posttrigger
  function updateMetadata() {
    var context = getContext();
    var container = context.getCollection();
    var response = context.getResponse();

    // item that was created
    var createdItem = response.getBody();

    // query for metadata document
    var filterQuery = 'SELECT * FROM root r WHERE r.id = "_metadata"';
    var accept = container.queryDocuments(
      container.getSelfLink(),
      filterQuery,
      updateMetadataCallback
    );
    if (!accept) throw "Unable to update metadata, abort";

    function updateMetadataCallback(err, items, responseOptions) {
      if (err) throw new Error("Error" + err.message);
      if (items.length != 1) throw "Unable to find metadata document";

      var metadataItem = items[0];

      // update metadata
      metadataItem.createdItems += 1;
      metadataItem.createdNames += " " + createdItem.id;
      var accept = container.replaceDocument(
        metadataItem._self,
        metadataItem,
        function (err, itemReplaced) {
          if (err) throw "Unable to update metadata, abort";
        }
      );
      if (!accept) throw "Unable to update metadata, abort";
      return;
    }
  }
  ```

- **User-defined functions (UDFs)**: These are functions used within a query, such as a function to calculate income tax based on different brackets.

  ```js
  function tax(income) {
    if (income == undefined) throw "no input";

    if (income < 1000) return income * 0.1;
    else if (income < 10000) return income * 0.2;
    else return income * 0.4;
  }
  ```

Cosmos DB operations must complete within a limited time frame.

### [Register and use stored procedures, triggers, and user-defined functions](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-use-stored-procedures-triggers-udfs)

Registering pre/post trigger uses `TriggerProperties` class with specific `TriggerType`. Calling them is a matter of using `ItemRequestOptions` with proper property and `new List<string> { "<name-of-registered-trigger>" } }`.

```cs
var container = await client.GetContainer("db", "container");

// Register Stored Procedure
container.Scripts.CreateStoredProcedureAsync(new StoredProcedureProperties { Id = "spCreateToDoItems", Body = File.ReadAllText("sp.js") });

// Call Stored Procedure
dynamic[] items = { new { category = "Personal", name = "Groceries" } };
container.Scripts.ExecuteStoredProcedureAsync<string>("spCreateToDoItem", new PartitionKey("Personal"), new[] { items });

// Register Pretrigger
container.Scripts.CreateTriggerAsync(new TriggerProperties { Id = "preTrigger", Body = File.ReadAllText("preTrigger.js"), TriggerOperation = TriggerOperation.Create, TriggerType = TriggerType.Pre });

// Call Pretrigger
dynamic newItem = new { category = "Personal", name = "Groceries" };
container.CreateItemAsync(newItem, null, new ItemRequestOptions { PreTriggers = new List<string> { "preTrigger" } });

// Register Post-trigger
container.Scripts.CreateTriggerAsync(new TriggerProperties { Id = "postTrigger", Body = File.ReadAllText("postTrigger.js"), TriggerOperation = TriggerOperation.Create, TriggerType = TriggerType.Post });

// Call Post-trigger
container.CreateItemAsync(newItem, null, new ItemRequestOptions { PostTriggers = new List<string> { "postTrigger" } });

// Register UDF
container.Scripts.CreateUserDefinedFunctionAsync(new UserDefinedFunctionProperties { Id = "Tax", Body = File.ReadAllText("Tax.js") });

// Call UDF
var iterator = container.GetItemQueryIterator<dynamic>("SELECT * FROM Incomes t WHERE udf.Tax(t.income) > 20000");

```

## [Azure Cosmos DB Change Feed](https://docs.microsoft.com/en-us/azure/cosmos-db/change-feed)

Enabled by default, tracking container changes chronologically (**order is guaranteed only per partition key**) but not deletions. For deletions, use a "deleted" attribute and set TTL. Azure Functions uses the change feed processor.

Azure Functions utilizes the change feed processor internally.  
Not compatible with Table and PostgreSQL databases.

Interaction Models:

- **Push Model**: Automatically sends updates to the client. ⭐.
- **Pull Model**: Requires manual client requests for updates. Useful for specialized tasks like data migration or controlling processing speed.

### Change feed processor

Host instance lifecycle: Reads change feed, sleeps if no changes, sends changes to delegate for processing, and updates lease store with latest processed time.

- Monitored container: has the data from which the change feed is generated. Inserts and updates are reflected in the change feed.
- Lease container: acts as a state storage and coordinate the processing of change feed across multiple workers. It can be in the same or different account as the monitored container.
- Delegate componet can be used to implement custom logic to process the changes that the change feed reads.
- Compute Instance: Hosts change feed processor to listen for changes. Can be a VM, Kubernetes pod, Azure App Service instance, or a physical machine.

```cs
private static async Task<ChangeFeedProcessor> StartChangeFeedProcessorAsync(CosmosClient cosmosClient)
{
    Container monitoredContainer = cosmosClient.GetContainer("databaseName", "monitoredContainerName");
    Container leaseContainer = cosmosClient.GetContainer("databaseName", "leaseContainerName");

    ChangeFeedProcessor changeFeedProcessor = monitoredContainer
        .GetChangeFeedProcessorBuilder<ToDoItem>(processorName: "changeFeedSample", onChangesDelegate: DelagateHandleChangesAsync)
            .WithInstanceName("consoleHost") // Compute Instance
            .WithLeaseContainer(leaseContainer)
            .Build();

    Console.WriteLine("Starting Change Feed Processor...");
    await changeFeedProcessor.StartAsync();
    Console.WriteLine("Change Feed Processor started.");
    return changeFeedProcessor;
}

//////////////////
// The delegate //
//////////////////
static async Task HandleChangesAsync(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<ToDoItem> changes,
    CancellationToken cancellationToken)
{
    Console.WriteLine($"Started handling changes for lease {context.LeaseToken}...");
    Console.WriteLine($"Change Feed request consumed {context.Headers.RequestCharge} RU.");
    // SessionToken if needed to enforce Session consistency on another client instance
    Console.WriteLine($"SessionToken ${context.Headers.Session}");

    foreach (ToDoItem item in changes)
    {
        Console.WriteLine($"Detected operation for item with id {item.id}.");
        await Task.Delay(10);
    }
}
```

## [Querying data](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/query/)

[More](https://cosmosdb.github.io/labs/dotnet/labs/03-querying_in_azure_cosmosdb.html)

- Data in Azure Cosmos DB is stored as JSON documents. This means you can query nested properties and arrays directly.
- Queries in Azure Cosmos DB are case sensitive.
- Azure Cosmos DB automatically **indexes all properties** in your data (configurable: `containerProperties.IndexingPolicy.ExcludedPaths.Add(new ExcludedPath{ Path = "/nonQueriedProperty/*" });` to exclude property you never query on). This makes queries fast, but it can also increase the cost of writes.
- Azure Cosmos DB supports ACID transactions within a single partition.
- Azure Cosmos DB uses optimistic concurrency control to prevent conflicts when multiple clients are trying to update the same data. This is done using `ETags` and HTTP headers.

```sql
SELECT
  c.name,
  c.age,
  {
    "phoneNumber": p.number,
    "phoneType": p.type
  } AS phoneInfo
FROM c
JOIN p IN c.phones
JOIN (SELECT VALUE t FROM t IN p.tags WHERE t.name IN ("winter", "fall"))
WHERE c.age > 21 AND ARRAY_CONTAINS(c.tags, 'student') AND STARTSWITH(p.number, '123')
ORDER BY c.age DESC
OFFSET 10 LIMIT 20

```

- Column names **require** specifiers, e.g. `c.name`, **not** `name`.
- `FROM` clause is just an alias (no need to specify container name).
- You can only join within a single container. You can't join data across different containers.
- Aggregation functions are not supported.

Queries that have an `ORDER BY` clause with two or more properties require a [composite index](https://learn.microsoft.com/en-us/azure/cosmos-db/index-policy):

```json
{
  "compositeIndexes": [
    [
      { "path": "/name", "order": "ascending" },
      { "path": "/age", "order": "descending" }
    ]
  ]
}
```

### Flattening data

The `VALUE` keyword is used to flatten the result of a `SELECT` statement (not individual fields within that statement)

```sql
SELECT VALUE {
  "name": p.name,
  "sku": p.sku,
  "vendor": p.manufacturer.name
}
FROM products p
WHERE p.sku = "teapo-surfboard-72109"
```

This is similar to aliasing `SELECT p.name AS name`

### Using UDF

```sql
SELECT c.id, udf.GetMaxNutritionValue(c.nutrients) AS MaxNutritionValue FROM c
```

### Get count

```sql
SELECT VALUE COUNT(1) FROM models
```

## [Connectivity modes](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/sdk-connection-modes)

- **Gateway mode** (⏺️): For environments that have a limited number of socket connections, or corporate network with strict firewall restrictions. It uses HTTPS port and a single DNS endpoint.

- **Direct mode**: Connectivity through TCP protocol, using TLS for initial authentication and encryption. ⚡

## Performance and best practices

- Latest SDK version
- Application must be in the same Azure region as the Cosmos DB account
- Use single instance of `CosmosClient`
- `Direct` mode for ⚡
- Leverage `PartitionKey` for efficient point reads and writes
- Retry logic for handling transient errors
- Read 🏋🏿: `Stream API` and `FeedIterator`
- Write 🏋🏿: Enable bulk support, set `EnableContentResponseOnWrite` to false, exclude unused paths from indexing and keep the size of your documents minimal

## [Global Distribution](https://docs.microsoft.com/en-us/azure/cosmos-db/distribute-data-globally)

- **Multi-region Writes** - Perform writes in all configured regions. This enhances write latency and availability.
- **Automatic Failover** - If an Azure region goes down, Cosmos DB can automatically failover to another region.
- **Manual Failover** - For testing purposes, you can trigger a manual failover to see how your application behaves during regional failures.
- **No downtime when adding or removing regions**

### [Conflict resolution](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-manage-conflicts)

🧊, conflict resolution policy can only be specified at container creation time.

- **Automatic**: Uses a Last Writer Wins (LWW) policy based on a system or user-defined property.
- **Custom**: Uses a user-defined stored procedure to handle conflicts.

## [Security](https://docs.microsoft.com/en-us/azure/cosmos-db/database-security)

- **Authentication**: Utilizes Entra and generates an access key upon account creation for request authentication.
- **Authorization**: Role-based with built-in resources for granular control over containers and documents.
- **Encryption**: 🔑 (⏺️) or 🗝️ for at-rest data; TLS for in-transit data.

## [Azure Cosmos DB Backup & Restore](https://docs.microsoft.com/en-us/azure/cosmos-db/online-backup-and-restore)

Regular automatic secure (🔑) backups without affecting performance. Restores are limited to accounts in the same subscription with equal or higher RU/s and partitions.

- **Continuous**: 7 or 30-day retention. Can't switch to periodic.
- **Periodic**: ⏺️. Custom intervals (min 1 hour), max retention: 1 month. Support team handles restores.

## [Monitoring and Diagnostics](https://docs.microsoft.com/en-us/azure/cosmos-db/how-to-monitor-account)

- Metrics: Built-in tracking for aspects like storage, latency, and availability.
- Azure Monitor: Enables custom dashboards, alerts, and metric visualization.
- Diagnostic Logs: Offers operation traces, which can be analyzed further in various Azure services.
- Query Stats: Provides execution metrics to help optimize and troubleshoot queries.

# [Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/what-is-entra#microsoft-entra-id)

Implements [OAuth 2.0](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols) authorization protocol, allowing third-party apps to access web-hosted resources on behalf of users. These resources have a unique _application ID URI_.

High-level view of relation between how the [`Azure roles`](https://learn.microsoft.com/en-us/azure/role-based-access-control/rbac-and-directory-admin-roles#azure-roles), [`Microsoft Entra roles`](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference), and `classic subscription administrator roles`.<br>
<img src="https://learn.microsoft.com/en-us/azure/role-based-access-control/media/rbac-and-directory-admin-roles/rbac-admin-roles.png" width="600">

## Microsoft Entra ID vs Role-Based Access Control (RBAC)

| Feature        | Entra ID                                                            | Azure RBAC                                                                                   |
| -------------- | ------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **Purpose**    | Authentication + Authorization(identity-related resources only)                        | Authorization                                                                                |
| **Focus**      | Manages identities (users, groups, devices, applications) and authenticates access to Microsoft 365, Azure services, and any application that supports modern identity protocols (SAML, OAuth, OpenID Connect)                                                 | Manages permissions for Azure resources                                                                              |
| **Scope**      | Tenant                                                              | Management Group, Subscription, Resource Group, Resource                                     |
| **Roles**      | Global, User, Billing Admins; Custom roles; Multiple roles per user | Owner, Contributor, Reader, User Access Admin; Custom roles (P1/P2); Multiple roles per user |
| **Access Via** | Azure Portal, MS 365 Admin, MS Graph, PowerShell                       | Azure Portal, CLI, PowerShell, ARM templates, REST API                                       |
| **Pricing**    | Free, P1, P2 (Monthly charged)                                      | Free (With Azure subscription)                                                               |

## Key Terms

### Application Object and Service Principal

- **[Application Object](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals?tabs=browser#application-object)**: The global template/representation of your application, residing in the Entra ID tenant where the app is registered. It has a 1:1 relationship with the software application and a 1:N relationship with service principal objects across different tenants.

- **[Service Principal (or Service Principal Object)](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals?tabs=browser#service-principal-object)**: The local instance (identity) created in each tenant that consumes your app. These are local representations within each tenant, derived from the application object. Managed via the **Enterprise applications** page in the Azure portal.

### Service Principal Types

| Type | Description | Key Characteristics |
|------|-------------|---------------------|
| Application | Local representation of a global application object in a specific tenant, created from the application object. | A service principal is created in each tenant where the app is used and defines what the app can do, who can access it, and what resources it can access. Created automatically upon app registration or when the app receives consent to access tenant resources. References the globally unique app object. |
| Managed Identity | Represents a managed identity that eliminates the need for developers to manage secrets/certificates. | Provides an identity for applications connecting to resources that support Entra authentication (primarily designed for Azure-to-Azure authentication). The service principal is created automatically when a managed identity is enabled. Can be granted access and permissions, but cannot be updated or modified directly and has no associated app object, unlike `Application` type. |
| Legacy | Represents apps created before app registrations were introduced or through legacy experiences. | Can have credentials, service principal names, reply URLs, and other editable properties, but has no associated app registration. Can only be used in the tenant where it was created. |

## Use case: App Service web app accessing Microsoft Graph API (or other service)

- For multi-tenant applications, a service principal (type: Application) is created in each tenant to grant the app access to that tenant's Microsoft Graph API resources without requiring a user to sign in.
- For single-tenant scenarios where the app only needs to access resources within a specific tenant, a _Managed Identity_ could be more appropriate.
- When the app needs to perform actions specific to an individual user (_on behalf of the user_), delegated permissions are used, requiring user authentication and consent.
- When different users need different levels of access, use delegated permissions with the [dynamic consent](https://learn.microsoft.com/en-us/entra/identity-platform/howto-update-permissions?pivots=ms-graph#add-permissions-into-dynamic-consent) to request permissions on-demand.
- For short-lived operations that don't require persistent permissions, _token-based or key-based temporary access_ could be more fitting.

**📝 NOTE:** Microsoft Graph API is the gateway to all the data in a customer's Microsoft 365 and Entra ID tenant.
## Application Registration

All applications _must [register with Entra ID](https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-provider-aad?tabs=workforce-tenant)_ to delegate identity and access management: `Portal > app > 'Authentication' > 'Add identity provider' > set provider to Microsoft > 'Add'`. This creates an application object and a globally unique ID (app/client ID).

Changes to your application object also affect its service principals in the home tenant only. Deleting the application also deletes its home tenant service principal, but restoring that application object won't recover its service principals.

List service principals associated with an app: `az ad sp list --filter "appId eq '<app-id>'"`

| [Integrate authentication and authorization](https://learn.microsoft.com/en-us/entra/identity-platform/media/v2-overview/application-scenarios-identity-platform.png) | Web App | Backend API                                                                            | Daemon |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- | -------------------------------------------------------------------------------------- | ------ |
| 1. Register in Entra ID                                                                                                                                               | ✓       | ✓                                                                                      | ✓      |
| 2. Configure app with code sample                                                                                                                                     | ✕       | ✓                                                                                      | ✕      |
| 3. Validate token                                                                                                                                                     | ID      | Access                                                                                 | ✕      |
| 4. Configure secrets & certificates                                                                                                                                   | ✓       | ✓                                                                                      | ✓      |
| 5. Configure permission & call API of choice                                                                                                                          | ✓       | ✓                                                                                      | ✓      |
| 6. Control access (authorization)                                                                                                                                     | ✓       | ✓ (add `validate-jwt` OR `validate-azure-ad-token` policy to validate the OAuth token) | ✕      |
| 7. Store token cache                                                                                                                                                  | ✓       | ✓                                                                                      | ✓      |

**App registration and configuration (secrets/certificates, permissions, SDK/middleware) are done ahead of time at design/deploy time. Above diagram shows the runtime ordering.** At runtime:

- The client requests a token (user sign-in and consent / authorization)
- The identity platform returns a token
- The client then stores the token in its token cache and uses the token to call the Web API
- The Web API validates the access token and then enforces authorization (scopes, roles, policies)

---

To [protect an API in Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-protect-backend-with-aad), register both the backend API and web app, configure permissions to allow the web app to call the backend API (`az ad app permission add --id <WebApp-Application-Id> --api <Backend-API-Application-Id> --api-permissions <Permission-Id>=Scope`), and enable OAuth 2.0 user authorization along with adding the `validate-jwt` policy for token validation.

- `<Permission-Id>=Scope`: Delegated permissions
- `<Permission-Id>=Role`: Application permissions

## [Permissions (Scopes)](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview#types-of-permissions)

The app specifies required permissions using the `scope` query parameter, comprises of resource and permissions. If resource is unspecified, the default resource is Microsoft Graph. For instance, `scope=User.Read` is the same as `https://graph.microsoft.com/User.Read`.

| Permission types | Delegated permissions                                                    | Application permissions                    |
| ---------------- | ------------------------------------------------------------------------ | ------------------------------------------ |
| Access context   | Get access on behalf of a user (a signed-in user is present)             | Get access without a user |
| Types of apps    | Web / Mobile / single-page app (SPA)                                     | Web / Daemon / Background services / Automation         |
| Other names      | Scopes<br>OAuth2 permission scopes                                        | App roles<br>App-only permissions           |
| Who can consent  | - Users can consent for their data<br>- Admins can consent for all users | Only admin can consent                     |
| Consent methods  | - Static: configured list on app registration<br>- Dynamic: request individual permissions at sign-in                                                        | Static ONLY                                |

```bash
# API permission to an existing Entra ID app registration.
`az ad app permission add --id {appId} --api {apiID} --api-permissions {permissionId}={Scope,Role}`
```
Breakdown of above command:
- {apiID}: This specifies the resource API your app wants to access. This could be Microsoft Graph, [Azure Key Vault or another API](https://gemini.google.com/share/d66e663a704f).
- {permissionId}: The unique ID for the permission you want (e.g., the ID for User.Read or Mail.Send).
- Scope: Defines **what the application is allowed to do** on behalf of a user.
- Role: Defines more privileged **operations that the application can perform**, without a user.

### [Consent](https://learn.microsoft.com/en-us/azure/active-directory/manage-apps/user-admin-consent-overview)

- **Static user consent**: Requires all permissions to be specified in the Azure portal during app registration. Users or admins are prompted for consent if not previously granted. Issues: requires long lists of permissions and knowing all resources in advance.
- **Incremental (Dynamic) user consent**: Allows permissions to be requested gradually. Scopes can be specified during runtime without predefinition in Azure portal.
- **Admin consent**: needed for [high-privilege permissions](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/user-admin-consent-overview#:~:text=highly%20privileged%20operations.%20Examples%20of%20such%20operations%20might%20be%20role%20management%2C%20full%20access%20to%20all%20mailboxes%20or%20all%20sites%2C%20and%20full%20user%20impersonation.). Admins authorize apps to access privileged data. `az ad app permission admin-consent`

[Admin consent workflow](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-admin-consent-workflow) gives admins a secure way to grant access to applications that require admin approval. When a user tries to access an application but is unable to provide consent, they can send a request for admin approval. The request is sent via email to admins who are designated as reviewers. 

[Requesting individual user consent](https://learn.microsoft.com/en-us/entra/identity-platform/consent-types-developer#requesting-individual-user-consent):

```http
GET https://login.microsoftonline.com/common/oauth2/v2.0/authorize?
client_id=00001111-aaaa-2222-bbbb-3333cccc4444
&response_type=code
&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F
&response_mode=query
&scope=https%3A%2F%2Fgraph.microsoft.com%2Fcalendars.read%20https%3A%2F%2Fgraph.microsoft.com%2Fmail.send
&state=12345
```

## [Conditional Access](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/overview) (Premium P1+ tier)
Combines signals including user or group membership, IP location information, device compliance status, application details, and real-time risk detection to make access decisions. Some examples of conditional access policies:
- Prompt additional verification (e.g., second password or fingerprint) when users sign in
- [Using a middle tier to solve a "challenge"](https://learn.microsoft.com/en-us/entra/identity-platform/v2-conditional-access-dev-guide#:~:text=The%20middle%20tier%20performs%20on%2Dbehalf%2Dof%20flow%20to%20request%20access%20to%20the%20downstream%20API.%20At%20this%20point%2C%20a%20claims%20%22challenge%22%20is%20presented%20to%20the%20middle%20tier.%20The%20middle%20tier%20sends%20the%20challenge%20back%20to%20the%20native%20app%2C%20which%20needs%20to%20comply%20with%20the%20Conditional%20Access%20policy.) presented by API
- [Multi-Factor Authentication](https://learn.microsoft.com/en-us/azure/active-directory/authentication/concept-mfa-licensing) (**all Microsoft 365 plans**). When [Security Defaults](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/security-defaults) is enabled, MFA is activated for **all users**. To apply MFA to specific users only, _disable Security Defaults_.
- [Risk-based policies](https://learn.microsoft.com/en-us/entra/fundamentals/security-defaults#:~:text=These%20policies%20need,strengthen%20their%20posture.) (require Entra ID Identity Protection - **Premium P2** tier)
- Device restrictions (enrolled in Microsoft's Intune service)
- Certain physical locations or IP ranges

When Conditional Access licenses expire, policies stay active but can't be updated.

Apps don't need to be changed, unless they need silent or indirect services access, or on-behalf-of flow.

## Other Entra ID features

- [Entra ID B2C](https://learn.microsoft.com/en-us/azure/active-directory-b2c/overview) supports multiple login methods, including social media, email/password.
- [Entra ID B2B](https://learn.microsoft.com/en-us/azure/active-directory/external-identities/what-is-b2b) allows you to share your company's applications with external users in a secure manner.
- [Entra ID Application Proxy](https://learn.microsoft.com/en-us/entra/identity/app-proxy/overview-what-is-app-proxy) provides secure remote access to on-premises applications.
- [Entra ID Connect](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-azure-ad-connect) allows you to synchronize hybrid identities.
- [Entra ID Enterprise Application](https://learn.microsoft.com/en-us/azure/active-directory/manage-apps/add-application-portal) allow you to integrate other applications with Entra ID, including your own/enterprise apps.

## MSAL (Microsoft Authentication Library)

Enables secure access to various APIs, with a unified API across platforms.

- Obtains tokens for users or applications (when applicable).
- Manages token cache and refreshes tokens automatically.
- Helps specify the desired audience for application sign-in.
- Assists with application setup from configuration files.
- Provides actionable exceptions, logging, and telemetry for troubleshooting.

### [Authentication flows](https://learn.microsoft.com/en-us/azure/active-directory/develop/msal-authentication-flows)

| Flow               | Application Type | Use Case                                          | Token Acquisition Method                    |
| ------------------ | ---------------- | ------------------------------------------------- | ------------------------------------------- |
| Authorization code | Both             | SPA, Native, Web Apps                             | Exchanges an authorization code for a token |
| Client credentials | Confidential     | Daemon, Backend                                   | App Secret/Certificate                      |
| On-behalf-of       | Both             | Service-to-Service, Service-to-API, Microservices | Uses an existing token to get another       |
| Device code        | Public           | IoT, CLI                                          | Polls the endpoint until user authenticates |
| Implicit Grant           | Public           | Legacy SPAs                                       | Token in URI fragment                       |
| Integrated Windows | Both             | Intranet Apps                                     | Auto auth on domain / Entra ID-joined PCs   |
| Interactive        | Public           | User Interactive Apps                             | Requires user action                        |
| Username/password  | Both             | Legacy, Testing                                   | Direct with Credentials                     |

- **Public client applications**: User-facing apps without the ability to securely store secrets. They interact with web APIs on the user's behalf.
- **Confidential client applications**: Server-based apps and daemons that can securely handle secrets. Each instance maintains a unique configuration, including identifiers and secrets.

**📝 NOTE:** It's no longer recommended to use the implicit grant flow. If you're building a SPA, use the `authorization code flow with PKCE` instead.

### [Working with MSAL](https://learn.microsoft.com/en-us/entra/msal/dotnet/)

When building web apps or public client apps that [require a broker](https://learn.microsoft.com/en-us/entra/msal/dotnet/getting-started/instantiate-public-client-config-options#:~:text=For%20web%20apps%2C%20and%20sometimes%20for%20public%20client%20apps%20(in%20particular%20when%20your%20app%20needs%20to%20use%20a%20broker)%2C%20you%27ll%20have%20also%20set%20the%20redirectUri%20where%20the%20identity%20provider%20will%20contact%20back%20your%20application%20with%20the%20security%20tokens.), ensure to set the `redirectUri`. This URL is used by the identity provider to return security tokens to your application.

[Integrating Entra ID authentication into an ASP.NET Core application](https://learn.microsoft.com/en-us/entra/identity-platform/tutorial-web-app-dotnet-sign-in-users?tabs=workforce-tenant):

```cs
builder.Services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
  .AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("AzureAd"));

builder.Services.AddRazorPages()
  .AddMicrosoftIdentityUI();

// OpenIdConnectDefaults.AuthenticationScheme: Enables OpenID Connect authentication, best for OAuth 2.0 and Single Sign-On (SSO).
// JwtBearerDefaults.AuthenticationScheme: Used for authenticating APIs via JSON Web Tokens (JWT), suitable for stateless and scalable APIs.
// CookieAuthenticationDefaults.AuthenticationScheme: Employs cookies for session-based authentication, optimal for traditional web apps that manage user sessions server-side.
// Custom Authentication Scheme: Allows for custom string identifiers for authentication middleware, ideal for specialized or unique authentication scenarios.
```
[Initializing a public client application](https://learn.microsoft.com/en-us/entra/msal/dotnet/getting-started/initializing-client-applications#initializing-a-public-client-application-from-code):
```cs
// Sign in users in the Microsoft Azure public cloud using their work and school accounts or personal Microsoft accounts.
IPublicClientApplication app = PublicClientApplicationBuilder.Create(clientId).Build();
```
[Initializing a confidential client application](https://learn.microsoft.com/en-us/entra/msal/dotnet/getting-started/initializing-client-applications#initializing-a-confidential-client-application-from-code):
```cs
// Confidential application that handles tokens from Microsoft Azure users using a shared client secret for identification.
// Example: A daemon application that does not interact with the user and acts on its own behalf, like a service accessing Graph API
IConfidentialClientApplication app = ConfidentialClientApplicationBuilder.Create(clientId)
    .WithClientSecret(clientSecret)
    .WithRedirectUri(redirectUri )
    .Build();
```

[Common modifiers](https://learn.microsoft.com/en-us/entra/msal/dotnet/getting-started/initializing-client-applications#modifiers-common-to-public-and-confidential-client-applications):

| Modifier                                              | Description                                                                                                                                                                                                                                                                                     |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.WithAuthority(AadAuthorityAudience authorityAudience, bool validateAuthority)`                                    | Sets the application default authority to an Microsoft Entra ID authority, with the possibility of choosing the Azure Cloud, the audience, the tenant (tenant ID or domain name), or providing directly the authority URI. Example: `.WithAuthority(AzureCloudInstance.AzurePublic, _tenantId)` |
| `.WithTenantId(string tenantId)`                      | Overrides the tenant ID, or the tenant description.                                                                                                                                                                                                                                             |
| `.WithClientId(string)`                               | Overrides the client ID.                                                                                                                                                                                                                                                                        |
| `.WithRedirectUri(string redirectUri)`                | Overrides the default redirect URI (ex: for scenarios requiring a broker)                                                                                                                                                                                                                       |
| `.WithComponent(string)`                              | Sets the name of the library using MSAL.NET (for telemetry reasons).                                                                                                                                                                                                                            |
| `.WithDebugLoggingCallback()`                         | If called, the application calls Debug.Write simply enabling debugging traces.                                                                                                                                                                                                                  |
| `.WithLogging()`                                      | If called, the application calls a callback with debugging traces. Allows you to route MSAL's logs into your application's own logging framework (like Serilog, NLog, or Application Insights).                                                                                                                                                                                                                             |
| `.WithTelemetry(TelemetryCallback telemetryCallback)` | Sets the delegate used to send telemetry.                                                                                                                                                                                                                                                       |

[Confidential client application only](https://learn.microsoft.com/en-us/entra/msal/dotnet/getting-started/initializing-client-applications#modifiers-specific-to-confidential-client-applications):

| Modifier                                         | Description                                                                                |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| `.WithCertificate(X509Certificate2 certificate)` | Sets the certificate identifying the application with Microsoft Entra ID.                  |
| `.WithClientSecret(string clientSecret)`         | Sets the client secret (app password) identifying the application with Microsoft Entra ID. |

[Acquiring Token](https://learn.microsoft.com/en-us/entra/msal/dotnet/acquiring-tokens/desktop-mobile/acquiring-tokens-interactively):

```cs
string[] scopes = { "user.read" };
AuthenticationResult result = await app.AcquireTokenInteractive(scopes).ExecuteAsync();
Console.WriteLine($"Token: {result.AccessToken}");
```

## [Application manifest](https://learn.microsoft.com/en-us/azure/active-directory/develop/reference-app-manifest)

Configures an app's identity and attributes, facilitating OAuth authorization and user consent. Also serves as a mechanism for updating the application object.

- `signInAudience`:

  - `AzureADMyOrg` - Users with a Microsoft work or school account in my organization's Microsoft Entra tenant (for example, single tenant)
  - `AzureADMultipleOrgs` - Users with a Microsoft work or school account in any organization's Microsoft Entra tenant (for example, multi-tenant)
  - `AzureADandPersonalMicrosoftAccount` - Users with a personal Microsoft account, or a work or school account in any organization's Microsoft Entra tenant
  - `PersonalMicrosoftAccount` - Personal accounts that are used to sign in to services like Xbox and Skype.

- `groupMembershipClaims`: (_Tenant-specific_) Configures the groups claim. It tells Entra ID which of the user's groups to list in the groups claim every time it issues a token for this specific app. Entra ID group objects in the tenant are unaffected when the associated app is removed.

  - "None"
  - "SecurityGroup" (will include security groups and Entra ID roles)
  - "ApplicationGroup" (this option includes only groups that are assigned to the application)
  - "DirectoryRole" (gets the Entra ID directory roles the user is a member of)
  - "All" (gets all of the security groups, distribution groups, and Entra ID directory roles that the signed-in user is a member of).

**📝 NOTE:** In `Client Credentials Flow` (e.g. `for a Daemon service`): When a daemon app (a service principal) authenticates using its own credentials (a secret or certificate), it's an "app-only" context. The Entra ID token service only looks for app roles assigned directly to that service principal. It does not check the service principal's group memberships to find inherited roles.

- [`appRoles`](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-add-app-roles-in-apps) (_Application-specific_): Collection of roles that an app may declare. Defined in the app registration, and will get removed with it. Correspond to `Role` in `--api-permissions`.

  ```jsonc
  "appRoles": [{
      "allowedMemberTypes": [ "User" ],
      "value": "ReadOnly" // expected value of the roles claim in the token, which must match the string in the application's code without spaces.
  }]
  ```
**📝 NOTE:** Currently, if you add a service principal to a group, and then assign an app role to that group, Microsoft Entra ID doesn't add the roles claim to tokens it issues.

**📝 NOTE:** Group-based app role assignments are primarily for granting permissions to human users. When a user authenticates, their group memberships are evaluated to determine their roles. Service principals: When an application authenticates using the client credentials flow, it acts on its own identity (its service principal), not on behalf of a user. The system only includes claims based on roles directly assigned to that service principal.

- `oauth2Permissions`: Specifies the collection of OAuth 2.0 permission scopes that the web API (resource) app exposes to client apps. Correspond to `Scope` in `--api-permissions`.

- `oauth2AllowImplicitFlow` - If the web app can request implicit flow access tokens (`oauth2AllowIdTokenImplicitFlow` for ID tokens). ⭐: SPAs, when using Implicit Grant Flow.

### [Other Important Attributes](https://learn.microsoft.com/en-us/entra/identity-platform/reference-app-manifest#manifest-reference)
| Attribute Name               | Brief Explanation                                                                                                                     |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `requiredResourceAccess`     | Specifies the resources that the app requires access to.                                                                              |
| `keyCredentials`             | Holds references to app-assigned credentials, string-based shared secrets and X.509 certificates.                                     |
| `acceptMappedClaims`         | Allows an application to use claims mapping without specifying a custom signing key.                                                  |
| `optionalClaims`             | The optional claims returned in the token by the security token service for this specific app.                                        |
| `addIns`                     | Defines custom behavior that a consuming service can use to call an app in specific contexts.                                         |
| `allowPublicClient`          | Specifies the fallback application type.                                                                                              |
| `knownClientApplications`    | Used for bundling consent if you have a solution that contains two parts: a client app and a custom web API app.                      |
| `oauth2RequirePostResponse`  | Specifies whether, as part of OAuth 2.0 token requests, Entra ID will allow POST requests, as opposed to GET requests.                |
| `passwordCredentials`        | Similar to `keyCredentials`, holds references to app-assigned credentials, string-based shared secrets.                               |
| `preAuthorizedApplications`  | Lists applications and requested permissions for implicit consent.                                                                    |
| `replyUrlsWithType`          | Holds the list of registered redirect_uri values that Entra ID will accept as destinations when returning tokens.                     |
| `signInAudience`             | Specifies what Microsoft accounts are supported for the current application.                                                          |
| `identifierUris`             | User-defined URI(s) that uniquely identify a web app within its Entra ID tenant or verified customer owned domain.                    |
| `tags`                       | Custom strings that can be used to categorize and identify the application.                                                           |
| `parentalControlSettings`    | Specifies the countries/regions in which the app is blocked for minors and the legal age group rule that applies to users of the app. |
| `accessTokenAcceptedVersion` | Specifies the access token version expected by the resource.                                                                          |
| `logoutUrl`                  | The URL to log out of the app.                                                                                                        |
| `signInUrl`                  | Specifies the URL to the app's home page.                                                                                             |
| `logoUrl`                    | Read only value that points to the CDN URL to logo that was uploaded in the portal.                                                   |
| `samlMetadataUrl`            | The URL to the SAML metadata for the app.                                                                                             |
| `publisherDomain`            | The verified publisher domain for the application.                                                                                    |
| `informationalUrls`          | Specifies the links to the app's terms of service and privacy statement.                                                              |
| `appId`                      | Specifies the unique identifier for the app that is assigned to an app by Entra ID.                                                   |
| `name`                       | The display name for the app.                                                                                                         |
| `id`                         | The unique identifier for the app in the directory.                                                                                   |

### Mapping Attributes to CLI
| Application Manifest Attribute | Azure CLI `--api-permissions` Value | Type |
|-------------------------------|-------------------------------------|------|
| `appRoles` | `Role` | Application permissions (app-only) |
| `oauth2Permissions` | `Scope` | Delegated permissions (on behalf of user) |

## ASP.NET Core Authorization: Working with Roles, Claims, and Policies

### Hierarchy Overview

```
┌─────────────────────────────────────┐
│          POLICIES                    │  ← Most Powerful
│  (Can combine roles + claims +       │
│   custom logic)                      │
└────────────┬────────────────────────┘
             │ uses
             ↓
┌─────────────────────────────────────┐
│          ROLES                       │  ← Medium Flexibility
│  (Implemented as special claims)     │
└────────────┬────────────────────────┘
             │ built on
             ↓
┌─────────────────────────────────────┐
│          CLAIMS                      │  ← Foundation
│  (Raw identity data)                 │
└─────────────────────────────────────┘
```

- [**Policies**](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies?view=aspnetcore-3.1): A policy is a function that can look at a user's identity and decide whether they are authorized to perform a given action.
- [**Roles**](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/roles?view=aspnetcore-3.1): Specifies whether the current user is a member of the roles necessary to access the requested resource.
- [**Claims**](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/claims?view=aspnetcore-3.1): A claim is a name-value pair that represents what the subject is, not what the subject can do.

```cs
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    // Define policies
    services.AddAuthorization(options =>
    {
        options.AddPolicy("ClientsOnly", policy =>
        {
            policy.RequireAuthenticatedUser(); // Requires the user to be authenticated
            policy.RequireRoles("PrivateClient", "CorporateClient"); // Requires ANY of these listed roles
            policy.RequireClaim("SubscriptionTier", "free", "basic", "premium"); // Requires ANY of the values for SubscriptionTier
        });
        options.AddPolicy("FreeloadersOnly", policy =>
        {
            policy.RequireAuthenticatedUser();
            policy.RequireRole("PrivateClient");
            policy.RequireClaim("SubscriptionTier", "free");
        });
        options.AddPolicy("EmployeeOnly", policy => policy.RequireClaim("EmployeeNumber")); // Requires EmployeeNumber claim
        options.AddPolicy("AdminOnly", policy => policy.RequireRole("Administrator")); // Requires Administrator role
    });
}

// LoginController.cs
[HttpPost]
public async Task<IActionResult> Login(LoginViewModel model)
{
  // Checks omitted for brevity
  var user = await _userManager.FindByNameAsync(model.Username);
  var claim = new Claim("EmployeeNumber", "123");
  await _userManager.AddClaimAsync(user, claim);
  await _signInManager.SignInAsync(user, isPersistent: false); // Sign in the user

  return RedirectToAction("Index", "Home");
}

[Authorize(Policy = "ClientsOnly")] // Allow clients with any subscription tier
public class AdminController : Controller { }

[Authorize(Roles = "CorporateClient")] // Allow corporate clients only
public class AdminController : Controller { }

[Authorize(Policy = "EmployeeOnly")] // Apply EmployeeOnly policy
public class WorkController : Controller { }

[Authorize(Policy = "AdminOnly")] // Apply AdminOnly policy
public class AdminController : Controller { }

[Authorize(Policy = "ClientsOnly")] // Clients only that also have "Administrator" role (AND)
[Authorize(Roles = "Administrator")]
public class ClientAdminController : Controller { }
```

# [Azure Event Grid](https://docs.microsoft.com/en-us/azure/event-grid/)

Azure Event Grid is a serverless broker that facilitates application integration through events. It simplifies event consumption and lowers costs by eliminating the need for constant polling. It routes events from sources such as applications and Azure services to subscribers, who select the events they want to handle.

**Publishers** send events without needing to know or expect specific subscribers. Event Grid supports both Azure and custom topics, simplifying the development of event-driven applications. It ensures reliable delivery by offering the ability to filter and route events to multiple endpoints. Subscriptions to events can include those from Azure services and third-party resources.

A _partner_ is a kind of publisher that sends events to Azure Event Grid for customers and can also receive events from it.

## Concepts

- **Events** - What happened. An event describes something that occurred, with common details like source, time, and unique identifier, and specific details relevant to the event type (example: in Azure Storage, a new file event includes details like `lastTimeModified` value, and in Event Hubs, it contains the Capture file URL). Events up to 64 KB are covered by the Service Level Agreement (SLA), and larger events are charged in increments (max size 1MB) - a 130 KB event is charged as three separate events.
- **Event sources** - Where the event took place. Each source is related to one or more event types, such as Azure Storage for blob creation, IoT Hub for device created events. Event sources send events to Event Grid.
- **Topics** - The endpoint where publishers send events. Topics are used for related events, and subscribers choose which to subscribe to. **System topics** are built-in and, while **custom topics** are application and third-party specific. You don't see system topics in your Azure subscription, but you can subscribe to them. [**Partner topics**](https://learn.microsoft.com/en-us/azure/event-grid/partner-events-overview) let you subscribe to events from external systems, like SaaS apps, and use Azure services like Functions, Logic Apps, or custom code to handle them. You can create subscriptions like any other topic.
- **Event subscriptions** - The endpoint or mechanism to route events, sometimes to multiple handlers. Subscriptions filter incoming events by type or subject pattern and can be set with an expiration for temporary needs (_no need of cleanup_).
- **Event handlers** - Receives and processes events. Handlers can be Azure services or custom webhooks. Event Grid ensures event delivery based on handler type.

## Schemas

The header values for CloudEvents and Event Grid schemas are identical except for the `content-type`. In CloudEvents, it's `"content-type":"application/cloudevents+json; charset=utf-8"`, while in Event Grid, it's `"content-type":"application/json; charset=utf-8"`.

### Event Schemas

- Every event that is generated has the same top-level schema within Azure. This makes it universally easier to process events across your subscription, and also allows for filtering of events.
- Azure Event Grid receives events in an array from event sources. The array's total size can be up to 1 MB, with each event limited to 1 MB. If the event or array exceeds the limit, you'll get a `413 Payload Too Large` response.

```ts
type EventSchema = {
  // Optional full resource path to the event source.
  // If not included, Event Grid stamps onto the event.
  // If included, it must match the Event Grid topic Azure Resource Manager ID exactly.
  topic?: string;
  // Publisher-defined path to the event subject.
  subject: string;
  // One of the registered event types for this event source.
  eventType: string;
  // The time the event is generated based on the provider's UTC time.
  eventTime: string;
  // Unique identifier for the event. Not really matters in general.
  id: string;
  // Event data specific to the resource provider.
  data?: {
    // Optional object unique to each publisher. Also defined as "payload of a notification".
    // For a BlobCreated event, the data field contains e.g. the blob's url, contentType, and contentLength
    // Place your properties specific to the resource provider here.
  };
  // The schema version of the data object. The publisher defines the schema version.
  // If not included, it is stamped with an empty value.
  dataVersion?: string;
  // The schema version of the event metadata. Event Grid defines the schema of the top-level properties.
  // If not included, Event Grid will stamp onto the event.
  // If included, must match the metadataVersion exactly (currently, only 1)
  metadataVersion?: string;
};
```

When creating subjects for publishing events to custom topics, it helps subscribers filter and route events based on where they happened. Include the path in the subject for better filtering.

- Example 1: With a path like `/A/B/C`, subscribers can filter by `/A` for a broad set, getting events like `/A/B/C` or `/A/D/E`. For a narrower set, they can filter by `/A/B`.
- Example 2: The **Storage Accounts** publisher provides `/blobServices/default/containers/<container-name>/blobs/<file>` when adding a file. Subscribers can filter by `/blobServices/default/containers/testcontainer` for events from that container or use `.txt` to only work with text files.

### Cloud Event Schema

CloudEvents offers a unified event schema for publishing and consuming cloud-based events (input and output in Event Grid). You can use CloudEvents for system events, like Blob Storage events and IoT Hub events, and custom events. It also supports bidirectional event transformation during transmission.

```ts
interface CloudEvent {
  // Identifies the event. Producers must ensure it's unique. Consumers can assume same source+id means duplicates.
  id: string;

  // Identifies the context in which an event happened.
  // Syntax defined by the producer, preferably an absolute URI
  source: string;

  // The version of the CloudEvents specification used. Compliant producers MUST use value "1.0".
  specversion: string;

  // Describes the type of event related to the originating occurrence.
  // Should be prefixed with a reverse-DNS name.
  type: string;

  subject?: string; // Required in EventSchema, but optional here

  // eventType is now "type"
  // eventTime is now "time" and is optional

  // ...
}
```

Example of an Azure Blob Storage event in CloudEvents format:

```json
{
  "specversion": "1.0",
  "type": "Microsoft.Storage.BlobCreated",
  "source": "/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.Storage/storageAccounts/{storage-account}",
  "id": "9aeb0fdf-c01e-0131-0922-9eb54906e209",
  "time": "2019-11-18T15:13:39.4589254Z",
  "subject": "blobServices/default/containers/{storage-container}/blobs/{new-file}",
  "dataschema": "#",
  "data": {
    "api": "PutBlockList",
    "clientRequestId": "4c5dd7fb-2c48-4a27-bb30-5361b5de920a",
    "requestId": "9aeb0fdf-c01e-0131-0922-9eb549000000",
    "eTag": "0x8D76C39E4407333",
    "contentType": "image/png",
    "contentLength": 30699,
    "blobType": "BlockBlob",
    "url": "https://gridtesting.blob.core.windows.net/testcontainer/{new-file}",
    "sequencer": "000000000000000000000000000099240000000000c41c18",
    "storageDiagnostics": {
      "batchId": "681fe319-3006-00a8-0022-9e7cde000000"
    }
  }
}
```

## Event Delivery Durability

- **Delivery Attempts**: Event Grid attempts to deliver each event at least once for each matching subscription immediately.
- **Single Event Payload**: By default, one event is delivered at a time, and the payload is an array with a single event.
- **Event Order**: Event Grid does not guarantee the order of event delivery.

Event subscriptions support custom headers for delivered events. _Up to 10 headers_ can be set, with each value _limited to 4,096 bytes_. Can be applied to events sent to:

- Webhooks
- Azure Service Bus topics and queues
- Azure Event Hubs
- Relay Hybrid Connections.

### [Delivery and Retry](https://learn.microsoft.com/en-us/azure/event-grid/delivery-and-retry)

#### Retry schedule

Event Grid handles errors during event delivery by deciding based on the error type whether to retry, dead-letter (only if enabled), or drop the event. Timeout is 30 sec, then event is rescheduled for retry (exponentially). Retries may be skipped or delayed (up to several hours) for consistently unhealthy endpoints (**delayed delivery**). If the endpoint responds within 3 minutes, Event Grid tries to remove the event from the retry queue. Because of this, duplicates may occur.

| Endpoint Type   | Success               | Error Codes with no retries (immediate dead-lettering)                                        |
| --------------- | --------------------- | --------------------------------------------------------------------------------------------- |
| Azure Resources | 200 OK                | 400 Bad Request, 413 Request Entity Too Large, 403 Forbidden                                  |
| Webhook         | Successful processing | 400 Bad Request, 413 Request Entity Too Large, 403 Forbidden, 404 Not Found, 401 Unauthorized |

#### Retry policy

An event is dropped if either of the limits of the retry policy is reached.

- **Maximum number of attempts** - 1 - 30 (default: 30)
- **Event time-to-live (TTL)** - 1 - 1440 minutes. (default 1440)

```sh
az eventgrid event-subscription create \
  -g $resourceGroup \
  --topic-name <topic_name> \
  --name <event_subscription_name> \
  --endpoint <endpoint_URL> \
  --max-delivery-attempts 18
```

#### [Dead-Letter Events](https://learn.microsoft.com/en-us/azure/event-grid/manage-event-delivery)

Occurs if the published event isn't delivered within the retry policy limits. Targets to ensure no event is ever lost. To enable, specify a storage account for undelivered events, and provide the endpoint for this container during event subscription creation. A five-minute delay exists between the last attempt and delivery to the dead-letter location, to reduce Blob storage operations.

The time-to-live expiration is checked **only** at the next scheduled delivery attempt.

```sh
az eventgrid event-subscription create \
  --source-resource-id $topicid \
  --name <event_subscription_name> \
  --endpoint <endpoint_URL> \
  # To turn off dead-lettering, rerun this command, but don't provide a value for deadletter-endpoint
  --deadletter-endpoint $storageid/blobServices/default/containers/$containername
```

#### Output Batching

For better HTTP performance in handling a large number of events. _Off by default_. It doesn't support partial success of a batch delivery (_All or None_). The settings for batching are not strict and are respected on a best-effort basis, possibly resulting in smaller batches at low event rates (_Optimistic Batching_).

Settings:

- **Max events per batch**: 1 - 5,000 (default: 1?). If no other events are available at the time of publishing, fewer events may be delivered.
- **Preferred batch size in kilobytes**: 1 - 1024 (default: 64KB?). Smaller if not enough events are available. If a single event is larger than the preferred size, it will be delivered as its own batch.

```sh
az eventgrid event-subscription create \
  --resource-id $storageid \
  --name <event_subscription_name> \
  --endpoint $endpoint \
  --max-events-per-batch 1000 \
  --preferred-batch-size-in-kilobytes 512
```

## Control access to events

### Built-in roles

| Role                                | Description                                               |
| ----------------------------------- | --------------------------------------------------------- |
| Event Grid Subscription Reader      | Lets you read Event Grid event subscriptions.             |
| Event Grid Subscription Contributor | Lets you manage Event Grid event subscription operations. |
| Event Grid Contributor              | Lets you create and manage Event Grid resources.          |
| Event Grid Data Sender              | Lets you send events to Event Grid topics.                |

Subscriptions roles don't grant access for actions such as creating topics.

To subscribe to event handlers (except WebHooks), you need **Microsoft.EventGrid/EventSubscriptions/Write** permission to the corresponding resource (for system topics and custom topics).

| Topic Type | Permission to write a new event subscription at scope | Description                                                                                                                           |
| ---------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| System     | Resource publishing the event                         | `/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/{resource-provider}/{resource-type}/{resource-name}` |
| Custom     | Event grid topic                                      | `/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/Microsoft.EventGrid/topics/{topic-name}`             |

## [Webhooks](https://learn.microsoft.com/en-us/azure/event-grid/webhook-event-delivery)

When a new event is ready, Event Grid service POSTs an HTTP request to the configured endpoint with the event in the request body.

Validation is automatically handled for:

- Azure Logic Apps with Event Grid Connector
- Azure Automation via webhook
- Azure Functions with Event Grid Trigger

### Endpoint validation with Event Grid events

When using an endpoint like an HTTP-triggered Azure function with Event Grid, you'll need to validate the subscription through handshake:

1. Webhook endpoint must return an HTTP 200 status code.
1. One of the following properties must be available in the response
   - `validationCode` to complete **Synchronous handshake**
   - `validationUrl` to transition to manual validation mode (**Asynchronous handshake**). Event subscription status changes to `AwaitingManualAction`. You must perform a GET request to this URL within 5 minutes, otherwise, status will change to `Failed`, and you must restart the process.

**📝 NOTE:** Self-signed certificates are not supported for validation; a signed certificate from a commercial CA is required.

## [Filtering](https://learn.microsoft.com/en-us/azure/event-grid/event-filtering)

### Using ARM

```json
{
  "filter": {
    "subjectBeginsWith": "/blobServices/default/containers/mycontainer/log",
    "subjectEndsWith": ".jpg"
  }
}
```

```json
{
  "filter": {
    "includedEventTypes": [
      "Microsoft.Resources.ResourceWriteFailure",
      "Microsoft.Resources.ResourceWriteSuccess"
    ]
  }
}
```

#### Advanced (Conditional)

```jsonc
{
  "filter": {
    // enableAdvancedFilteringOnArrays: true // Allow array keys
    "advancedFilters": [
      // AND operation
      {
        "operatorType": "NumberGreaterThanOrEquals",
        "key": "Data.Key1", // The field in the event data that you're using for filtering (number, boolean, string)
        "value": 5
      },
      {
        "operatorType": "StringContains",
        "key": "Subject",
        "values": ["container1", "container2"] // OR operation
      }
    ]
  }
}
```

#### Limitations:

- 25 advanced filters and 25 filter values across all the filters per Event Grid subscription
- 512 characters per string value
- No support for escape characters in keys

### Using CLI

```sh
az eventgrid event-subscription create
  --name "<Subscription_Name>"
  --source-resource-id "<Event_Grid_Topic_Resource_Id>"
  --endpoint "<Azure_Function_URL>"
  --endpoint-type azurefunction
  --subject-begins-with "/A/B"
  --subject-ends-with ".jpg"
```

## [Route custom events to web endpoint](https://learn.microsoft.com/en-us/azure/event-grid/custom-event-quickstart-portal)

```sh
mySiteURL="https://${mySiteName}.azurewebsites.net"

# Create a resource group
az group create --name $resourceGroup --location $myLocation

# Create a custom topic
az eventgrid topic create --name $myTopicName \
    --location $myLocation \
    --resource-group $resourceGroup

# Create a message endpoint
az deployment group create \
    --resource-group $resourceGroup \
    --template-uri "https://raw.githubusercontent.com/Azure-Samples/azure-event-grid-viewer/main/azuredeploy.json" \
    --parameters siteName=$mySiteName hostingPlanName=viewerhost

# Subscribe to a custom topic
endpoint="${mySiteURL}/api/updates"
subId=$(az account show --subscription "" | jq -r '.id')
az eventgrid event-subscription create \
    --source-resource-id "/subscriptions/$subId/resourceGroups/$resourceGroup/providers/Microsoft.EventGrid/topics/$myTopicName" \
    --name az204ViewerSub \
    --endpoint $endpoint

# Send an event to the custom topic
topicEndpoint=$(az eventgrid topic show --name $myTopicName -g $resourceGroup --query "endpoint" --output tsv)
key=$(az eventgrid topic key list --name $myTopicName -g $resourceGroup --query "key1" --output tsv)
event='[ {"id": "'"$RANDOM"'", "eventType": "recordInserted", "subject": "myapp/vehicles/motorcycles", "eventTime": "'`date +%Y-%m-%dT%H:%M:%S%z`'", "data":{ "make": "Contoso", "model": "Monster"},"dataVersion": "1.0"} ]'
curl -X POST -H "aeg-sas-key: $key" -d "$event" $topicEndpoint
```

## [Configure Azure Event Grid service to send events to an Azure Event Hub instance](https://learn.microsoft.com/en-us/azure/event-grid/custom-event-to-eventhub)

```sh
az eventgrid topic create --name $topicName --location $location --resource-group $resourceGroup

# az eventgrid topic show --name $topicName --resource-group $resourceGroup --query "{endpoint:endpoint, primaryKey:primaryKey}" --output json

# Create a namespace
az eventhubs namespace create --name $namespaceName --location $location --resource-group $resourceGroup

# Create an event hub
az eventhubs eventhub create --name $eventHubName --namespace-name $namespaceName --resource-group $resourceGroup

topicId=$(az eventgrid topic show --name $topicName --resource-group $resourceGroup --query "id" --output tsv)
hubId=$(az eventhubs eventhub show --name $eventHubName --namespace-name $namespaceName --resource-group $resourceGroup --query "id" --output tsv)

# Link the Event Grid Topic to the Event Hub
az eventgrid event-subscription create \
  --name "<Event_Subscription_Name>" \
  --source-resource-id $topicId \
  --endpoint-type eventhub \
  --endpoint $hubId
```

## Working with EventGrid

[Register `Microsoft.EventGrid` provider](./AZ%20CLI.md)

### Publish new events

Create an Event Grid subscription: `Azure portal > Resource groups > PubSubEvents > eventviewer[yourname] web app > + Event Subscription`

```cs
Uri endpoint = new Uri(topicEndpoint);

// topicKey is the key for your Event Grid topic, which you can find in the Azure Portal
var credential = new AzureKeyCredential(topicKey);

var client = new EventGridPublisherClient(endpoint, credential);

var newEmployeeEvent = new EventGridEvent(
  subject: $"New Employee: Alba Sutton",
  eventType: "Employees.Registration.New",
  dataVersion: "1.0",
  data: new
  {
    FullName = "Alba Sutton",
    Address = "4567 Pine Avenue, Edison, WA 97202"
  }
);

// Alt: CloudEvent is a standardized specification designed to provide interoperability across services, platforms, and systems.
// It can be used across different cloud providers and platforms, unlike EventGridEvent, which is specific to Azure.
// var newEmployeeEvent = new CloudEvent("Employees.Registration.New", new Uri("/mycontext"))
// {
//     Subject = $"New Employee: Alba Sutton",
//     DataContentType = new ContentType("application/json"),
//     Data = JsonConvert.SerializeObject(new
//     {
//         FullName = "Alba Sutton",
//         Address = "4567 Pine Avenue, Edison, WA 97202"
//     })
// };


await client.SendEventAsync(newEmployeeEvent);
```

# [Azure Event Hub](https://docs.microsoft.com/en-us/azure/event-hubs/)

- **Fully Managed PaaS**, integrating with Apache Kafka.
- **Real-Time and Batch Processing**: Uses partitioned consumer model to process streams concurrently and control processing speed.
- **Capture Event Data**: Stores data in near-real-time in Azure Blob storage or Azure Data Lake Storage.

## Key Concepts

- **Producer**: Source of various types of data such as telemetry, diagnostics, logs, etc. Send data to an event hub via SDKs or Kafka producer clients.
- **Namespace**: A container managing event hubs or Kafka topics, handling tasks like capacity, security, and disaster recovery.
- **Event Hubs/Kafka topic**: An append-only log for organizing events, consisting of one or more partitions.
- **Partitions**: Ordered sequence of events in an Event Hub - grouped by partition key-, used for parallelism in data processing (increasing throughput).
- **Receiver/Consumer**: Any service/app that reads information from Event Hubs for processing, distribution, or storage. Uses SDKs or Kafka.
- **Consumer group**: A logical group allowing multiple consumers to read the same data independently. Up to 20 consumer groups with 5 concurrent consumers per each are allowed.

![Image showing the event processing flow.](https://learn.microsoft.com/en-us/training/wwl-azure/azure-event-hubs/media/event-hubs-stream-processing.png)

## [Tiers](https://learn.microsoft.com/en-us/azure/event-hubs/compare-tiers)

| Limit                                                         | Basic                                                                         | Standard                                                                      | Premium                                                                    | Dedicated                          |
| ------------------------------------------------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------- |
| **Maximum size of Event Hubs publication**                    | 256 KB                                                                        | 1 MB                                                                          | 1 MB                                                                       | 1 MB                               |
| **Number of consumer groups per event hub**                   | 1                                                                             | 20                                                                            | 100                                                                        | 1,000 (No limit per CU)            |
| **Number of Kafka consumer groups per namespace**             | N/A                                                                           | 1,000                                                                         | 1,000                                                                      | 1,000                              |
| **Number of brokered connections per namespace**              | 100                                                                           | 5,000                                                                         | 10,000 per PU (e.g., 4 PUs = 40,000)                                       | 100,000 per CU                     |
| **Maximum retention period of event data**                    | 1 day                                                                         | 7 days                                                                        | 90 days                                                                    | 90 days                            |
| **Event storage for retention**                               | 84 GB per TU                                                                  | 84 GB per TU                                                                  | 1 TB per PU                                                                | 10 TB per CU                       |
| **Maximum TUs or PUs or CUs**                                 | 40 TUs                                                                        | 40 TUs                                                                        | 16 PUs                                                                     | 20 CUs                             |
| **Number of partitions per event hub**                        | 32                                                                            | 32                                                                            | 100 per event hub (200 per PU at namespace level, e.g., 2 PUs = 400 total) | 1,024 per event hub (2,000 per CU) |
| **Number of namespaces per subscription**                     | 1,000                                                                         | 1,000                                                                         | 1,000                                                                      | 1,000 (50 per CU)                  |
| **Number of event hubs per namespace**                        | 10                                                                            | 10                                                                            | 100 per PU                                                                 | 1,000                              |
| **Capture**                                                   | N/A                                                                           | Pay per hour                                                                  | Included                                                                   | Included                           |
| **Size of compacted event hub**                               | N/A                                                                           | 1 GB per partition                                                            | 250 GB per partition                                                       | 250 GB per partition               |
| **Size of the schema registry (namespace) in megabytes**      | N/A                                                                           | 25                                                                            | 100                                                                        | 1,024                              |
| **Number of schema groups in a schema registry or namespace** | N/A                                                                           | 1 (excluding default group)                                                   | 100 (1 MB per schema)                                                      | 1,000 (1 MB per schema)            |
| **Number of schema versions across all schema groups**        | N/A                                                                           | 25                                                                            | 1,000                                                                      | 10,000                             |
| **Throughput per unit**                                       | Ingress: 1 MB/sec or 1,000 events/sec<br>Egress: 2 MB/sec or 4,096 events/sec | Ingress: 1 MB/sec or 1,000 events/sec<br>Egress: 2 MB/sec or 4,096 events/sec | No limits per PU\*                                                         | No limits per CU\*                 |

## Abbreviations

- **TU**: Throughput Unit = pre-purchased compute resource
- **PU**: Processing Unit = same as TU but isolated
- **CU**: Capacity Unit = same as above but for Dedicated tier only

- **Pricing mechanics**: Basic/Standard are billed per events/throughput units; Premium and Dedicated use reserved/dedicated capacity models and are priced accordingly

## AMQP vs. HTTPS

- **Initialization**: AMQP requires a persistent bidirectional socket plus TLS or SSL/TLS, resulting in _higher initial network costs_. HTTPS has extra TLS overhead for each request.
- **Performance**: AMQP offers _higher throughput and lower latency_ for frequent publishers. HTTPS can be slower due to the extra overhead.

## [Features](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-features)

### Namespace

An Event Hubs namespace is a management container for event hubs. It provides DNS-integrated network endpoints and a range of access control and network integration management features such as IP filtering, virtual network service endpoint, and Private Link.

### Event Retention

Published events are removed from an event hub based on a configurable, time-based retention policy. The default value and shortest possible retention period is 1 hour.

Max retention periods:

- Standard: 7 days
- Premium and Dedicated: 90 days.

If you need to archive events beyond the allowed retention period, you can have them automatically stored in Azure Storage or Azure Data Lake by turning on the Event Hubs Capture feature.

### [Event Hubs Capture](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-capture-overview)

Built-in feature of Azure Event Hubs that automatically captures the streaming event data and saves it to an  **Azure Blob storage** or **Azure Data Lake Storage**. It can process real-time and batch-based pipelines on the same stream. You can specify the time or size interval for capturing, and it scales automatically with throughput units.

It is a durable buffer for telemetry ingress (similar to a distributed log) with a partitioned consumer model. Captured data is written in Apache Avro format.

Storeage accounts can be in the same region as your event hub or in another region.

Capture allows you to set up a minimum size (default: 300 mb) and time window (default: 5 min) for capturing data (_capture windowing_). The "first wins policy" means the first trigger encountered initiates the capture operation. Each partition captures data independently and creates a block blob when the capture interval is reached, named after that time.

Example:

```txt
https://mystorageaccount.blob.core.windows.net/mycontainer/mynamespace/myeventhub/0/2017/12/08/03/03/17.avro
{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}
```

Integration with Event Grid: Create an Event Grid subscription with an Event Hubs namespace as its source.

Azure Storage account as a destination: Needs write permissions on blobs and containers level. The `Storage Blob Data Owner` is a built-in role with above permissions.

### Log Compaction

Azure Event Hubs supports compacting event log to retain the latest events of a given event key. With compacted event hubs/Kafka topic, you can use key-based retention rather than using the coarser-grained time-based retention.

## [Scaling to throughput units](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas)

Traffic is managed by throughput units. One unit permits 1 MB or 1000 events per second incoming (ingress), and twice that outgoing (egress). Standard hubs support 1-20 units (you can buy more). Exceeding your units limit will be throttled. Event Hubs Capture directly copies data and bypasses outgoing limits.

To scale your event processing app, run multiple instances using **EventProcessorClient** and let them balance the load. Event processor instances usually handle data from several partitions (_distributed ownership_). They claim ownership of partitions through entries in a checkpoint store. All processors update this store to manage their state and balance the workload.

Designing large systems:

- **Scale**: Have several readers, each handling some of the event hub partitions.
- **Load Balance**: Adjust the number of readers as needed. If a new type of sensor is added, the operator can increase readers to handle more data.
- **Resume After Failures**: If a reader fails, others take over from where it left off.
- **Consume Events**: There must be code to process the data, like combining it and saving it for the webpage.

## [Event Processor](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-event-processor-host)

- **Receiving Messages**: Create an event processor to handle specific partition events. Include retry logic to process every message at least once, and use two consumer groups for storage and routing needs.
- **Checkpointing**: The event processor marks the last processed event within a partition, allowing for resiliency. If an event processor disconnects, another can resume at the last checkpoint, and it's possible to return to older data by specifying a lower offset.

## Partitions

They serve as "commit logs" for organizing sequences of events, with **new events added in the order they were received**. 4 by default. They enhance raw IO throughput by allowing multiple parallel logs and streamline processing by assigning clear ownership, thus efficiently handling large volumes of events. The number of partitions, set within an allowed tier range at creation, influences throughput but not cost, and CANNOT be changed later. While increasing partitions can boost throughput, it may complicate processing. Balancing scaling units and partitions, with a guideline of 1 MB/s per partition, is recommended for optimal scale. The key directs events to specific partition, allowing related events to be grouped together by attributes like unique identity or geography.

## Control access to events

- [Data Owner](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#azure-event-hubs-data-owner): _complete access_ to Event Hubs resources.
- [Data Sender](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#azure-event-hubs-data-sender): _send access_ to Event Hubs resources.
- [Data Receiver](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#azure-event-hubs-data-receiver): _receiving access_ to Event Hubs resources.

## SAS Policy to RBAC Role Mapping

| SAS Policy Claim (Legacy) | Equivalent RBAC Role |
| ------------------------- | -------------------- |
| Send                      | Data Sender          |
| Listen                    | Data Receiver        |
| Manage                    | Data Owner           |

- Every namespace has a `RootManageSharedAccessKey` that has all the rights on the entire namespace, so you should RARELY USE THIS and you should never share it.

- `SAS authentication` can be handled at the namespace level, or it can be at the specific hub level for granular control.

## [Working with Event Hubs](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-dotnet-standard-getstarted-send)

```cs
// Connection strings and Event Hub name
var eventHubsConnectionString = "Endpoint=sb://example-namespace.servicebus.windows.net/;SharedAccessKeyName=KeyName;SharedAccessKey=AccessKey";
var eventHubName = "example-event-hub";
var storageConnectionString = "DefaultEndpointsProtocol=https;AccountName=exampleaccount;AccountKey=examplekey;EndpointSuffix=core.windows.net";
var blobContainerName = "example-container";
var consumerGroup = EventHubConsumerClient.DefaultConsumerGroupName;

// Alt to connection string: ClientSecretCredential, DefaultAzureCredential with fullyQualifiedNamespace

// Application Groups: You can connect via SAS or Entra ID (just pass credential to EventHubProducerClient), allowing you to use access policies, throttling, etc.
await using (var producerClient = new EventHubProducerClient(eventHubsConnectionString, eventHubName))
{
    string[] partitionIds = await producerClient.GetPartitionIdsAsync(); // Query partition IDs

    // Publish events to Event Hubs

    // Create a batch of events
    using EventDataBatch eventBatch = await producerClient.CreateBatchAsync();
    // Add events to the batch. An event is represented by a collection of bytes and metadata.
    eventBatch.TryAdd(new EventData(Encoding.UTF8.GetBytes("First event")));
    eventBatch.TryAdd(new EventData(Encoding.UTF8.GetBytes("Second event")));
    // Use the producer client to send the batch of events to the event hub
    await producerClient.SendAsync(eventBatch);
}

// Using buffer
// EventHubBufferedProducerClient abstracts away the complexities of batching and sending events, making it easier to use but with less control.
using(var bufferedProducerClient = new EventHubBufferedProducerClient(connectionString, eventHubName))
{
    await bufferedProducerClient.EnqueueEventAsync(new EventData(Encoding.UTF8.GetBytes("First event")));
    await bufferedProducerClient.EnqueueEventAsync(new EventData(Encoding.UTF8.GetBytes("Second event")));
}

// Read events from Event Hubs
await using (var consumer = new EventHubConsumerClient(consumerGroup, eventHubsConnectionString, eventHubName))
{
    using var cancellationSource = new CancellationTokenSource(TimeSpan.FromSeconds(45));
    await foreach (PartitionEvent receivedEvent in consumer.ReadEventsAsync(cancellationSource.Token)) {} // Wait for events
}

// Read events from an Event Hubs partition
// The events will be returned in the order they were added to that partition
await using (var consumer = new EventHubConsumerClient(consumerGroup, eventHubsConnectionString, eventHubName))
{
    EventPosition startingPosition = EventPosition.Earliest;
    string partitionId = (await consumer.GetPartitionIdsAsync()).First();

    using var cancellationSource = new CancellationTokenSource(TimeSpan.FromSeconds(45));
    await foreach (PartitionEvent receivedEvent in consumer.ReadEventsFromPartitionAsync(partitionId, startingPosition, cancellationSource.Token)) // Wait for events in partition
    {
        string readFromPartition = partitionEvent.Partition.PartitionId;
        byte[] eventBody = partitionEvent.Data.EventBody.ToArray();
    }
}

// Process events using Event Processor client
// NOTE: You need Blob Storage for checkpointing
var storageClient = new BlobContainerClient(storageConnectionString, blobContainerName);
var processor = new EventProcessorClient(storageClient, consumerGroup, eventHubsConnectionString, eventHubName);
Task processEventHandler(ProcessEventArgs eventArgs) => {
    // Checkpointing: Update checkpoint in the blob storage so that you can resume from this point if the processor restarts
    await eventArgs.UpdateCheckpointAsync();
};
Task processErrorHandler(ProcessErrorEventArgs eventArgs) => Task.CompletedTask;

processor.ProcessEventAsync += processEventHandler;
processor.ProcessErrorAsync += processErrorHandler;

await processor.StartProcessingAsync();
try { await Task.Delay(Timeout.Infinite, new CancellationTokenSource(TimeSpan.FromSeconds(45)).Token); } catch (TaskCanceledException) {}
try { await processor.StopProcessingAsync(); } finally
{
    processor.ProcessEventAsync -= processEventHandler;
    processor.ProcessErrorAsync -= processErrorHandler;
}
```

### CLI

```sh
# Create a resource group
az group create --name $resourceGroup --location eastus

# Create an Event Hubs namespace
# Throughput units are specified here
az eventhubs namespace create --name $eventHubNamespace --sku Standard --location eastus --resource-group $resourceGroup

# Get the connection string for a namespace
az eventhubs namespace authorization-rule keys list --namespace-name $eventHubNamespace --name RootManageSharedAccessKey --resource-group $resourceGroup

# Create an Event Hub inside the namespace
# Partition count and retention days are specified here
az eventhubs eventhub create --name $eventHub --namespace-name $eventHubNamespace --partition-count 2 --message-retention 1 --resource-group $resourceGroup

# Get the connection string for a specific event hub within a namespace
az eventhubs eventhub authorization-rule keys list --namespace-name $eventHubNamespace --eventhub-name $eventHubName --name MyAuthRuleName --resource-group $resourceGroup

# Create a Consumer Group (Consumer Groups)
az eventhubs eventhub consumer-group create --eventhub-name $eventHub --name MyConsumerGroup --namespace-name $eventHubNamespace --resource-group $resourceGroup

# Capture Event Data (Event Hubs Capture)
# Enable capture and specify the storage account and container
az eventhubs eventhub update --name $eventHub --namespace-name $eventHubNamespace --enable-capture True --storage-account sasurl --blob-container containerName --resource-group $resourceGroup

# Scale the throughput units (Throughput Units)
az eventhubs namespace update --name $eventHubNamespace --sku Standard --capacity 2 --resource-group $resourceGroup

# Get Event Hub details (Partitions, Consumer Groups)
az eventhubs eventhub show --name $eventHub --namespace-name $eventHubNamespace --resource-group $resourceGroup

# Delete the Event Hub Namespace (this will delete the Event Hub and Consumer Groups within it)
az eventhubs namespace delete --name $eventHubNamespace --resource-group $resourceGroup
```

# Events

## General Comparison

| Feature         | Azure Event Hubs                           | Azure Event Grid                          |
| --------------- | ------------------------------------------ | ----------------------------------------- |
| **Data**        | Handles high-volume data                   | Focuses on event, not payload             |
| **Integration** | Works with Azure Stream Analytics          | Built-in Azure service integration        |
| **Pricing**     | Charges by data ingested                   | Charges per operation, 🏷️ for low payload |
| **Scalability** | Millions of events/sec, ideal for big data | Auto-scales, limited max throughput       |
| **Use Case**    | Real-time analytics, large data volumes    | Real-time event processing                |

<br>

## Event Grid vs Service Bus: Event vs Message

| Feature           | Event (for Event Grid)                 | Message (for Service Bus)                 |
| :---------------- | :------------------------------------- | :---------------------------------------- |
| **Purpose**       | Notification (A state changed)         | Command/Data (Do this work)               |
| **Content**       | Lightweight, context-only              | Contains the full data payload            |
| **Sender Intent** | "Just letting you know this happened." | "Please process this data."               |
| **Pattern**       | Publisher/Subscriber (Pub/Sub)         | Queue (Point-to-Point) or Topic (Pub/Sub) |
| **Coupling**      | Fully decoupled                        | Decoupled via intermediary (queue/topic)  |

<br>

# [Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/)

## [Introduction](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview)

Azure Functions, a serverless compute service, allows you to execute small code snippets in response to events or on a schedule, eliminating the need to manage servers. It triggers your code based on specific events and simplifies data handling. Typical use cases include processing file uploads, handling real-time data streams, performing machine learning tasks, running scheduled tasks, building scalable web APIs, orchestrating serverless workflows, responding to database changes, and establishing reliable message systems.

## [Azure Functions vs Azure Logic Apps vs WebJobs](https://docs.microsoft.com/en-us/azure/azure-functions/functions-compare-logic-apps-ms-flow-webjobs)

| Feature          | Azure Functions                                                                        | Azure Logic Apps                                                         | Azure WebJobs                                                                                                      |
| ---------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| Development      | Code-first approach                                                                    | Designer-first approach                                                  | Code-first approach                                                                                                |
| Monitoring       | Azure Application Insights                                                             | Azure portal and Azure Monitor logs                                      | Application Insights                                                                                               |
| Execution        | Can run in Azure or locally                                                            | Can run in Azure, locally, or on premises                                | Runs in the context of an App Service web app                                                                      |
| Pricing          | Pay-per-use                                                                            | Based on execution and connector usage                                   | Part of the App Service plan                                                                                       |
| Unique Strengths | Flexibility and cost-effectiveness, many supported languages, built on the WebJobs SDK | Extensive integration capabilities with a large collection of connectors | Ideal for tasks related to an existing App Service app and need more control over the code that listens for events |

## [Hosting Options](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale)

Consumption is the lowest plan that supports scaling and offer event based scheduling behavior.

### Azure Functions Hosting Plans Comparison

| Plan                                                                                                           | Cost Model                                                   | Cold Start                                                                                    | Scale                                        | Best For                                                                                                                                                             | Limitations                                                                                                                                                                              |
| -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [**Consumption**](https://learn.microsoft.com/en-us/azure/azure-functions/consumption-plan)                    | Serverless (Pay per use)                                     | When idle or scaled down to zero instances                                                    | Event-driven                                 | - Low-traffic, cost-sensitive workloads<br/>- Variable or unpredictable workloads                                                                                    | - No container support<br/>- 5–10 min timeout<br/>- Max instances: 100 (Linux), 200 (Windows)<br/>- No VNET integration<br/>- No support for durable functions chaining large executions |
| [**Flex Consumption**](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan)          | Serverless (Pay per use) + memory for always-ready instances | Mitigates cold starts during scaling via always-ready instances                               | Per-function                                 | - Low-latency apps needing faster startup                                                                                                                            | - No containers or Windows support<br/>- Regional availability limited                                                                                                                   |
| [**Premium**](https://learn.microsoft.com/en-us/azure/azure-functions/functions-premium-plan)                  | Predictable: runtime + memory for pre-warmed instances       | No cold starts (minimum instance count always warm)                                           | Event-driven                                 | - Production-grade apps<br/>- Long-running tasks<br/>- Custom images                                                                                                 | - Must keep at least one instance warm<br/>- Higher cost                                                                                                                                 |
| [**Dedicated**](https://learn.microsoft.com/en-us/azure/azure-functions/dedicated-plan)                        | Predicatble (Fixed) - App Service Plan pricing               | Avoided for the always-on instance, but may occur during scale-out (no pre-warmed instances). | Manual or App Service autoscale (rule-based) | - App Service integration (reuse VMs)<br/>- Run multiple apps on one plan<br/>- Access to larger compute sizes<br/>- Full isolation (VNET) & secure networking (ASE) | - Not cost-effective unless fully utilized<br/>- Manual scaling unless using App Service autoscale                                                                                       |
| [**Container Apps**](https://learn.microsoft.com/en-us/azure/azure-functions/functions-container-apps-hosting) | Azure Container Apps plan (consumption / dedicated)          | Depends on `minReplicas`: 1+ avoids cold start                                                | Event-driven (KEDA based)                    | - Custom runtimes/libraries<br/>- Legacy/on-prem to containers<br/>- No Kubernetes management<br/>- GPU-powered functions                                            | - Cold start unless minReplica ≥ 1<br/>- Complexer setup                                                                                                                                 |

#### Scaling

- [Event driven](https://learn.microsoft.com/en-us/azure/azure-functions/event-driven-scaling): The _scale controller_ adjusts resources based on event rates and trigger types. It uses heuristics for each trigger type (for Queue storage trigger, it scales based on the queue length and the age of the oldest queue message).
- [Per function](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan#per-function-scaling): Most functions scale independently, while HTTP, Blob (Event Grid), and Durable Function triggers each scale as grouped sets on shared instances.
- [Rule based (autoscale)](https://learn.microsoft.com/en-us/azure/azure-monitor/autoscale/autoscale-get-started): Unlike event-driven scaling, it doesn't directly respond to function invocations, but to the resource metrics (CPU, memory, HTTP queue length) or schedules.

#### CLI

```sh
az functionapp plan create
    --name
    --resource-group
    --sku # F1(Free), D1(Shared), B1(Basic Small), B2(Basic Medium), B3(Basic Large), S1(Standard Small), P1V2(Premium V2 Small), I1 (Isolated Small), I2 (Isolated Medium), I3 (Isolated Large), K1 (Kubernetes)
    [--is-linux {false, true}]
    [--location]
    [--max-burst]
    [--min-instances]
    [--tags]
    [--zone-redundant] # Cannot be changed after plan creation. Minimum instance count is 3.
# az functionapp plan create -g $resourceGroup -n MyPlan --min-instances 1 --max-burst 10 --sku EP1

# Get a list of all Consumption plans in your resource group
az functionapp plan list --resource-group $resourceGroup --query "[?sku.family=='Y'].{PlanName:name,Sites:numberOfSites}" -o table

# Get your hosting plan type
appServicePlanId=$(az functionapp show --name $functionApp --resource-group $resourceGroup --query appServicePlanId --output tsv)
az appservice plan list --query "[?id=='$appServicePlanId'].sku.tier" --output tsv

# Get a list of all Premium plans in your resource group
az functionapp plan list --resource-group $resourceGroup --query "[?sku.family=='EP'].{PlanName:name,Sites:numberOfSites}" -o table
```

## [Storage Considerations](https://learn.microsoft.com/en-us/azure/azure-functions/storage-considerations?tabs=azure-cli)

- Function code and configuration files are stored in Azure Files in the linked storage account. Deleting this account will result in the loss of these files.
- Azure Functions requires an Azure Storage account for services like Azure Blob Storage, Azure Files, Azure Queue Storage, and Azure Table Storage.
- Storage accounts used by function apps must support Blob, Queue, and Table storage.
- The storage account should be in the same region as the function app for performance optimization.
- Each function app should use a separate storage account for best performance.
- Azure Storage encrypts all data in a storage account at rest.
- Functions use a host ID value to uniquely identify a function app in stored artifacts. Host ID collisions can occur and should be avoided.

## [Configuration](https://learn.microsoft.com/en-us/azure/azure-functions/functions-how-to-use-azure-function-app-settings)

The following languages can be used directly in Azure Portal (no external editor needed): `C# Script`, `JavaScript`, `Python`, `PowerShell`.

- **Application Settings**: Cloud-based env vars. Securely store secrets (e.g. connection strings). Set via Portal/CLI/ARM. Accessed via `%VAR%` or `Environment.GetEnvironmentVariable()`.
- **function.json**: Per-function bindings config (script languages only). Declares triggers, inputs, outputs. References **Application Settings** by name.
- **host.json**: App-wide runtime config. Controls logging, timeouts, retries, and extension settings.
- **local.settings.json**: Local dev only. `"Values"` mimic **Application Settings**. Never deploy or commit (can contain secrets).

### [host.json](https://learn.microsoft.com/en-us/azure/azure-functions/functions-host-json)

- `functionTimeout`: Default: 5 min for Consumption, 30 for rest.
- `logging.applicationInsights`
- `aggregator` - Specifies how many function invocations are aggregated when calculating metrics for Application Insights.
- `extensions.http.dynamicThrottlesEnabled`: Enabled by default for Consumption only. Throttles on high resource usage.
- `extensions.blobsmaxDegreeOfParallelism`: concurrent invocations allowed for all blob-triggered functions (min: 1, default: 8)
- `extensions.cosmosDB.connectionMode`: _Gateway_ (default) or _Direct_. _Gateway_ is preferable for _Consumption_ plan due to connections limit. _Direct_ has better performance.
- `extensions.cosmosDB.userAgentSuffix`: Appends the given string to all service requests from the trigger or binding, enhancing tracking in Azure Monitor by function app and User Agent filtering.

### [function.json](https://github.com/Azure/azure-functions-host/wiki/function.json)

Auto generated for compiled languages.  
For scripting languages, like `C# Script`, `Python`, you must provide the config file yourself.

- For triggers, the direction is always `in`
- Input and output bindings use `in` and `out`, or `inout` in some cases.
- `connection`: refers to an environment variable holding the connection string. **It never contains the actual secret**. Define the connection string in `Application Settings`.

```jsonc
{
  "bindings": [
    {
      "type": "queueTrigger",
      "direction": "in",
      "name": "order",
      "queueName": "myqueue-items",
      "connection": "MY_STORAGE_ACCT_APP_SETTING"
    },
    {
      "type": "table",
      "direction": "out",
      "name": "$return",
      "tableName": "outTable",
      "connection": "MY_TABLE_STORAGE_ACCT_APP_SETTING"
    }
  ]
}
```

### Configuration via CLI

Setting properties: `az resource update --resource-type Microsoft.Web/sites -g $resourceGroup -n <FUNCTION_APP-NAME>/config/web --set properties.XXX`, where `XXX` is the name of the property.

- `functionAppScaleLimit`: 0 or null for unrestricted, or a valid value between 1 and the app maximum (200 for Consumption, 100 for premium).

### [local.settings.json](https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-local) (code and test locally)

```ts
type LocalSettings = {
  // When true, all values are encrypted with a local machine key.
  IsEncrypted: boolean; // false

  // Correspond to application settings in your function app in Azure.
  Values: {
    [key: string]: string;
  };

  // Customize the Functions host process
  Host: {
    LocalHttpPort: number;

    CORS: string; // supports wildcard value (*)

    // When set to true, allows withCredentials requests
    CORSCredentials: boolean;
  };

  // Used by frameworks that get connection strings from the ConnectionStrings section of a config file
  ConnectionStrings: {
    [key: string]: string;
  };
};
```

## [Triggers and Bindings](https://learn.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings?tabs=csharp)

Triggers cause a function to run. A trigger defines how a function is invoked and a function must have exactly one trigger. Binding to a function is a way of declaratively connecting another resource to the function; bindings may be connected as _input bindings_ (read), _output bindings_ (write), or both.

Triggers and bindings must be defined in C# (also supported: Python, Java, TypeScript, PS)

Cannot be trigger: _Table Storage_  
Cannot be input binding: _Event Grid_, _Event Hubs_, _HTTP & webhooks_, _IoT Hub_, _Queue storage_, _SendGrid_, _Service Bus_, _Timer_  
Cannot be output binding: _IoT Hub_, _Timer_  
Available by default ([others need to be installed as separate package](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-register)): _Timer_, _HTTP & webhooks_  
Not supported on Consumption plan ([requires runtime-driven triggers](https://learn.microsoft.com/en-us/azure/azure-functions/functions-networking-options?tabs=azure-cli#premium-plan-with-virtual-network-triggers)): _RabbitMQ_, _Kafka_

### Triggers and Bindings Gist

```cs
// https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook
// Set Route to a string like "products/{id}" to create a custom URL for the function. This makes it accessible at https://<APP_NAME>.azurewebsites.net/api/<FUNCTION_NAME>/products/{id}
[HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = "blob/{name}")] HttpRequest req, string name;
[HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequestMessage req; // return req.CreateResponse(HttpStatusCode.OK, string);

// https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-blob
[BlobTrigger("container/{name}")] string myBlob, string name;
[Blob("container/{name}", FileAccess.Read)] Stream myBlob; // or BlobContainerCLient, or BlobCLient
[Blob("container/{name}", FileAccess.Write)] Stream myBlob;
[Blob("container/{name}.jpg", FileAccess.Read)] Stream myBlob; // Wildcard in Blob Name: a jpg file
[BlobOutput("test-samples-output/{name}-output.txt")]; // return obj;

// https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-cosmosdb-v2
[CosmosDBTrigger(
            databaseName: "databaseName",
            containerName: "containerName",
            Connection = "CosmosDBConnectionSetting",
            LeaseContainerName = "leases",
            CreateLeaseContainerIfNotExists = true)]IReadOnlyList<ToDoItem> input;
[CosmosDB(databaseName:"myDb", collectionName:"collection", Id = "{id}", PartitionKey ="{partitionKey}")] dynamic document; // input
[CosmosDB(databaseName:"myDb", collectionName:"collection", CreateIfNotExists = true)] out dynamic document; // output

// https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-grid
[EventGridTrigger]EventGridEvent ev; // ev.Data
// No Input binding
[return: EventGrid(TopicEndpointUri = "EventGridTopicUriAppSetting", TopicKeySetting = "EventGridTopicKeyAppSetting")] // return new EventGridEvent(...); or new CloudEvent(...)
[EventGrid(TopicEndpointUri = "EventGridTopicUriAppSetting", TopicKeySetting = "EventGridTopicKeyAppSetting")] out eventGridEvent;
[EventGrid(TopicEndpointUri = "EventGridTopicUriAppSetting", TopicKeySetting = "EventGridTopicKeyAppSetting")]IAsyncCollector<EventGridEvent> outputEvents; // (batch processing): outputEvents.AddAsync(myEvent)

// https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-hubs
[EventHubTrigger("hub", Connection = "EventHubConnectionAppSetting")] EventData[] events; // var messageBody = Encoding.UTF8.GetString(eventData.Body.Array, eventData.Body.Offset, eventData.Body.Count);
// No Input binding
[return: EventHub("outputEventHubMessage", Connection = "EventHubConnectionAppSetting")] // return string
[EventHub("outputEventHubMessage", Connection = "EventHubConnectionAppSetting")] IAsyncCollector<string> outputEvents; // (batch processing): outputEvents.AddAsync(string)

// https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-iot
[IoTHubTrigger("messages/events", Connection = "IoTHubConnectionAppSetting")]EventData message; // Encoding.UTF8.GetString(message.Body.Array)

// https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-queue
[QueueTrigger("queue", Connection = "StorageConnectionAppSetting")]string myQueueItem;
// No Input binding
[return: Queue("queue")] // return string

// https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-service-bus
[ServiceBusTrigger("queue", Connection = "ServiceBusConnectionAppSetting")] string myQueueItem;
// No Input binding
[return: ServiceBus("queue", Connection = "ServiceBusConnectionAppSetting")] // return string


// https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer
// The 6-field format for cron jobs is `{second} {minute} {hour} {day} {month} {day-of-week}`. The 5-field format omits the `second` and starts with `{minute}`.
// - Specific value: `5` (exactly the 5th minute)
// - List: `5,10` (5th and 10th minute)
// - Range: `9-17` (from 9 to 17)
// - Step: `*/5` (every 5 units)
// - Any value: `*` (every unit)
// NOTE: Day of week: MON is 1, Sunday is 0 or 7; Day and Month start from 1
// Examples: `*/5 * * * *`: Every 5 minutes,; `0 9-17 * * MON-FRI`: 9 AM to 5 PM on weekdays.
[TimerTrigger("0 */5 * * * *")] TimerInfo myTimer;
// - `WEBSITE_TIME_ZONE` and `TZ` are not currently supported on the Linux Consumption plan.
// - RunOnStartup is not recommended for production (messes up schedule). Schedule, RunOnStartup and UseMonitor can be set in local.settings.json > Values
```

## Working with Azure Functions

```sh
# List the existing application settings
az functionapp config appsettings list --name $name --resource-group $resourceGroup

# Add or update an application setting
az functionapp config appsettings set --settings CUSTOM_FUNCTION_APP_SETTING=12345 --name $name --resource-group $resourceGroup

# Create a new function app (Consumption)
az functionapp create --resource-group $resourceGroup --name $consumptionFunctionName --consumption-plan-location $regionName --runtime dotnet --functions-version 3 --storage-account $storageName

# Get the default (host) key that can be used to access any HTTP triggered function in the function app
subName='<SUBSCRIPTION_ID>'
resGroup=AzureFunctionsContainers-rg
appName=glengagtestdocker
path=/subscriptions/$subName/resourceGroups/$resGroup/providers/Microsoft.Web/sites/$appName/host/default/listKeys?api-version=2018-11-01
az rest --method POST --uri $path --query functionKeys.default --output tsv
```

## [Security](https://learn.microsoft.com/en-us/azure/azure-functions/security-concepts)

### [Authorization level](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook-trigger#http-auth)

Indicates the kind of authorization key that's required to access the function endpoint, via `code` param: `https://<APP_NAME>.azurewebsites.net/api/<FUNCTION_NAME>?code=<API_KEY>`.

- **Anonymous**: No API key is required.
- **Function** (default): A function-specific or host-wide API key is required. The best practice as per _least privilege_ principle.
- **Admin**: The master key is required. For administrative tasks, management scripts.

#### Access scopes

- **Function** keys grant access only to the specific function they're defined under.
- **Host** keys allow access to all functions within the function app.
  - **master**. provides administrative access to the runtime REST APIs. This key can't be revoked.

Each key is named, with a `default` key at both levels. If a function and a host key share a name, the function key takes precedence.

#### [Working with access keys](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook-trigger#obtaining-keys)

Base URL: `https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Web/sites/{name}/{scope}/{host-or-function-name}/{action}?api-version=2022-03-01`

Scope can be `functions` or `host`. For slots add `/slots/{slot-name}/` before scope.

List keys: `POST`, action: `listkeys`  
Create or update keys: `PUT`, action: `keys/{keyName}`
Delete or revoke keys: `DELETE`, action: `/keys/{keyName}`

### [Client identities](https://learn.microsoft.com/en-us/azure/app-service/overview-managed-identity#rest-endpoint-reference)

`ClaimsPrincipal identity = req.HttpContext.User;` available via `X-MS-CLIENT-PRINCIPAL` [header](https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-user-identities#access-user-claims-in-app-code).

### CORS

```sh
# Add a domain to the allowed origins list
az functionapp cors add --allowed-origins https://contoso.com --name $name --resource-group $resourceGroup

# List the current allowed origins
az functionapp cors show
```

## [Monitoring](https://learn.microsoft.com/en-us/azure/azure-functions/functions-monitoring)

Automatic collection of Performance Counters isn't supported when running on Linux.

## [Enable Application Insights](https://learn.microsoft.com/en-us/azure/azure-functions/configure-monitoring?tabs=v2#enable-application-insights-integration)

Enabled by default for new functions (created in the same or nearest region to your function app), but it may have to be manually enabled for old functions.

To send data, you need the key named `APPINSIGHTS_INSTRUMENTATIONKEY`. `ILogger` is used (not `TelemetryClient`!). Azure Functions use [Adaptive sampling](./Application%20Insights.md).

Application Insights are configured in [host.json](https://learn.microsoft.com/en-us/azure/azure-functions/functions-host-json#applicationinsights) (`logging.applicationInsights` and `aggregator`)

### [Configure monitorung](https://learn.microsoft.com/en-us/azure/azure-functions/configure-monitoring)

Enable SQL query:

```json
{
  "logging": {
    "applicationInsights": {
      "enableDependencyTracking": true,
      "dependencyTrackingOptions": {
        "enableSqlCommandTextInstrumentation": true
      }
    }
  }
}
```

Turn on verbose logging from the scale controller to Application Insights:

```sh
az functionapp config appsettings set --settings SCALE_CONTROLLER_LOGGING_ENABLED=AppInsights:Verbose \
--name $name --resource-group $resourceGroup
```

Disable logging:

```sh
az functionapp config appsettings delete --setting-names SCALE_CONTROLLER_LOGGING_ENABLED \
--name $name --resource-group $resourceGroup
```

#### Categories

Identify which part of the system or user code generated the log.

- `Function.<YOUR_FUNCTION_NAME>`: Relates to dependency data, custom metrics and events, trace logs for function runs, and user-generated logs.
- `Host.Aggregator`: Provides aggregated counts and averages of function invocations.
- `Host.Results`: Records the success or failure of functions.
- `Microsoft`: Reflects a .NET runtime component invoked by the host.
- `Worker`: Logs generated by language worker processes for non-.NET languages.

#### [Solutions with high volume of telemetry](https://learn.microsoft.com/en-us/azure/azure-functions/configure-monitoring?tabs=v2#solutions-with-high-volume-of-telemetry)

```jsonc
{
  "version": "2.0",
  "logging": {
    "logLevel": {
      "default": "Warning",
      "Function": "Error",
      // Be aware of the `flushTimeout` (in aggregator) delay if you set a different value than Information
      "Host.Aggregator": "Error",
      "Host.Results": "Information",
      "Function.Function1": "Information",
      "Function.Function1.User": "Error"
    },
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "maxTelemetryItemsPerSecond": 1,
        "excludedTypes": "Exception"
      }
    }
  }
}
```

## [Custom Handlers](https://learn.microsoft.com/en-us/azure/azure-functions/functions-custom-handlers)

Lightweight web servers that interact with the Azure Functions host. They can be implemented in any language that supports HTTP primitives.

When an event occurs, the Azure Functions host forwards the request to the custom handler's web server, which executes the function and returns the output for further processing by the host.

The custom handler web server needs to start within 60 seconds.

### Application structure

- `host.json` - use the `customHandler.description.defaultExecutablePath` property to set the executable path and arguments
- `local.settings.json` - set `FUNCTIONS_WORKER_RUNTIME` to "custom" for local development
- A command / script / executable, which runs a web server
- A `function.json` file for each function (**inside a folder that matches the function name**). Example: `MyFunctionName/function.json`

### HTTP-only function

`customHandler.enableForwardingHttpRequest` lets your HTTP-triggered function directly handle the original HTTP request and response instead of Azure's custom payloads. This makes your handler act more like a traditional web server. It's handy for simpler setups with no extra bindings or outputs, but keep in mind Azure Functions isn't a full-fledged reverse proxy—some features like HTTP/2 and WebSockets aren't supported. Think of it as giving your handler raw access to the HTTP action without Azure's middleman interference.

## Triggers and Bindings full samples

```cs
[HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = "blob/{name}")] HttpRequest req, string name;

[Blob("container/{name}", FileAccess.Read)] Stream myBlob; // FileAccess.Write

[CosmosDBTrigger(
    databaseName: "database",
    collectionName: "collection",
    ConnectionStringSetting = "CosmosDBConnection", // Note: this refers to env var name, not an actual connection string
    LeaseCollectionName = "leases")]IReadOnlyList<Document> input;
```

```cs
////////////////////////////////////
// Blob
////////////////////////////////////

[FunctionName("BlobTrigger")]
public static void RunBlob([BlobTrigger("container/{name}")] string myBlob, string name, ILogger log)
{
    log.LogInformation($"Blob trigger function processed blob\n Name:{name} \n Data: {myBlob}");
}


[FunctionName("BlobStorageInputBinding")]
public static void RunBlobStorageInputBinding(
    [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = "blob/{name}")] HttpRequest req, string name,
    [Blob("container/{name}", FileAccess.Read)] Stream myBlob,
    ILogger log)
{
    // Reads the content from the blob storage for further processing
    using StreamReader reader = new StreamReader(myBlob);
    string content = reader.ReadToEnd();
    log.LogInformation($"Blob Content: {content}");
}

// [CosmosDBTrigger(...)] IReadOnlyList<Document> input,
// [Blob("container/{input[0].Id}", FileAccess.Read)] Stream myBlob

[FunctionName("BlobStorageOutputBinding")]
public static async Task<IActionResult> RunBlobStorageOutputBinding(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "blob/{name}")] HttpRequest req, string name,
    [Blob("container/{name}", FileAccess.Write)] Stream myBlob,
    ILogger log)
{
    // Writes the request body to a blob
    string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    using StreamWriter writer = new StreamWriter(myBlob);
    writer.Write(requestBody);
    log.LogInformation($"Blob written: {name}");
    return new OkResult();
}

////////////////////////////////////
// CosmosDB
////////////////////////////////////

[FunctionName("CosmosDBTrigger")]
public static void RunCosmos([CosmosDBTrigger(
    databaseName: "database",
    collectionName: "collection",
    ConnectionStringSetting = "CosmosDBConnection", // Note: this refers to env var name, not an actual connection string
    LeaseCollectionName = "leases")]IReadOnlyList<MyObj> input, ILogger log)
{
    if (input != null && input.Count > 0)
    {
        log.LogInformation("Documents modified " + input.Count);
        log.LogInformation("First document Id " + input[0].Id);
    }
}

[FunctionName("CosmosDBInputBinding")]
public static void RunCosmosDBInputBinding(
    [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = "doc/{id}")] HttpRequest req, string id,
    [CosmosDB(databaseName:"myDb", collectionName:"collection", Id = "{id}", PartitionKey ="{partitionKey}")] dynamic document,
    ILogger log)
{
    // Retrieves a specific document from Cosmos DB for further processing
    log.LogInformation($"Document Content: {document}");
}

[FunctionName("CosmosDBOutputBinding")]
public static IActionResult RunCosmosDBOutputBinding(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "doc/{id}")] HttpRequest req, string id,
    [CosmosDB(databaseName:"myDb", collectionName:"collection", CreateIfNotExists =true)] out dynamic document,
    ILogger log)
{
    // Writes a new document to Cosmos DB
    string requestBody = new StreamReader(req.Body).ReadToEnd();
    document = new { id = id, content = requestBody };
    log.LogInformation($"Document written: {id}");
    return new OkResult();
}

////////////////////////////////////
// EventGrid
////////////////////////////////////

[FunctionName("EventGridTrigger")]
public static async Task RunEventGrid([EventGridTrigger]EventGridEvent eventGridEvent, ILogger log)
{
    log.LogInformation(eventGridEvent.Data.ToString());
}

// No Input binding

[FunctionName("EventGridOutputBinding")]
[return: EventGrid(TopicEndpointUri = "EventGridTopicUriAppSetting", TopicKeySetting = "EventGridTopicKeyAppSetting")]
public static async Task<EventGridEvent> RunEventGridOutputBinding(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "event/{subject}")] HttpRequest req, string subject,
    ILogger log)
{
    // Sends an event to Event Grid Topic
    string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    var eventGridEvent = new EventGridEvent(subject: "IncomingRequest", eventType: "IncomingRequest", dataVersion: "1.0", data: requestBody);
    log.LogInformation($"Event sent: {subject}");
    return eventGridEvent;
}

// Output with out eventGridEvent
[FunctionName("EventGridOutputBinding")]
[return: EventGrid(TopicEndpointUri = "EventGridTopicUriAppSetting", TopicKeySetting = "EventGridTopicKeyAppSetting")]
public static async void RunEventGridOutputBinding(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "event/{subject}")] HttpRequest req, string subject,
    EventGrid(TopicEndpointUri = "EventGridTopicUriAppSetting", TopicKeySetting = "EventGridTopicKeyAppSetting") out eventGridEvent
    ILogger log)
{
    // Sends an event to Event Grid Topic
    string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    eventGridEvent = new EventGridEvent(subject: "IncomingRequest", eventType: "IncomingRequest", dataVersion: "1.0", data: requestBody);
    log.LogInformation($"Event sent: {subject}");
}

// Output with out batch processing
[FunctionName("EventGridOutputBinding")]
public static async Task RunEventGridOutputBinding(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "event/{subject}")] HttpRequest req, string subject,
    [EventGrid(TopicEndpointUri = "EventGridTopicUriAppSetting", TopicKeySetting = "EventGridTopicKeyAppSetting")]IAsyncCollector<EventGridEvent> outputEvents,
    ILogger log)
{
    // Sends an event to Event Grid Topic
    string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    var myEvent = new EventGridEvent(subject: "IncomingRequest", eventType: "IncomingRequest", dataVersion: "1.0", data: requestBody);
    await outputEvents.AddAsync(myEvent);
    log.LogInformation($"Event sent: {subject}");
}

////////////////////////////////////
// EventHub
////////////////////////////////////

[FunctionName("EventHubTrigger")]
public static async Task RunEventHub([EventHubTrigger("hub", Connection = "EventHubConnectionAppSetting")] EventData[] events, ILogger log)
{
    foreach (EventData eventData in events)
    {
        string messageBody = Encoding.UTF8.GetString(eventData.Body.Array, eventData.Body.Offset, eventData.Body.Count);
        log.LogInformation($"Event Hub trigger function processed a message: {messageBody}");
    }
}

// No Input binding

[FunctionName("EventHubOutputBinding")]
[return: EventHub("outputEventHubMessage", Connection = "EventHubConnectionAppSetting")]
public static async string RunEventHubOutputBinding(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "event/{message}")] HttpRequest req, string message
    ILogger log)
{
    // Sends an event to Event Hub
    log.LogInformation($"Event sent: {message}");
    return message;
}

// Output with out batch processing
[FunctionName("EventHubOutputBinding")]
public static async Task RunEventHubOutputBinding(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "event/{message}")] HttpRequest req, string message,
    [EventHub("outputEventHubMessage", Connection = "EventHubConnectionAppSetting")] IAsyncCollector<string> outputEvents,
    ILogger log)
{
    // Sends an event to Event Hub
    await outputEvents.AddAsync(message);
    log.LogInformation($"Event sent: {message}");
}

////////////////////////////////////
// IoTHub
////////////////////////////////////

[FunctionName("IoTHubTrigger")]
public static async Task RunIoTHub([IoTHubTrigger("messages/events", Connection = "IoTHubConnectionAppSetting")]EventData message, ILogger log)
{
    log.LogInformation($"IoT Hub trigger function processed a message: {Encoding.UTF8.GetString(message.Body.Array)}");
}

// No Input binding

////////////////////////////////////
// Http
////////////////////////////////////

// Accessible via GET https://<APP_NAME>.azurewebsites.net/api/<FUNCTION_NAME>
// You can specify a custom route by setting Route to a string.
// For example, Route = "products/{id}" would make the function accessible at https://<APP_NAME>.azurewebsites.net/api/<FUNCTION_NAME>/products/{id}.

// Request length limit: 100 MB
// URL length limit: 4 KB
// Timeout: 230s (502 error)

[FunctionName("HttpTriggerFunction")]
public static async Task<HttpResponseMessage> Run([HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequestMessage req, ILogger log)
{
    log.LogInformation("C# HTTP trigger function processed a request.");
    return req.CreateResponse(HttpStatusCode.OK, "Hello from Azure Functions!");
}

// No Input binding

// No Output binding

////////////////////////////////////
// Queue
////////////////////////////////////

[FunctionName("QueueTrigger")]
public static void RunQueue([QueueTrigger("queue", Connection = "StorageConnectionAppSetting")]string myQueueItem, ILogger log)
{
    log.LogInformation($"Queue trigger function processed: {myQueueItem}");
}

// No Input binding

[FunctionName("QueueStorageOutputBinding")]
[return: Queue("queue")]
public static string RunQueueStorageOutputBinding(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "queue/{message}")] HttpRequest req, string message
    ILogger log)
{
    // Sends a message to Azure Queue Storage
    log.LogInformation($"Message sent: {message}");
    return message;
}

[FunctionName(nameof(AddMessages))]
public static void AddMessages(
[HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req,
[Queue("outqueue"), StorageAccount("AzureWebJobsStorage")] ICollector<string> msg,
ILogger log)
{
    msg.Add("First");
    msg.Add("Second");
}

////////////////////////////////////
// ServiceBus
////////////////////////////////////

[FunctionName("ServiceBusTrigger")]
public static async Task RunServiceBus([ServiceBusTrigger("queue", Connection = "ServiceBusConnectionAppSetting")] string myQueueItem, ILogger log)
{
    log.LogInformation($"Service Bus Queue trigger function processed message: {myQueueItem}");
}

// No Input binding

[FunctionName("ServiceBusOutputBinding")]
[return: ServiceBus("queue", Connection = "ServiceBusConnectionAppSetting")]
public static string RunServiceBusOutputBinding(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "servicebus/{message}")] HttpRequest req, string message,
    ILogger log)
{
    // Sends a message to Service Bus Queue
    log.LogInformation($"Message sent: {message}");
    return message;
}

////////////////////////////////////
// Timer
////////////////////////////////////

[FunctionName("TimerTriggerFunction")]
public static void Run(
    [TimerTrigger("0 */5 * * * *")] TimerInfo myTimer, ILogger log)
{
    log.LogInformation($"C# Timer trigger function executed at: {DateTime.Now}");
}

// No Input binding

// No Output bindong
```

## CLI

| Command                                                                                                                                     | Brief Explanation                        | Example                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| [az functionapp plan create](https://learn.microsoft.com/en-us/cli/azure/functionapp/plan?view=azure-cli-latest#az-functionapp-plan-create) | Create an Azure Function App plan.       | `az functionapp plan create --name MyPlan --resource-group $resourceGroup --sku Y1 --is-linux`                     |
| [az functionapp plan update](https://learn.microsoft.com/en-us/cli/azure/functionapp/plan?view=azure-cli-latest#az-functionapp-plan-update) | Update a Function App plan.              | `az functionapp plan update --name MyPlan --sku Y2`                                                                |
| [az functionapp plan list](https://learn.microsoft.com/en-us/cli/azure/functionapp/plan?view=azure-cli-latest#az-functionapp-plan-list)     | List function app plans.                 | `az functionapp plan list --resource-group $resourceGroup`                                                         |
| [az appservice plan list](https://learn.microsoft.com/en-us/cli/azure/appservice/plan?view=azure-cli-latest#az-appservice-plan-list)        | List app service plans.                  | `az appservice plan list --resource-group $resourceGroup`                                                          |
| [az functionapp show](https://learn.microsoft.com/en-us/cli/azure/functionapp?view=azure-cli-latest#az-functionapp-show)                    | Show the details of a function app.      | `az functionapp show --name MyFunctionApp --resource-group $resourceGroup`                                         |
| [az functionapp create](https://learn.microsoft.com/en-us/cli/azure/functionapp?view=azure-cli-latest#az-functionapp-create)                | Create a function app.                   | `az functionapp create --name MyFunctionApp --storage-account mystorageaccount --consumption-plan-location eastus` |
| [az functionapp config appsettings](https://learn.microsoft.com/en-us/cli/azure/functionapp/config/appsettings?view=azure-cli-latest)       | Manage function app settings.            | `az functionapp config appsettings set --name MyFunctionApp --settings KEY=VALUE`                                  |
| [az functionapp cors add](https://learn.microsoft.com/en-us/cli/azure/functionapp/cors?view=azure-cli-latest#az-functionapp-cors-add)       | Add allowed origins to function app.     | `az functionapp cors add --name MyFunctionApp --allowed-origins 'https://example.com'`                             |
| [az functionapp cors show](https://learn.microsoft.com/en-us/cli/azure/functionapp/cors?view=azure-cli-latest#az-functionapp-cors-show)     | Show details of CORS for a function app. | `az functionapp cors show --name MyFunctionApp`                                                                    |
| [az resource update](https://learn.microsoft.com/en-us/cli/azure/resource?view=azure-cli-latest#az-resource-update)                         | Update a resource.                       | `az resource update --ids RESOURCE_ID --set properties.key=value`                                                  |
| [az rest](https://learn.microsoft.com/en-us/cli/azure/reference-index?view=azure-cli-latest#az-rest)                                        | Invoke a custom request.                 | `az rest --uri /subscriptions/{subscriptionId}/resourcegroups?api-version=2019-10-01`                              |

# [Microsoft Graph](https://learn.microsoft.com/en-us/graph/)

Provides a unified programmability model that you can use to access data in Microsoft 365, Windows 10, and Enterprise Mobility + Security.

- Endpoint: `https://graph.microsoft.com`. Can manage user and device identity, access, compliance, security, and help protect organizations from data leakage or loss.

- [Microsoft Graph connectors](https://learn.microsoft.com/en-us/microsoftsearch/connectors-overview): delivering **data external to the Microsoft cloud into Microsoft Graph services and applications** (Box, Google Drive, Jira, and Salesforce).
- [Microsoft Graph Data Connect](https://learn.microsoft.com/en-us/graph/data-connect-concept-overview): delivering **Microsoft Graph data to popular Azure data stores**.

## Resources

Resource specify the entity or complex type you're interacting with, like `me`, `user`, `group`, `drive`, or `site`. Top-level resources may have relationships, allowing access to other resources, like `me/messages` or `me/drive`. Interactions with resources are done through methods, e.g., `me/sendMail` for sending an email. Permissions needed for each resource may vary, with higher permissions often required for creation or updates compared to reading. [More on permissions](https://learn.microsoft.com/en-us/graph/permissions-reference).

## [Headers](https://learn.microsoft.com/en-us/graph/use-the-api#headers)

Include standard and custom HTTP types. Certain APIs might need extra headers in requests. Mandatory headers like the `request-id` are always returned by Microsoft Graph, and certain headers, like `Retry-After` during throttling or `Location` for long-running operations, are specific to certain APIs or features.

**Evolvable enumerations**: By default, a GET operation returns only known (existing) members for properties. Adding members to existing enumerations can break applications already using these enums. You can opt in to receive all members by using an HTTP `Prefer` request header.

## Query Microsoft Graph by using REST

[Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer)

### [Metadata](https://learn.microsoft.com/en-us/graph/traverse-the-graph?tabs=http#microsoft-graph-api-metadata)

`https://graph.microsoft.com/v1.0/$metadata`

Metadata in Microsoft Graph provides insight into its data model, including entity types, complex types, and enumerations present in request and response data. It defines types, methods, and enumerations in OData namespaces, with most of the API in the namespace `microsoft.graph` and some in subnamespaces like `microsoft.graph.callRecords`. It helps understand relationships between entities and enables URL navigation between them.

### Using REST

```http
{HTTP method} https://graph.microsoft.com/{version}/{resource}?{query-parameters}
```

- HTTP _Authorization_ request header, as a _Bearer_ token
- [Pagination](https://learn.microsoft.com/en-us/graph/paging) is handled via `@odata.nextLink`.

| Method | Description                                  |
| ------ | -------------------------------------------- |
| GET    | Read data from a resource.                   |
| POST   | Create a new resource, or perform an action. |
| PATCH  | Update a resource with new values.           |
| PUT    | Replace a resource with a new one.           |
| DELETE | Remove a resource.                           |

- For the CRUD methods `GET` and `DELETE`, no request body is required.
- The `POST`, `PATCH`, and `PUT` methods require a request body specified in JSON format.

#### Examples

Sure, here's the table based on the provided information:

| Operation                                | URL                                                                                                                      |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| GET my profile                           | `https://graph.microsoft.com/v1.0/me`                                                                                    |
| GET my files                             | `https://graph.microsoft.com/v1.0/me/drive/root/children`                                                                |
| GET my photo                             | `https://graph.microsoft.com/v1.0/me/photo/$value`                                                                       |
| GET my photo metadata                    | `https://graph.microsoft.com/v1.0/me/photo/`                                                                             |
| GET my mail                              | `https://graph.microsoft.com/v1.0/me/messages`                                                                           |
| GET my high importance email             | `https://graph.microsoft.com/v1.0/me/messages?$filter=importance eq 'high'`                                              |
| GET my calendar events                   | `https://graph.microsoft.com/v1.0/me/events`                                                                             |
| GET my manager                           | `https://graph.microsoft.com/v1.0/me/manager`                                                                            |
| GET last user to modify file foo.txt     | `https://graph.microsoft.com/v1.0/me/drive/root/children/foo.txt/lastModifiedByUser`                                     |
| GET Microsoft 365 groups I'm a member of | `https://graph.microsoft.com/v1.0/me/memberOf/$/microsoft.graph.group?$filter=groupTypes/any(a:a eq 'unified')`          |
| GET users in my organization             | `https://graph.microsoft.com/v1.0/users`                                                                                 |
| GET groups in my organization            | `https://graph.microsoft.com/v1.0/groups`                                                                                |
| GET people related to me                 | `https://graph.microsoft.com/v1.0/me/people`                                                                             |
| GET items trending around me             | `https://graph.microsoft.com/beta/me/insights/trending`                                                                  |
| GET my recent activities                 | `https://graph.microsoft.com/v1.0//me/activities/recent`                                                                 |
| PATCH (update) a recent activity of mine | `https://graph.microsoft.com/v1.0//me/activities/{activityId}`                                                           |
| GET my notes                             | `https://graph.microsoft.com/v1.0/me/onenote/notebooks`                                                                  |
| Select specific fields                   | `https://graph.microsoft.com/v1.0/groups?$filter=adatumisv_courses/id eq '123'&$select=id,displayName,adatumisv_courses` |
| Alerts, filter by Category, top 5        | `https://graph.microsoft.com/v1.0/security/alerts?$filter=Category eq 'ransomware'&$top=5`                               |
| To get only the display name of users with last name 'Bob' | `https://graph.microsoft.com/v1.0/users?$filter=surname eq 'Bob'&$select=displayName`|

#### Using MSAL

```cs
var authority = "https://login.microsoftonline.com/" + tenantId;
var scopes = new []{ "https://graph.microsoft.com/.default" };

var app = ConfidentialClientApplicationBuilder.Create(clientId)
    .WithAuthority(authority)
    .WithClientSecret(clientSecret)
    .Build();

var result = await app.AcquireTokenForClient(scopes).ExecuteAsync();

var httpClient = new HttpClient();
var request = new HttpRequestMessage(HttpMethod.Get, "https://graph.microsoft.com/v1.0/me");
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", result.AccessToken);

var response = await httpClient.SendAsync(request);
var content = await response.Content.ReadAsStringAsync();
```

### Using SDK

```cs
var scopes = new[] { "User.Read" };

// Multi-tenant apps can use "common",
// single-tenant apps must use the tenant ID from the Azure portal
var tenantId = "common";

// Value from app registration
var clientId = "YOUR_CLIENT_ID";

// using Azure.Identity;
var options = new TokenCredentialOptions
{
    AuthorityHost = AzureAuthorityHosts.AzurePublicCloud
};

// Using device code: https://learn.microsoft.com/dotnet/api/azure.identity.devicecodecredential
var deviceOptions = new DeviceCodeCredentialOptions
{
    AuthorityHost = AzureAuthorityHosts.AzurePublicCloud,
    ClientId = clientId,
    TenantId = tenantId,
    // Callback function that receives the user prompt
    // Prompt contains the generated device code that user must
    // enter during the auth process in the browser
    DeviceCodeCallback = (code, cancellation) =>
    {
        Console.WriteLine(code.Message);
        return Task.FromResult(0);
    },
};
var credential = new DeviceCodeCredential(deviceOptions);
// var credential = new DeviceCodeCredential(callback, tenantId, clientId, options);

// Using a client certificate: https://learn.microsoft.com/dotnet/api/azure.identity.clientcertificatecredential
// var clientCertificate = new X509Certificate2("MyCertificate.pfx");
// var credential = new ClientCertificateCredential(tenantId, clientId, clientCertificate, options);

// Using a client secret: https://learn.microsoft.com/dotnet/api/azure.identity.clientsecretcredential
// var credential = new ClientSecretCredential(tenantId, clientId, clientSecret, options);

// On-behalf-of provider
// var oboToken = "JWT_TOKEN_TO_EXCHANGE";
// var onBehalfOfCredential = new OnBehalfOfCredential(tenantId, clientId, clientSecret, oboToken, options);

var graphClient = new GraphServiceClient(credential, scopes);

var user = await graphClient.Me.GetAsync();

var messages = await graphClient.Me.Messages
.GetAsync(requestConfig =>
{
    requestConfig.QueryParameters.Select =
        new string[] { "subject", "sender" };
    requestConfig.QueryParameters.Filter =
        "subject eq 'Hello world'";

    requestConfig.Headers.Add(
        "Prefer", @"outlook.timezone=""Pacific Standard Time""");
});

var message = await graphClient.Me.Messages[messageId].GetAsync();

var newCalendar = await graphClient.Me.Calendars
    .PostAsync(new Calendar { Name = "Volunteer" }); // new

await graphClient.Teams["teamId"]
    .PatchAsync(new Team { }); // update

await graphClient.Me.Messages[messageId]
    .DeleteAsync();
```

## Token acquisition flow

- **Acquire an Authorization Code**: `GET https://login.microsoftonline.com/{tenant}/oauth2/authorize`
- **Acquire an Access Token**: `POST https://login.microsoftonline.com/customer.com/oauth2/token`
- **Call Microsoft Graph**:

  ```http
  GET https://graph.microsoft.com/beta/users
  Authorization: Bearer <token>
  ```

## [Permissions](https://learn.microsoft.com/en-us/graph/permissions-reference)

`<Resource>.<Permission>` or `<Resource>.<Permission>.<Optional-Constrain>`

Example for `Users`:

- Current user: `User.Read`, `User.ReadWrite`
- All users (require admin consent): `User.Read.All`, `User.ReadWrite.All`, `User.ReadBasic.All` (no admin consent)

The optional `All` sonstrain grants access to all users.

Example for `Calendars`:

- Current user's calendars: `Calendars.Read`, `Calendars.ReadWrite`
- Calendars shared with current user: `Calendars.Read.Shared`, `Calendars.ReadWrite.Shared`

The optional `Shared` constrain grants access to calendars user has access to, there is no `All` contrain.

# [Azure Key Vault](https://docs.microsoft.com/en-us/azure/key-vault/)

Endpoint: `https://vault.azure.net`

- Azure Key Vault securely stores secrets, keys, and certificates.
- Available in two tiers: **Standard** for software encryption, **Premium** for HSM-protected keys.
- Centralizes security data to minimize leaks and avoid storing sensitive info in code.
- Offers monitoring to track key and secret access.
- Scales automatically and ensures high availability through data replication.
- Automates certificate tasks.
- Integrates with other Azure services for disk and database encryption.

`az keyvault create --name <YourKeyVaultName> --resource-group $resourceGroup --location <YourLocation>`

Set secret: `az keyvault secret set --vault-name $myKeyVault --name "ExamplePassword" --value "hVFkk965BuUv"`

Retrieve secret (in _JSON_ format): `az keyvault secret show --name "ExamplePassword" --vault-name $myKeyVault` (`value` property contains the secret value)

Get secret version: `GET {vaultBaseUrl}/secrets/{secret-name}/{secret-version}?api-version=2025-07-01`

## [Security](https://learn.microsoft.com/en-us/azure/key-vault/general/security-features)

## Key operations

| Operation                                                                                                                                                                | Command                                                                                                                | Description                                                                           |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Manual Key Rotation                                                                                                                                                      | `az keyvault key rotate --name <YourKeyName> --vault-name <YourVaultName>`                                             | Manually rotate a key to create a new version                                         |
| [Automated Key Rotation](https://learn.microsoft.com/en-us/cli/azure/keyvault/key/rotation-policy?view=azure-cli-latest#az-keyvault-key-rotation-policy-update-examples) | `az keyvault key rotation-policy update --name <YourKeyName> --vault-name <YourVaultName> --value path/to/policy.json` | Configure automated rotation policy (e.g., time-based).                               |
| List All Keys                                                                                                                                                            | `az keyvault key list --vault-name <YourVaultName>`                                                                    | List all keys in the vault                                                            |
| List Enabled Keys Only                                                                                                                                                   | `az keyvault key list --vault-name <YourVaultName> --query "[?attributes.enabled].name" -o tsv`                        | Filter and display only enabled key names using JMESPath query                        |
| Backup Key                                                                                                                                                               | `az keyvault key backup --name <YourKeyName> --vault-name <YourVaultName> --file ./old-key-backup.blob`                | Create a backup of a key to a file                                                    |
| Delete Key (Soft Delete)                                                                                                                                                 | `az keyvault key delete --name <YourKeyName> --vault-name <YourVaultName>`                                             | Move key to soft-deleted state (if enabled) or remove it                              |
| Purge Key (Permanent)                                                                                                                                                    | `az keyvault key purge --name <YourKeyName> --vault-name <YourVaultName>`                                              | Permanently remove a soft-deleted key. Only applicable for soft-delete enabled vaults |

**📝 NOTE:** Latter sends an event to Azure Event Grid, which you can subscribe to for sending email alerts.

### Access Model

- **Management plane**: for managing the Key Vault itself
- **Data plane**: for working with the data stored in the Key Vault

Both planes use Azure Microsoft Entra ID for authentication, and [RBAC](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide?tabs=azure-cli) for authorization (access control). _Data plane_ also uses a [access policies](https://learn.microsoft.com/en-us/azure/key-vault/general/assign-access-policy?tabs=azure-portal) (legacy) for authorization. Minimum standard role for granting management and data (policies) access: `Contributor`.

```sh
az keyvault set-policy --name myKeyVault --object-id <object-id> --secret-permissions <secret-permissions> --key-permissions <key-permissions> --certificate-permissions <certificate-permissions>
```

### Authentication

Key Vault is associated with the Entra ID tenant of the subscription and all callers must register in this tenant and authenticate to access the key vault.

For applications, there are two ways to obtain a service principal - first is recommended:

- Enable a system-assigned **managed identity** for the application. With managed identity, Azure internally manages the application's service principal and automatically authenticates the application with other Azure services. Managed identity is available for applications deployed to various services. It also handles credential rotation.
- If you can't use managed identity, you instead register the application with your Entra ID tenant. Registration also creates a second object (`Application Object`, as global definition of your app) that identifies the app across all tenants.

#### Usage

```cs
string keyVaultName = Environment.GetEnvironmentVariable("KEY_VAULT_NAME");
var kvUri = "https://" + keyVaultName + ".vault.azure.net";

var client = new SecretClient(
    new Uri(kvUri),      // Where: Your Key Vault URL (e.g., https://my-vault.vault.azure.net/)
    new DefaultAzureCredential()    // How: Automatically handles authentication
);

...

// Read a secret
KeyVaultSecret secret = await client.GetSecretAsync("ConnectionString");
string value = secret.Value;

// Create/Update a secret. If name exists, will create new version of that secret.
await client.SetSecretAsync("ApiKey", "abc123");

// The secret is soft-deleted by default, not permanently erased.
DeleteSecretOperation operation = await client.StartDeleteSecretAsync("OldPassword");
// You need to wait for completion if you want to purge or recover the key.
await operation.WaitForCompletionAsync();

await client.PurgeDeletedSecretAsync(secretName);
```

Authentication using REST:

```http
PUT https://<your-key-vault-name>.vault.azure.net/keys/<your-key-name>?api-version=7.2 HTTP/1.1
Authorization: Bearer <access_token> # token obtained from Microsoft Entra ID
```

If Authorization token is missing or rejected:

```http
401 Not Authorized
WWW-Authenticate: Bearer authorization="…", resource="…"
```

The `WWW-Authenticate` header parameters are:

- `authorization`: OAuth2 authorization service address.
- `resource`: Resource name (`https://vault.azure.net`) for the authorization request.

### Authentication flow for "Get Secret" API

<img src="https://learn.microsoft.com/en-us/azure/key-vault/media/authentication/authentication-flow.png" width="600">

<br>

### Restricting access

As best practice:
For single-resource, use `System-Assigned Managed Identities` to avoid hardcoding credentials. Using environment variables can expose them in your code.

If multiple resources need the same identity, use `User-Assigned Managed Identities`.

Limit vault access to specific IPs via using VNet integration and Private Endpoints to ensure the Key Vault isn't exposed to the public internet.

### [Data Transit Encryption](https://learn.microsoft.com/en-us/azure/key-vault/general/basic-concepts#encryption-of-data-in-transit)

Secure communication through **HTTPS** and **TLS** (min 1.2).

**Perfect Forward Secrecy** (PFS - protects connections between customer and cloud services by unique keys) and RSA-based 2,048-bit encryption key lengths secure connections.

## [Certificates](https://learn.microsoft.com/en-us/azure/key-vault/certificates/quick-create-net)

Create an access policy for your key vault that grants certificate permissions to your user account:

```sh
az keyvault set-policy --name <your-key-vault-name> --upn user@domain.com --certificate-permissions delete get list create purge
```

**📝 NOTE:** `--certificate-permissions` specifies the exact permissions that this user will have, but **only for certificates**. It does not grant any permissions for keys or secrets. UPN stands for User Principle Name.

Store and retieve certificates:

```cs
var client = new CertificateClient(new Uri($"https://{keyVaultName}.vault.azure.net"), new DefaultAzureCredential());

// Create certificate
var operation = await client.StartCreateCertificateAsync(certificateName, CertificatePolicy.Default);

// Wait for the certificate creation to fully complete on the server.
// This is necessary because the next step needs the certificate to exist.
await operation.WaitForCompletionAsync();

// Retrieve
var certificate = await client.GetCertificateAsync(certificateName);
```

| Method           | `GetCertificateAsync()`   | `DownloadCertificateAsync()`                                                                                                                   |
| ---------------- | ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Purpose          | Get metadata + public key | Get full bundle + private key                                                                                                                  |
| Returns          | `KeyVaultCertificate`     | [`X509Certificate2`](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.x509certificate2?view=net-9.0) |
| Has Private Key? | No                        | Yes (if permissions allow)                                                                                                                     |
| Permissions      | `certificates/get`        | `certificates/get` and `secrets/get`                                                                                                           |

## Best Practices

- Use a separate vault for each application and environment (production, test, staging).
- Restrict vault access to authorized applications and users. (`az keyvault set-policy --name <YourKeyVaultName> --object-id <PrincipalObjectId> --secret-permissions get list`)
- Regularly backup your vault. (`az keyvault key backup --vault-name <YourKeyVaultName> --name <KeyName> --file <BackupFileName>`)
- Enable logging and alerts.
- Enable **soft-delete** and **purge protection** to keep secrets for 7-90 days and prevent forced deletion.

  ```sh
    az keyvault update --subscription {SUBSCRIPTION ID} -g {RESOURCE GROUP} -n {VAULT NAME} --enable-soft-delete true
    az keyvault update --subscription {SUBSCRIPTION ID} -g {RESOURCE GROUP} -n {VAULT NAME} --enable-purge-protection true
  ```

## Charging

- Charges apply for HSM-keys (charge per key version per month) in the last 30 days of use. After that, since the object is in deleted state no operations can be performed against it, so no charge will apply.
- Operations are disabled on deleted objects, and no charges apply. (NOTE: _soft-delete_ increases security, but also _increases storage cost_!)

**📝 NOTE:** Soft-delete can't be turned off for [Managed HSM, NOT standard Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/soft-delete-overview) resources. Soft-deleted Managed HSM resources will continue to be billed at their full hourly rate until they're purged.

## [Disaster and recovery](https://learn.microsoft.com/en-us/azure/key-vault/general/disaster-recovery-guidance)

Below applies for both Standard and Premium tiers.

| Region Type          | Within-Region Replication                          | Cross-Region Replication         | Behavior During Regional Failure                                          | Examples                                      |
| -------------------- | -------------------------------------------------- | -------------------------------- | ------------------------------------------------------------------------- | --------------------------------------------- |
| **Paired Regions**   | Yes (Zone-Redundant Storage in AZ-enabled regions) | Yes (automatic to paired region) | Key Vault becomes **read-only** during failover to prevent data conflicts | East US ↔ West US, North Europe ↔ West Europe |
| **Unpaired Regions** | Yes (Zone-Redundant Storage in AZ-enabled regions) | No                               | Service becomes **unavailable** until region recovers                     | Brazil South, West US 3, Brazil Southeast     |

## Disk Encryption ([Windows](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/disk-encryption-key-vault?tabs=azure-portal), [Linux](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/disk-encryption-key-vault?tabs=azure-portal))

```sh
az login

# A resource group is a logical container into which Azure resources are deployed and managed.
az group create --name $resourceGroup --location eastus

# Create a key vault in the same region and tenant as the VMs to be encrypted.
# The key vault will be used to control and manage disk encryption keys and secrets.
az keyvault create --name "<keyvault-id>" --resource-group $resourceGroup --location "eastus"

# Update the key vault's advanced access policies
az keyvault update --name "<keyvault-id>" --resource-group $resourceGroup --enabled-for-disk-encryption "true"
# Optional: Enables the 'Microsoft.Compute' resource provider to retrieve secrets from this key vault when this key vault is referenced in resource creation, for example when creating a virtual machine.
az keyvault update --name "<keyvault-id>" --resource-group $resourceGroup --enabled-for-deployment "true"
# Optional: Allow Resource Manager to retrieve secrets from the vault.
az keyvault update --name "<keyvault-id>" --resource-group $resourceGroup --enabled-for-template-deployment "true"

# Optional: When a key encryption key (KEK) is specified, Azure Disk Encryption uses that key to wrap the encryption secrets before writing to Key Vault.
az keyvault key create --name "myKEK" --vault-name "<keyvault-id>" --kty RSA --size 4096

# Enable disk encryption:
## Option 1: use KEK by name
az vm encryption enable -g $resourceGroup --name "myVM" --disk-encryption-keyvault "<keyvault-id-or-name>" --key-encryption-key "myKEK"
## Option 2: use KEK by url
## Obtain <kek-url> as pre-step
az keyvault key show --name <keyvault-id> --vault-name <vault-name> --query key.kid -o tsv
az vm encryption enable -g $resourceGroup --name "MyVM" --disk-encryption-keyvault "<keyvault-id-or-name>" --key-encryption-key "https://myvault.vault.azure.net/keys/mykey/<key-version>" --volume-type All
```

**📝 NOTE:** If you have previously used Azure Disk Encryption with Microsoft Entra ID to encrypt a VM, you must continue using this option to encrypt your VM.

## [Logging](https://learn.microsoft.com/en-us/azure/key-vault/key-vault-insights-overview)

### [Monitoring Key Vault with Azure Event Grid](https://learn.microsoft.com/en-us/azure/key-vault/general/event-grid-overview)

Key Vault events can be identified by property name which starts with [`Microsoft.KeyVault`](https://learn.microsoft.com/en-us/azure/event-grid/event-schema-key-vault?tabs=cloud-event-schema).

`Portal > All Services > Key Vaults > key vault > Events > Event Grid Subscriptions > + Event Subscription` and fill in the details including name, event types, and endpoint.

**Event Flow**: Key Vault → Event Grid → Below Function → Log Analytics/Alerts

```cs
// This file contains the Azure Function that listens for Key Vault events.
public static class KeyVaultMonitoring
{
    /// <summary>
    /// This function is triggered by an HTTP request.
    /// It's designed to be used as a Webhook endpoint for an Azure Event Grid subscription.
    /// Event Grid will POST a JSON array of events to this endpoint.
    /// </summary>
    [FunctionName("KeyVaultMonitoring")]
    public static async Task<IActionResult> Run(
        // [HttpTrigger] defines this function as an HTTP endpoint.
        // - AuthorizationLevel.Function: Requires a Function API key for security.
        // ILogger for writing logs to Azure Monitor.
        // Event Grid sends POST for both:
        // 1. Validation event (once, during subscription setup)
        // 2. Actual Key Vault events (ongoing)
        // Common practice for webhook endpoints to accept GET for e.g.: quick browser-based testing (just paste URL in browser)
        [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequest req, ILogger log)
    {
        log.LogInformation("KeyVaultMonitoring function processed a request.");

        // --- Read and Parse the Incoming Event ---

        // Read the raw JSON payload from the HTTP request body.
        var requestBody = await new StreamReader(req.Body).ReadToEndAsync();

        // Parse the Event Grid event from the JSON payload
        // BinaryData provides efficient handling of the JSON string
        var eventGridEvent = EventGridEvent.Parse(new BinaryData(requestBody));

        // --- Process the Event ---

        // Use a switch statement on the EventType property to determine
        // what kind of Key Vault event just occurred.
        switch(eventGridEvent.EventType)
        {
            // A new version of a certificate was created.
            case SystemEventNames.KeyVaultCertificateNewVersionCreated:
            // A new version of a secret was created.
            case SystemEventNames.KeyVaultSecretNewVersionCreated:
                log.LogInformation($"A new secret or certificate version was created.");
                // eventGridEvent.Data contains the specific payload for this event,
                // which includes the VaultName, ObjectName, etc.
                log.LogInformation($"Event Data: {eventGridEvent.Data}");

                // TODO: Add your logic here.
                // e.g., send an email, update a database, post to Teams.
                break;

            // A new version of a key was created.
            case SystemEventNames.KeyVaultKeyNewVersionCreated:
                log.LogInformation($"A new key version was created.");
                log.LogInformation($"Event Data: {eventGridEvent.Data}");

                // TODO: Add your logic here.
                break;

            // This is not an error - it's normal to receive events you don't process
            // Event Grid may send validation events or other system events
            default:
                log.LogInformation($"Event of type '{eventGridEvent.EventType}' received but not processed.");
                log.LogInformation($"Event data: {eventGridEvent.Data}");
                break;
        }

        // Return HTTP 200 OK to acknowledge successful receipt to Event Grid
        // Returning errors causes Event Grid to retry, which can cause duplicate processing
        return new OkResult();
    }
}
```

## Working with KeyVault

```cs
var keyClient = new KeyClient(vaultUri: new Uri(vaultUrl), credential: new DefaultAzureCredential());
// Creating a new key
KeyVaultKey key = await keyClient.GetKeyAsync("YourKeyName");
// Encrypting and decrypting data using the key via CryptographyClient
CryptographyClient cryptoClient = keyClient.GetCryptographyClient(key.Name, key.Properties.Version);
EncryptResult encryptResult = cryptoClient.Encrypt(EncryptionAlgorithm.RsaOaep, Encoding.UTF8.GetBytes(plaintext));
DecryptResult decryptResult = cryptoClient.Decrypt(EncryptionAlgorithm.RsaOaep, encryptResult.Ciphertext);
```

## CLI

- [az keyvault set-policy](https://learn.microsoft.com/en-us/cli/azure/keyvault?view=azure-cli-latest#az-keyvault-set-policy)
- [az keyvault secret](https://learn.microsoft.com/en-us/cli/azure/keyvault/secret?view=azure-cli-latest)
- [az keyvault key](https://learn.microsoft.com/en-us/cli/azure/keyvault/key?view=azure-cli-latest)

# [Azure Managed Identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/)

Enables [`Azure-hosted apps/VMs`](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/managed-identities-status#:~:text=This%20page%20provides%20links%20to%20services%27%20content%20that%20can%20use%20managed%20identities%20to%20access%20other%20Azure%20resources) to access other services without handling credentials. These identities are Azure-exclusive and can't be used with other cloud providers.

For full list of supported services and details, [click](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/managed-identities-status).

**📝 NOTE:** If you change a managed identity's group/role membership to add or remove permissions, you must wait up to 24 hours.

[S.A.M.I vs U.A.M.I](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview#differences-between-system-assigned-and-user-assigned-managed-identities)
| Property                       | System-assigned managed identity                                                                                                                                       | User-assigned managed identity                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Creation                       | Created as part of an Azure resource (e.g. `Azure Virtual Machines` or `Azure App Service`).                                                                           | Created as a stand-alone Azure resource.                                                                                                                                                                                                                                                                                                                                                                          |
| Life cycle                     | Shared life cycle with the Azure resource.<br>Deleted when the parent resource is deleted.                                            | Independent life cycle.<br>Must be explicitly deleted.                                                                                                                                                                                                                                                                                                                                                            |
| Sharing across Azure resources | Can’t be shared.<br>It can only be associated with a single Azure resource.                                                                                            | Can be shared.<br>The same user-assigned managed identity can be associated (shared) with more than one Azure resource.                                                                                                                                                                                                                                                                                           |
| Common use cases               | Workloads contained within a single Azure resource.<br>Workloads needing independent identities.<br>For example, an application that runs on a single virtual machine. | Workloads that run on multiple resources and can share a single identity.<br>[Workloads needing pre-authorization](https://gemini.google.com/share/3a088a2cbf6f#:~:text=For%20your%20scenario,you%20deploy%20it.) to a secure resource, as part of a provisioning flow.<br>Workloads where resources are recycled frequently, but permissions should stay consistent.<br>For example, a workload where multiple virtual machines need to access the same resource. |

**📝 NOTE:** If you have multi-tenant setup, use `Application Service Principal`!

## [Role-based access control (Azure RBAC)](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal#assign-a-user-as-an-administrator-of-a-subscription)

Manages access to Azure resources. To assign Azure roles, you must have `Microsoft.Authorization/roleAssignments/write` permissions, belong to such as `User Access Administrator`, `Role Based Access Control Administrator` or `Owner` roles.

```bash
# Check specific permissions
az role definition list --name "<role-name>" --query "[].permissions[].actions"
```

Similarly, you can verify read access to all resources by permission: `*/read`.

[**Roles**](https://docs.microsoft.com/en-us/azure/role-based-access-control/role-definitions): Defines what actions you can perform.
  - **Owner**: Full access, including role assignment.
  - **Contributor**: Full access, no role assignment.
  - **Reader**: View-only.
  - **User Access Administrator**: Manages user access to resources.
  <br>

**📝 NOTE:** [Deny assignments](https://docs.microsoft.com/en-us/azure/role-based-access-control/deny-assignments) **override role assignments** to block specific actions.

[**Scope Levels**](https://docs.microsoft.com/en-us/azure/role-based-access-control/scope-overview): Defines where actions apply.
  - **Management Group** (largest scope)
    - **Subscription**
      - **Resource Group**
        - **Resource** (smallest scope)
  <br>

**📝 NOTE:** When assigning access, follow the rule of least privilege. A role assigned at a higher level automatically applies to all lower levels in scope.

## Managing Identities

1. **System-assigned Identity**

   ```sh
   # Creating a resource (like a VM or any other service that supports it) with a system-assigned identity
   az <service> create --resource-group $resourceGroup --name myResource --assign-identity [system]

   # Assigning a system-assigned identity to an existing resource
   az <service> identity assign --resource-group $resourceGroup --name myResource --identities [system]
   ```

1. **User-assigned Identity**

   ```sh
   # First, create the identity
   az identity create --resource-group $resourceGroup --name identityName

   # Creating a resource (like a VM or any other service that supports it) with a user-assigned identity
   az <service> create --assign-identity $identityName --resource-group $resourceGroup --name $resourceName
   #az <service> create --assign-identity '/subscriptions/<SubId>/resourcegroups/$resourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myIdentity' --resource-group $resourceGroup --name $resourceName

   # Assigning a user-assigned identity to an existing resource
   az <service> identity assign --identities $identityName --resource-group $resourceGroup --name $resourceName
   # az <service> identity assign --identities '/subscriptions/<SubId>/resourcegroups/$resourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myIdentity' --resource-group $resourceGroup --name $resourceName
   ```

Both system-assigned and user-assigned managed identities can be assigned specific Azure roles, allowing them to perform certain actions on specific Azure resources. These roles are part of Azure's Role-Based Access Control ([RBAC](https://docs.microsoft.com/en-us/azure/role-based-access-control/overview)) system, which provides fine-grained access management to Azure resources.

```sh
az role assignment create --assignee <PrincipalId> --role <RoleName> --scope <Scope>
```

### RBAC Scope Requirements for `az identity` Commands

| Command | Minimum Scope Level | Required Role(s) | Notes |
|---------|-------------------|------------------|-------|
| `az identity create` | Resource Group | Managed Identity Contributor, Contributor, or Owner | Can create identities in the RG where you have permissions |
| `az identity delete` | Resource Group | Managed Identity Contributor, Contributor, or Owner | Can delete identities from the RG where you have permissions |
| `az identity show` | Resource Group | Managed Identity Contributor, Managed Identity Operator, Reader, or higher | Can view identity details |
| `az identity list` | Resource Group / Subscription | Managed Identity Contributor, Managed Identity Operator, Reader, or higher | Lists identities within the scope |
| `az identity assign` (to resource) | Resource + Resource Group | Managed Identity Operator (on identity) + Contributor/Owner (on target resource) | Requires permissions on both identity and target resource |

- For more, [click](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/managed-identities-faq#which-azure-role-based-access-control-rbac-permissions-are-required-to-use-a-managed-identity-on-a-resource).

**📝 NOTE:** An Azure resource can have both a system-assigned and one or more user-assigned managed identities at a time.

## Acquiring an Access Token with Azure Managed Identities

**DefaultAzureCredential**: This class attempts multiple methods of authentication based on the available environment or sign-in details, stopping once it's successful. It checks the following sources in order:

1. Environment variables ([`EnvironmentCredential`](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.environmentcredential?view=azure-dotnet)): Checks for three possible combinations of environment variables.

| Authentication Method | Required Environment Variables | Optional Environment Variables |
|----------------------|-------------------------------|-------------------------------|
| Service Principal with Secret | `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET` | - |
| Service Principal with Certificate | `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_CERTIFICATE_PATH` | `AZURE_CLIENT_CERTIFICATE_PASSWORD`, `AZURE_CLIENT_SEND_CERTIFICATE_CHAIN` |
| Username/Password | `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_USERNAME`, `AZURE_PASSWORD` | - |

**📝 NOTE:** This credential ultimately uses a `ClientSecretCredential` or `ClientCertificateCredential`.

2. [WorkloadIdentityCredential](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.workloadidentitycredential?view=azure-dotnet) supports Microsoft Entra Workload ID authentication on **Kubernetes** and other hosts supporting workload identity.

3. Managed Identity if the application is deployed on an Azure resource. If ManagedIdentityClientId (last line code snippet), it will check for that specific user-assigned identity.

   ```cs
   new ManagedIdentityCredential(); // system-assigned
   new ManagedIdentityCredential(clientId: userAssignedClientId); // user-assigned
   new DefaultAzureCredential(new DefaultAzureCredentialOptions { ManagedIdentityClientId = userAssignedClientId }); // user-assigned
   ```

3. `VisualStudioCredential` if the developer has authenticated through it. For Visual Studio 2017 or later.
4. `VisualStudioCodeCredential`.
5. Azure CLI (`AzureCliCredential`) if the developer has authenticated through `az login` command.
6. Azure PowerShell (`AzurePowerShellCredential`) if the developer has authenticated via the `Connect-AzAccount` command.
7. Azure Dev CLI (`AzureDeveloperCliCredential`).
8. `InteractiveBrowserCredential`, though this option is disabled by default.

   ```cs
   new DefaultAzureCredential(includeInteractiveCredentials: true);
   ```

**ChainedTokenCredential**: Enables users to combine multiple credential instances to define a customized chain of credentials.

```cs
// Authenticate using managed identity if it is available; otherwise use the Azure CLI to authenticate.
var credential = new ChainedTokenCredential(new ManagedIdentityCredential(), new AzureCliCredential());
```
### Quick Reference by Environment

| Environment | Primary Credentials Used |
|------------|-------------------------|
| **Production (Azure)** | ManagedIdentityCredential (preferred) |
| **Production (Non-Azure)** | EnvironmentCredential with Service Principal |
| **Kubernetes/AKS** | WorkloadIdentityCredential or ManagedIdentityCredential |
| **Local Development** | Visual Studio / VS Code / Azure CLI / PowerShell / Azure Developer CLI |
| **CI/CD Pipelines** | EnvironmentCredential with Service Principal |
| **Interactive Testing** | InteractiveBrowserCredential (must enable explicitly) |

---

## [Logging](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Identity_1.9.0/sdk/core/Azure.Core/samples/Diagnostics.md#logging)

```cs
// Ensure AzureEventSourceListener is in scope and active while using the client library for log collection.
// Create it as a top-level member of the class using the Event Hubs client.
using AzureEventSourceListener listener = AzureEventSourceListener.CreateConsoleLogger();

DefaultAzureCredentialOptions options = new DefaultAzureCredentialOptions
{
    Diagnostics =
    {
        LoggedHeaderNames = { "x-ms-request-id" },
        LoggedQueryParameters = { "api-version" },
        IsAccountIdentifierLoggingEnabled = true, // enable logging of sensitive information
        IsLoggingContentEnabled = true // log details about the account that was used to attempt authentication and authorization
    }
};
```
For more details, [click](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/eventhub/Azure.Messaging.EventHubs/samples/Sample10_AzureEventSourceListener.md#:~:text=In%20order%20for,inspected%20is%20used.).

Exceptions: Service client methods raise `AuthenticationFailedException` for token issues.

## [Token caching](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Identity_1.9.0/sdk/identity/Azure.Identity/samples/TokenCache.md)

### Default Cache (Shared)
All applications share the same cache.
```cs
var options = new DefaultAzureCredentialOptions
{
    TokenCachePersistenceOptions = new TokenCachePersistenceOptions()
};
var credential = new DefaultAzureCredential(options);
```

### Isolated Cache (Named)
Each application has its own cache.
```cs
var options = new DefaultAzureCredentialOptions
{
    TokenCachePersistenceOptions = new TokenCachePersistenceOptions
    {
        Name = "MyAppCache" // Separate cache per app
    }
};
var credential = new DefaultAzureCredential(options);
```

### Unencrypted Storage (Not Recommended)
Use only for testing/development.
```cs
var options = new DefaultAzureCredentialOptions
{
    TokenCachePersistenceOptions = new TokenCachePersistenceOptions
    {
        UnsafeAllowUnencryptedStorage = true // ⚠️ Security risk
    }
};
var credential = new DefaultAzureCredential(options);
```
## [Cache Support by Credential Type](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Identity_1.17.0/sdk/identity/Azure.Identity/samples/TokenCache.md#credentials-supporting-token-caching)

| Credential Type | Memory Cache | Disk Cache |
|-----------------|--------------|------------|
| EnvironmentCredential | ✅ Yes | ✅ Yes |
| ManagedIdentityCredential | ✅ Only | ❌ Not supported |
| DefaultAzureCredential | ✅ Only(**\***) | ❌ Not supported |
| AzureCliCredential | ❌ None | ❌ None |
| InteractiveBrowserCredential | ✅ Yes | ✅ Yes |
| DeviceCodeCredential | ✅ Yes | ✅ Yes |
| ClientSecretCredential | ✅ Yes | ✅ Yes |

(**\***): Supported if the target credential in the credential chain supports it.

## Portal: Using Managed Identities for App Service and Azure Functions

### Step 1: Enable Managed Identity

#### For App Service:
1. Go to Azure Portal → Navigate to your App Service
2. Click **"Identity"** under Settings
3. In **"System assigned"** tab → Switch to **"On"** → Click **"Save"**

#### For Azure Functions:
- **Automatically enabled** when creating new function apps
- For existing apps: Go to Portal → Function App → Settings → **"Identity"** blade → Enable

### Step 2: Assign Permissions to Managed Identity

Once enabled, grant the identity access to required Azure resources:

1. Navigate to the **target resource** (e.g., Storage Account, Key Vault)
2. Open **"Access control (IAM)"**
3. Click **"Add role assignment"**
4. Select appropriate role (e.g., "Storage Blob Data Contributor")
5. Search for your App Service/Function name in the dropdown
6. Click **"Save"**

**📝 NOTE:** System-assigned identities automatically get some permissions within the same subscription, but explicit role assignments are often still needed.

### Step 3: Configure Target Resource

**📝 NOTE:** For resources mentioned in this step, you might skip this step if you went for `default` settings on target resource which **disables Access Policy usage** anyways.  
Some resources require additional configuration beyond IAM roles:

- **Key Vault**: Must add an access policy containing the managed identity
- **Azure SQL Database**: Requires Azure AD authentication setup
- **General rule**: Target resource must accept Azure AD tokens

**Why?** Even with a valid token, calls will be denied without proper resource-level configuration.

### Step 4: Use Managed Identity in Code

Azure SDKs **automatically detect and use** the managed identity when running in Azure.

#### System-Assigned Identity Example:
```cs
using Azure.Storage.Blobs;
using Azure.Identity;

// DefaultAzureCredential automatically uses managed identity
var blobServiceClient = new BlobServiceClient(
    new Uri("https://your-storage-account.blob.core.windows.net/"), 
    new DefaultAzureCredential());

var containerClient = blobServiceClient.GetBlobContainerClient("your-container-name");
```

#### User-Assigned Identity Example:
```cs
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var userAssignedIdentityClientId = "your-user-assigned-identity-client-id";
var keyVaultUrl = "https://your-key-vault-name.vault.azure.net/";

var client = new SecretClient(
    new Uri(keyVaultUrl), 
    new DefaultAzureCredential(new DefaultAzureCredentialOptions
    {
        ManagedIdentityClientId = userAssignedIdentityClientId
    }));
```

### Step 5: Local Development (Optional)

`DefaultAzureCredential()` works locally too by trying multiple authentication methods:
- Environment variables
- Visual Studio authentication
- Azure CLI
- Other local credentials

You authenticate with your own user account locally, but the **same code works in Azure** with managed identity - no code changes needed!

### How Token Retrieval Works

Behind the scenes:
1. App Service/Functions provides an **internal REST endpoint** (IMDS, Instance Metadata Service)
2. Your app makes HTTP GET request to this endpoint
3. Endpoint returns Azure AD token representing the **application** (not a user)
4. Token is used to access Azure resources
5. **Azure Identity library abstracts this** - you just use `DefaultAzureCredential()`

---

# [Message Queues](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-azure-and-service-bus-queues-compared-contrasted)

## Use Cases

### Storage Queues

- **Large Message Storage**: Suitable for storing over 80 gigabytes of messages in a queue.
- **Progress Tracking**: Useful for tracking progress for processing a message, especially if a worker crashes.
- **Server-Side (Transaction) Logs**: If you require server-side logs of all transactions executed against your queues.

### Service Bus Queues

- **Real-Time Messaging**: When your solution needs to receive messages without polling the queue.
- **Transactional Behavior**: If your solution requires transactional behavior and atomicity when sending or receiving multiple messages.
- **Role-Based Access**: If you need a role-based access model to the queues with different rights/permissions for senders and receivers.
- **Processing Messages as Parallel Long-Running Streams**: When your application processes messages as parallel streams using the **session ID**, allowing consuming nodes to compete for streams and examine the application stream state with transactions.
- **Larger message size**: 64-256KB.

Storage Queues are generally more suitable for basic communication and large storage needs, while Service Bus Queues offer more advanced features and real-time capabilities.

## Comparison

Azure Service Bus supports "Receive and Delete" mode, where messages are immediately consumed and removed from the queue.  
Messages in Event Hubs are retained for a configured retention period, and consumers are responsible for tracking their position in the stream.  
In Queue Storage messages are hidden for a specified visibility timeout period, and if not deleted within that time (or not expired), they become visible again. The visibility timeout cannot be set to a value later than the expiry time. Once you set the visibility, you are essentially making it so that the queue can’t see or alter message(s), **hence it cannot be deleted during visibility period.**

### Foundational Capabilities

| Comparison Criteria      | Storage Queues                                                  | Service Bus Queues                                                                |
| ------------------------ | --------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| FIFO Ordering guarantee  | ❌                                                              | ✅ (by using message sessions)                                                    |
| At-Most-Once Delivery    | ❌                                                              | ✅                                                                                |
| Filtering                | ❌                                                              | ✅                                                                                |
| Atomic operation support | ❌                                                              | ✅                                                                                |
| Receive behavior         | Non-blocking (completes immediately if no new message is found) | Blocking with or without a timeout (offers long polling) / Non-blocking           |
| Push-style API           | ❌                                                              | ✅                                                                                |
| Receive mode             | Peek & Lease                                                    | Peek & Lock / Receive & Delete                                                    |
| Exclusive access mode    | Lease-based                                                     | Lock-based                                                                        |
| Lease/Lock duration      | 30 seconds (default) / 7 days (maximum)                         | 30 seconds (default)                                                              |
| Lease/Lock precision     | Message level                                                   | Queue level                                                                       |
| Batched receive          | ✅ (up to 32 messages)                                          | ✅ (implicitly enabling a pre-fetch property or explicitly by using transactions) |
| Batched send             | ❌                                                              | ✅ (by using transactions or client-side batching)                                |

### Advanced Capabilities

| Comparison Criteria         | Storage Queues       | Service Bus Queues               |
| --------------------------- | -------------------- | -------------------------------- |
| Automatic dead lettering    | ❌                   | ✅                               |
| Server-side transaction log | ✅                   | ❌                               |
| State (Session) management  | ❌                   | ✅                               |
| Message autoforwarding      | ❌                   | ✅                               |
| Purge queue function        | ✅                   | ❌                               |
| Message groups              | ❌                   | ✅ (by using messaging sessions) |
| Duplicate detection         | ❌ (_at least once_) | ✅ (_at most once_)              |

### Capacity and Quotas

| Comparison Criteria                  | Storage Queues                                        | Service Bus Queues                                                         |
| ------------------------------------ | ----------------------------------------------------- | -------------------------------------------------------------------------- |
| Maximum queue size                   | 500 TB (limited to a single storage account capacity) | 1 GB to 80 GB (defined upon creation of a queue and enabling partitioning) |
| Maximum message size                 | 64 KB (48 KB when using Base64 encoding)              | 256 KB or 100 MB (depends on the service tier)                             |
| Maximum number of queues             | Unlimited                                             | 10,000 (per service namespace)                                             |
| Maximum number of concurrent clients | Unlimited                                             | 5,000                                                                      |

### Management and Operations

| Comparison Criteria        | Storage Queues                                       | Service Bus Queues                                 |
| -------------------------- | ---------------------------------------------------- | -------------------------------------------------- |
| Management protocol        | REST over HTTP/HTTPS                                 | REST over HTTPS                                    |
| Runtime protocol           | REST over HTTP/HTTPS                                 | REST over HTTPS / AMQP 1.0 Standard (TCP with TLS) |
| Arbitrary metadata support | Yes                                                  | No                                                 |
| Queue naming rules         | Up to 63 characters long (Letters must be lowercase) | Up to 260 characters long (case-insensitive)       |

# [Azure Queue Storage](https://docs.microsoft.com/en-us/azure/storage/queues/)

Endpoint: `https://queue.core.windows.net`

- May contain millions of messages, up to the total capacity limit of a storage account.
- Commonly used to create a backlog of work to process asynchronously.
- Max size: 64KB
- TTL: 7 days (⏺️), -1 to never expire.
- Applications can scale indefinitely to meet demand.
- Lease-based: When a message is retrieved, it becomes invisible for a set period (default: 30 seconds). Prevents other consumers from processing the same message. If the message isn’t deleted within that time, it becomes visible again for reprocessing.
- Lease Renewal: If processing take longer than original lease time (and not encountered an unrecoverable error), then the consumer client needs to send a command to renew the lease with Azure Storage Queue. If the consumer fails to do this, the message will be put back into the queue. Max renewal duration is 7 days.

- [Azure.Core library for .NET](https://www.nuget.org/packages/azure.core/): Shared primitives, abstractions
- [Azure.Storage.Common client library for .NET](https://www.nuget.org/packages/azure.storage.common/): Infrastructure shared by the other Azure Storage client libraries.
- [Azure.Storage.Queues client library for .NET](https://www.nuget.org/packages/azure.storage.queues/): Working with Azure Queue Storage.
- [System.Configuration.ConfigurationManager library for .NET](https://www.nuget.org/packages/system.configuration.configurationmanager/): Configuration files for client applications.

```cs
// Get the connection string from app settings
// Example: DefaultEndpointsProtocol=https;AccountName={your_account_name};AccountKey={your_account_key};EndpointSuffix={endpoint_suffix}
string connectionString = ConfigurationManager.AppSettings["StorageConnectionString"];

// Instantiate a QueueClient which will be used to create and manipulate the queue
QueueClient queueClient = new QueueClient(connectionString, queueName);

// Create the queue if it doesn't already exist
await queueClient.CreateIfNotExistsAsync();

if (await queueClient.ExistsAsync())
{
    await queueClient.SendMessageAsync("message");

    // Peek at the next message
    // If you don't pass a value for the `maxMessages` parameter, the default is to peek at one message.
    PeekedMessage[] peekedMessage = await queueClient.PeekMessagesAsync();

    // Change the contents of a message in-place
    // This code saves the work state and grants the client an extra minute to continue their message (default is 30 sec).
    QueueMessage[] message = await queueClient.ReceiveMessagesAsync();
    // PopReceipt must be provided when performing operations to the message
    // in order to prove that the client has the right to do so when locked
    queueClient.UpdateMessage(message[0].MessageId,
            message[0].PopReceipt,
            "Updated contents",
            TimeSpan.FromSeconds(60.0)  // Make it invisible for another 60 seconds
        );

    // Dequeue the next message
    QueueMessage[] retrievedMessage = await queueClient.ReceiveMessagesAsync();
    Console.WriteLine($"Dequeued message: '{retrievedMessage[0].Body}'");
    await queueClient.DeleteMessageAsync(retrievedMessage[0].MessageId, retrievedMessage[0].PopReceipt);

    // Get the queue length
    QueueProperties properties = await queueClient.GetPropertiesAsync();
    int cachedMessagesCount = properties.ApproximateMessagesCount; // >= of actual messages count
    Console.WriteLine($"Number of messages in queue: {cachedMessagesCount}");

    // Delete the queue
    await queueClient.DeleteAsync();
}
```

```sh
az storage account create --name mystorageaccount --resource-group $resourceGroup --location eastus --sku Standard_LRS
az storage queue create --name myqueue --account-name mystorageaccount
az storage queue list --account-name mystorageaccount --output table
az storage message put --queue-name myqueue --account-name mystorageaccount --content "Hello, World!"
az storage message peek --queue-name myqueue --account-name mystorageaccount
az storage message get --queue-name myqueue --account-name mystorageaccount
az storage message delete --queue-name myqueue --account-name mystorageaccount --message-id <message-id> --pop-receipt <pop-receipt>
az storage queue delete --name myqueue --account-name mystorageaccount
```

# [Azure Service Bus](https://docs.microsoft.com/en-us/azure/service-bus-messaging/)

Endpoint: `https://servicebus.azure.net`

## Overview

Supports AMQP 1.0, enabling applications to work with Service Bus, and on-premises brokers like ActiveMQ or RabbitMQ.

Up to 80 GB only.

- **Queue**: Only one consumer receives and processes each message at a time (_point-to-point_ connection), and since messages are stored durably in the queue, producers and consumers don't need to handle messages concurrently.
- **Load-leveling**: Effectively buffering against fluctuating system loads, ensuring the system is optimized to manage the average load, instead of peaks.
- **Decoupling**: Client and service don't have to be online at the same time.
- [**Receive modes**](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-transfers-locks-settlement):
  - **Receive and delete**: _At-Most-Once_ approach. Messages are immediately consumed and removed.
  - **Peek lock**: _At-Least-Once_ approach. Messages are locked for processing, and the application must explicitly complete the processing to mark the message as consumed. If the application fails, the message can be abandoned or automatically unlocked after a timeout (1min default).
- **Topics:** Publishers send messages to a topic (1:n), and each message is distributed to all subscriptions registered with the topic.
- **Subscriptions:** Subscribers receive message copies from the topic, based on filter rules set on their subscriptions. Subscriptions act like virtual queues and can apply filters to receive specific messages. Any consumers of the subscription automatically benefit from the filter.
- **Rules and actions**: You can configure subscriptions to process messages with specific characteristics differently. This is done using **filter actions**. When a subscription is created, you can define a filter expression that operates on message properties - system (ex. `"Label"`) or custom (ex: `"StoreName"`). This expression allows you to copy only the desired subset of messages to the subscription queue. If no SQL filter expression is provided, the filter action applies to all messages in that subscription.

## Tiers

| Feature                          | Basic             | Standard                                    | Premium                                  |
| -------------------------------- | ----------------- | ------------------------------------------- | ---------------------------------------- |
| **Core Entities**                |                   |                                             |                                          |
| Queues                           | ✅                | ✅                                          | ✅                                       |
| Topics & Subscriptions           | ❌                | ✅                                          | ✅                                       |
| Pricing Model                    | Pay-per-operation | Pay-per-operation (base charge + operation) | Per Messaging Unit (dedicated resources) |
| **Advanced Messaging**           |                   |                                             |                                          |
| Sessions                         | ❌                | ✅                                          | ✅                                       |
| Transactions                     | ❌                | ✅                                          | ✅                                       |
| De-Duplication                   | ❌                | ✅                                          | ✅                                       |
| Scheduled Messages               | ✅                | ✅                                          | ✅                                       |
| Dead-lettering                   | ❌                | ✅                                          | ✅                                       |
| Auto-forwarding                  | ❌                | ✅                                          | ✅                                       |
| **Performance & Scale**          |                   |                                             |                                          |
| Resource Model                   | Shared            | Shared                                      | Dedicated (predictable performance)      |
| Max Message/Batch Size           | 256 KB            | 256 KB                                      | 100 MB                                   |
| Throughput                       | Variable          | Variable                                    | High & Predictable                       |
| **Enterprise & Network**         |                   |                                             |                                          |
| Availability Zones               | ❌                | ❌                                          | ✅                                       |
| Geo-Disaster Recovery            | ❌                | ❌                                          | ✅                                       |
| VNet Integration                 | ❌                | ❌                                          | ✅                                       |
| Private Endpoints                | ❌                | ❌                                          | ✅                                       |
| JMS (Java Messaging) 2.0 Support | ❌                | ❌                                          | ✅                                       |

**📝 NOTE:** _Sessions_ meant here are necessary for FIFO guarantee and long-running polling.  
**📝 NOTE:** Specific to Service Bus Dead-letter only, those messages NEVER expire. You will need to purposefully interact with the dead-letter queue to remove these messages, if desired.

## Components

- **Namespace**: A container for all messaging components.
- **Queues** (point-to-point communication): Send and receive messages from.  
  Multiple queues and topics are supported in a single namespace, and namespaces often serve as application containers.
- **Topics**: **Imagine topics as queues versioned per each subscriber.** Subscriber gets all the benefits of viewing their own version of the main queue. Used in publish/subscribe scenarios. It contains multiple independent subscriptions called entities.

**📝 NOTE:** You must use Topics+Subscriptions if you need multicasting. Service Bus Queues do not support multicasting. Same applies to _Filtering_ feature.  
**📝 NOTE:** The _Via_ entity acts as a transaction coordinator that allows you to send messages to multiple queues or topics in an all-or-nothing operation.

## Payload and serialization

In the form of key-value pairs. The payload is always an opaque _binary block_ when stored or in transit. Its format can be described using the `ContentType` property. Applications are advised to manage object serialization themselves.

The AMQP protocol serializes objects into an AMQP object, retrievable by the receiver using `GetBody<T>()`. Objects are serialized into an AMQP graph of `ArrayList` and `IDictionary<string,object>`.

Each message has two sets of properties: _system-defined broker properties_, and _user-defined properties_.

### Message Routing and Correlation

Broker properties like `To`, `ReplyTo`, `ReplyToSessionId`, `MessageId`, `CorrelationId`, and `SessionId` assist in message routing. The routing patterns are:

- **Simple request/reply**: Publishers send messages and await replies in a queue. Replies are addressed using `ReplyTo` and correlated with `MessageId`. Multiple replies are possible.
- **Multicast request/reply**: Similar to the simple pattern, but messages are sent to a topic, and multiple subscribers can respond. Responses can be distributed if `ReplyTo` points to a topic.
- **Multiplexing**: Streams of related messages are directed through one queue or subscription using matching `SessionId` values.
- **Multiplexed request/reply**: Multiple publishers share a reply queue, and replies are guided by `ReplyToSessionId` and `SessionId`.

Routing is managed internally, but applications can also use user properties for routing, as long as they don't use the reserved `To` property.

## Advanced features

| Feature                    | Description                                                                                                                                                                                                                                                                                                                                                          |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Message sessions           | The SessionId acts as the partition key. This ensures that all messages with the same session ID are routed to the same partition, maintaining strict order (FIFO) and session state consistency.                                                                                                                                                                    |
| Parallel Stream Processing | Can process messages as parallel, long-running streams using **session ID**                                                                                                                                                                                                                                                                                          |
| Auto-forwarding            | Removes messages from a queue or subscription and transfer it to a different queue or topic within the same namespace. Similar to fan-out in Durable functions. **📝 NOTE:** it preserves message properties (including MessageId, SessionId, CorrelationId, etc.), but stateful features like locks, deduplication caches, and sessions don’t flow across entities. |
| Dead-letter queue          | Holds messages that can't be delivered, allows for removal and inspection.                                                                                                                                                                                                                                                                                           |
| Scheduled delivery         | Allows delayed processing by scheduling a message for a certain time.                                                                                                                                                                                                                                                                                                |
| Message deferral           | Defers message retrieval until a later time, message remains set aside.                                                                                                                                                                                                                                                                                              |
| Batching                   | Delays sending a message for a certain period.                                                                                                                                                                                                                                                                                                                       |
| Transactions               | Groups operations into an execution scope for a single messaging entity.                                                                                                                                                                                                                                                                                             |
| Auto-delete on idle        | Automatically deletes a queue after a specified idle interval. Minimum duration is 5 minutes.                                                                                                                                                                                                                                                                        |
| De-duplicate               | Resends same message or discards any duplicate copies in case of send operation doubt. **📝 NOTE:** It is achieved by maintaining an in-memory and persisted cache of recent MessageId values within the configured deduplication window (e.g., 10 minutes). This logic applies at the entity level (Queue or Topic Subscription).                                   |
| Security protocols         | Supports protocols like SAS, RBAC, and Managed identities for Azure.                                                                                                                                                                                                                                                                                                 |
| Geo-disaster recovery      | Continues data processing in a different region or datacenter during downtime. When a failover is initiated, the original primary namespace remains available, and the alias (endpoint) is simply pointed to the secondary namespace. The replication link is broken.                                                                                                |
| Security                   | Supports standard AMQP 1.0 and HTTP/REST protocols.                                                                                                                                                                                                                                                                                                                  |
| At-Most-Once Delivery      | When enabled, if something goes wrong during processing of the message, there is no chance for recovery. As the message is retrieved successfully, the message is immediately deleted from the queue so that At-Most-Once delivery is achieved. If something goes wrong, the message is lost forever.                                                                |
| Multicasting               | A message can be received by multiple clients at the same time or asynchronously.                                                                                                                                                                                                                                                                                    |

## [Message expiration (TTL)](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-expiration)

- **Setting TTL**: Message-level TTL **cannot be higher than topic's (queue) TTL**. If not set, queue's TTL is used.
- **Message Lock**: When a message is locked, its expiration is halted. Expiration resumes if the lock expires or the message is abandoned.
- **Dead-Letter Queue**: Expired messages can be moved to a dead-letter queue for further inspection.

## [Scheduled messages](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-sequencing#scheduled-messages)

To schedule messages, you have two options:

1. Use the regular API and set the `ScheduledEnqueueTimeUtc` property before sending.
2. Use the schedule API, provide the message and time, and get a `SequenceNumber` for possible cancellation later.

## [Best Practices for performance improvements](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-performance-improvements?tabs=net-standard-sdk-2)

- Always prefer asynchronous methods to improve performance and responsiveness.
- Message factories: Use multiple message factories to enhance throughput, particularly when both sides have a large number of senders and receivers. Opt for a single message factory per process when one side has significantly fewer senders or receivers.
- Batched store access (batching): Increases throughput. Consider disabling for low-latency requirements. A batch is always written to a single partition to ensure atomicity. The specific partition is chosen based on _PartitionKey_ or _SessionId_, or assigned by Service Bus if neither is present.
- Prefetch count:
  - **Default:** Set it to 20 times the speed (processing rate) of your fastest receiver.
  - **If you have many receivers:** Use a lower number to save resources.
  - **If you need low-latency (fast response) with multiple clients:** Set it to 0.  
    **📝 NOTE:** When Prefetch is enabled, the Service Bus SDK pulls multiple messages (say, 100) into a local cache. Each message’s PeekLock timer starts immediately upon retrieval from the broker, not when your code actually begins processing it.

## Security

RBAC:

- Azure Service Bus Data Owner
- Azure Service Bus Data Sender
- Azure Service Bus Data Receiver

Also supports SAS and Managed Identities

## [Topic filters and actions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/topic-filters)

### Filters

- **SQL Filters** (`SqlRuleFilter`):

  - **Use**: Complex conditions using SQL-like expressions. All system properties must be prefixed with `sys.` in the conditional expression. (IS NULL, EXISTS, LIKE, AND/OR/NOT).
  - **Example**: Filtering messages having specific property value (or not) and quantities.
  - **Impact**: Lower throughput compared to Correlation Filters.

- **Boolean Filters** (`TrueRuleFilter` and `FalseRuleFilter`):

  - **Use**: Select all (TrueFilter) or none (FalseFilter) of the messages.
  - **Example**: `new TrueRuleFilter()` for all messages.

- **Correlation Filters** (`CorrelationRuleFilter`):
  - **Use**: Match messages based on specific properties like CorrelationId, ContentType, Label, MessageId, ReplyTo, ReplyToSessionId, SessionId, To, any user-defined properties.
  - **Example**: Filtering messages with a specific subject and correlation ID.
  - **Impact**: More efficient in processing, preferred over SQL filters.

### Actions

- **Use**: Modify message properties after matching and before selection.
- **Example**: Setting a new quantity value if property matches a value (or not).

### Usage Patterns

- **Broadcast Pattern**: Every subscription gets a copy of each message.
- **Partitioning Pattern**: Distributes messages across subscriptions in a mutually exclusive manner.
- **Routing Pattern**: When you need to route messages based on their content or some attributes.

## [Azure Service Bus for .NET](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/service-bus)

## Message Queue Implementation

```cs
using Azure.Messaging.ServiceBus;

string queueName = "az204-queue";

ServiceBusClient GetClient()
{
    return new ServiceBusClient(connectionString);

    // Alternatives

    var serviceBusEndpoint = new Uri($"sb://{serviceBusNamespace}.servicebus.windows.net/");

    // SAS
    return new ServiceBusClient(serviceBusEndpoint, new AzureNamedKeyCredential(sharedAccessKeyName, sharedAccessKey));

    // Managed identity
    return new ServiceBusClient(serviceBusEndpoint, new DefaultAzureCredential());
}

await using (var client = GetClient())
{
    await using (sender = client.CreateSender(queueName))
    using (ServiceBusMessageBatch messageBatch = await sender.CreateMessageBatchAsync())
    {
        for (int i = 1; i <= 3; i++)
            if (!messageBatch.TryAddMessage(new ServiceBusMessage($"Message {i}")))
                throw new Exception($"Exception {i} has occurred.");
        await sender.SendMessagesAsync(messageBatch);
        Console.WriteLine($"A batch of three messages has been published to the queue.");

        sender.SendMessagesAsync(new ServiceBusMessage($"Messages complete"));
    }

    // default receive mode for the ServiceBusProcessor (and the underlying ServiceBusReceiver) is PeekLock.
    using (var processor = client.CreateProcessor(queueName, new ServiceBusProcessorOptions()))
    {
        processor.ProcessMessageAsync += MessageHandler;
        processor.ProcessErrorAsync += ErrorHandler;
        await processor.StartProcessingAsync();

        Console.WriteLine("Wait for a minute and then press any key to end the processing");
        Console.ReadKey();

        Console.WriteLine("\nStopping the receiver...");
        await processor.StopProcessingAsync();
        Console.WriteLine("Stopped receiving messages");
    }
}

async Task MessageHandler(ProcessMessageEventArgs args)
{
    string body = args.Message.Body.ToString();
    Console.WriteLine($"Received: {body}");
    // complete the message. message is deleted from the queue.
    // This action is only necessary and appropriate in PeekLock mode.
    // Alternative to Complete are Abandon and DeadLetter.
    await args.CompleteMessageAsync(args.Message);
}

Task ErrorHandler(ProcessErrorEventArgs args)
{
    Console.WriteLine(args.Exception.ToString());
    return Task.CompletedTask;
}
```

## Topics/Subscription Implementation

```cs
using Azure.Messaging.ServiceBus;

// --- Client Setup ---
// The GetClient function remains the same as it connects to the same (Service Bus Namespace).
ServiceBusClient GetClient()
{
    return new ServiceBusClient(connectionString);

    // Alternatives

    var serviceBusEndpoint = new Uri($"sb://{serviceBusNamespace}.servicebus.windows.net/");

    // SAS
    return new ServiceBusClient(serviceBusEndpoint, new AzureNamedKeyCredential(sharedAccessKeyName, sharedAccessKey));

    // Managed identity
    return new ServiceBusClient(serviceBusEndpoint, new DefaultAzureCredential());
}

ServiceBusAdministrationClient GetAdminClient()
{
    return new ServiceBusAdministrationClient(connectionString);

    // Alternatives

    var serviceBusEndpoint = new Uri($"sb://{serviceBusNamespace}.servicebus.windows.net/");

    // SAS
    return new ServiceBusAdministrationClient(serviceBusEndpoint, new AzureNamedKeyCredential(sharedAccessKeyName, sharedAccessKey));

    // Managed Identity
    return new ServiceBusAdministrationClient(serviceBusEndpoint, new DefaultAzureCredential());
}

// --- Main Application Logic ---
// NOTE: You have to create topics/subscriptions using adminClient if needed
await using (var adminClient = GetAdminClient()) // ServiceBusAdministrationClient for management
{
    // 1. Check Topic Existence and Create
    try
    {
        // Throws exception if not found, making this functionally idempotent.
        await adminClient.GetTopicAsync(topicName);
    }
    catch (ServiceBusException ex) when (ex.Reason == ServiceBusException.FailureReason.ResourceNotFound)
    {
        await adminClient.CreateTopicAsync(topicName);
    }

    // 2. Check Subscription Existence and Create
    try
    {
        await adminClient.GetSubscriptionAsync(topicName, subscriptionName);
    }
    catch (ServiceBusException ex) when (ex.Reason == ServiceBusException.FailureReason.ResourceNotFound)
    {
        await adminClient.CreateSubscriptionAsync(
            new CreateSubscriptionOptions(topicName, subscriptionName),
            new CreateRuleOptions("someFilter", new TrueRuleFilter()));
    }
}

await using (var client = GetClient())
{
    // 3. Publishing (Sending to a Topic)
    Console.WriteLine($"\n--- Publishing to Topic: {topicName} ---");

    // Create a ServiceBusSender to send messages to the specified TOPIC.
    await using (var sender = client.CreateSender(topicName))
    // Create a batch to send multiple messages efficiently.
    using (ServiceBusMessageBatch messageBatch = await sender.CreateMessageBatchAsync())
    {
        // Add messages to the batch.
        for (int i = 1; i <= 3; i++)
            if (!messageBatch.TryAddMessage(new ServiceBusMessage($"Topic Message {i}")))
                throw new Exception($"Message {i} was too large for the batch.");

        // Send the batch of messages to the Topic.
        await sender.SendMessagesAsync(messageBatch);
        Console.WriteLine($"A batch of three messages has been published to the topic.");

        // Send a single, separate message.
        await sender.SendMessagesAsync(new ServiceBusMessage($"Messages complete"));
    }

    // 4. Receiving (Consuming from a Subscription)
    Console.WriteLine($"\n--- Receiving from Subscription: {subscriptionName} on Topic: {topicName} ---");

    // Create a ServiceBusProcessor from a SPECIFIC TOPIC/SUBSCRIPTION combination.
    // The default receive mode is PeekLock.
    using (var processor = client.CreateProcessor(topicName, subscriptionName, new ServiceBusProcessorOptions()))
    {
        // Register the handler for when a message is received.
        processor.ProcessMessageAsync += MessageHandler;

        // Register the handler for any errors that occur during processing.
        processor.ProcessErrorAsync += ErrorHandler;

        // Start the message processing loop.
        await processor.StartProcessingAsync();

        Console.WriteLine("Wait for a minute and then press any key to end the processing...");
        Console.ReadKey();

        // Cleanly stop the processor.
        Console.WriteLine("\nStopping the receiver...");
        await processor.StopProcessingAsync();
        Console.WriteLine("Stopped receiving messages");
    }
}

// Rest of implementation is same as the previous one.
```

```sh
az servicebus namespace create --name mynamespace --resource-group $resourceGroup --location eastus
az servicebus queue create --name myqueue --namespace-name mynamespace --resource-group $resourceGroup

az servicebus queue list --namespace-name mynamespace --resource-group $resourceGroup

az servicebus namespace authorization-rule keys list --name RootManageSharedAccessKey --namespace-name mynamespace --resource-group $resourceGroup --query primaryConnectionString

# Send, Peek, Delete - You would use an SDK or other tooling

az servicebus queue delete --name myqueue --namespace-name mynamespace --resource-group $resourceGroup
az servicebus namespace delete --name mynamespace --resource-group $resourceGroup
```

```ps
New-AzServiceBusNamespace -ResourceGroupName $resourceGroup -Name myNamespace -SkuName Premium -Location northeurope -IdentityType UserAssigned
```

**📝 NOTE:** Renewing the PeekLock only extends the time the current receiver has to process the message; it does not "refresh" the message's lifetime. If the processing takes longer than the message's total TTL, the message will expire immediately after the lock is finally released or expired.

## [Claim Check Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check)

Use the Claim-Check pattern, and don't send large messages to the messaging system. Instead, send the payload to an external data store and generate a claim-check token for that payload.
![alt text](https://learn.microsoft.com/en-us/azure/architecture/patterns/_images/claim-check-diagram.svg)

1. Payload
2. Save payload in data store.
3. Generate claim-check token and send message with claim-check token.
4. Receive message and read claim-check token.
5. Retrieve the payload.
6. Process the payload.

<br>

| Service                 | Streaming Context | Description                                                                                                                                                                                                                                                                                       |
| ----------------------- | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Service Bus "Streaming" | Read experience   | The text calls long-polling "like streaming messages into your client." This refers to the read experience. Because the server holds your request open, new messages feel like they are being streamed to you instantly. But you are still processing them one by one, as discrete units of work. |
| Event Hubs "Streaming"  | Write-path        | The text refers to "streaming data into an event hub." This refers to the write-path. Event Hubs is designed for millions of devices to all "stream" (continuously write) their data to the service simultaneously.                                                                               |

<br>

---

# [Shared Access Signatures (SAS)](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)

A Shared Access Signature (SAS) is a URI that grants restricted access rights to Azure Storage resources. It's a signed URI that points to one or more storage resources and includes a token that contains a special set of query parameters.

Use a SAS for secure, temporary access to your storage account, especially when users need to read/write their own data or for copying data within Azure Storage.

**📝 NOTE:** You should prefer Entra ID

## Types of SAS

1. [**User Delegation SAS**](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-user-delegation-sas-create-dotnet): This method uses Microsoft Entra ID credentials to create a SAS. It's a secure way to grant limited access to your Azure Storage resources without sharing your account key. It's recommended when you want to provide fine-grained access control to clients who are authenticated with Entra ID. The account _must have_ `generateUserDelegationKey` permission, or `Contributor` role.

1. [**Service SAS**](https://learn.microsoft.com/en-us/azure/storage/blobs/sas-service-create-dotnet): This method uses your storage account key to create a SAS. It's a straightforward way to grant limited access to your Azure Storage resources. However, it's less secure than the User Delegation SAS because it involves sharing your account key. It's typically used when you want to provide access to clients who are not authenticated with Entra ID.

1. [**Account SAS**](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-sas-create-dotnet): This method uses your storage account key to create a SAS. It's created at the storage account level, allowing access to multiple services within the account. It's typically used when you need to provide access to several services in your storage account. However, it involves sharing your account key, similar to the Service SAS.

   - Ad hoc SAS: Defines start, expiry, and permissions in the SAS URI. Any SAS can be an ad hoc SAS.
   - Service SAS: Uses a stored policy on resources to inherit start, expiry, and permissions.

## How SAS Works

A SAS requires two components: a URI to the resource you want to access and a SAS token that you've created to authorize access to that resource.

- **URI**: `https://<account>.blob.core.windows.net/<container>/<blob>?`
- **SAS token**: `sp=r&st=2020-01-20T11:42:32Z&se=2020-01-20T19:42:32Z&spr=https&sv=2019-02-02&sr=b&sig=SrW1HZ5Nb6MbRzTbXCaPm%2BJiSEn15tC91Y4umMPwVZs%3D`

[Reference](https://learn.microsoft.com/en-us/rest/api/storageservices/create-service-sas):

| Component | Friendly Name                           | Description                                                                                                                                                                                       |
| --------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sp        | `S`hared `P`ermissions                  | Controls the access rights. Possible values include: `a`dd, `c`reate, `d`elete, `l`ist, `r`ead, `w`rite. Ex: `sp=acdlrw` grants all the available rights.                                         |
| st        | `S`hared Access Signature Start `T`ime  | The date and time when access starts. Ex: `st=2023-07-28T11:42:32Z` means the access starts at 11:42:32 UTC on July 28, 2023.                                                                     |
| se        | `S`hared Access Signature `E`xpiry Time | The date and time when access ends. Ex: `se=2023-07-28T19:42:32Z` means the access ends at 19:42:32 UTC on July 28, 2023.                                                                         |
| sv        | `S`torage API `V`ersion                 | The version of the storage API to use. Ex: `sv=2020-02-10` means the storage API version 2020-02-10 is used.                                                                                      |
| sr        | `S`torage `R`esource                    | The kind of storage being accessed. Possible values include: `b`lob, `f`ile, `q`ueue, `t`able, `c`ontainer, `d`irectory. Ex: `sr=b` means a blob is being accessed.                               |
| sig       | `Sig`nature                             | The cryptographic signature. Ex: `sig=SrW1HZ5Nb6MbRzTbXCaPm%2BJiSEn15tC91Y4umMPwVZs%3D` is a cryptographic signature.                                                                             |
| sip       | `S`ource `IP` Range                     | (Optional) Allowed IP addresses or IP range. Ex: `sip=168.1.5.60-168.1.5.70` means only the IP addresses from 168.1.5.60 to 168.1.5.70 are allowed.                                               |
| spr       | `S`upported `Pr`otocols                 | (Optional) Allowed protocols. Possible values include: 'https', 'http,https'. Ex: `spr=https` means only HTTPS is allowed.                                                                        |
| si        | `S`tored Access **Policy** `I`dentifier | (Optional) The name of the stored access policy. Ex: `si=MyAccessPolicy` means the stored access policy named "MyAccessPolicy" is used.                                                           |
| rscc      | `R`e`s`ponse `C`ache `C`ontrol          | (Optional) The response header override for cache control. Ex: `rscc=public` means the "Cache-Control" header is set to "public".                                                                 |
| rscd      | `R`e`s`ponse `C`ontent `D`isposition    | (Optional) The response header override for content disposition. Ex: `rscd=attachment; filename=example.txt` means the "Content-Disposition" header is set to "attachment; filename=example.txt". |
| rsce      | `R`e`s`ponse `C`ontent `E`ncoding       | (Optional) The response header override for content encoding. Ex: `rsce=gzip` means the "Content-Encoding" header is set to "gzip".                                                               |
| rscl      | `R`e`s`ponse `C`ontent `L`anguage       | (Optional) The response header override for content language. Ex: `rscl=en-US` means the "Content-Language" header is set to "en-US".                                                             |
| rsct      | `R`e`s`ponse `C`ontent `T`ype           | (Optional) The response header override for content type. Ex: `rsct=text/plain` means the "Content-Type" header is set to "text/plain".                                                           |

**📝 NOTE:** All 2-letter parameters are required, except `si` (Access Policy)

## Best Practices

- Always use HTTPS.
- Use user delegation SAS wherever possible.
- Set your expiration time to the smallest useful value.
- Only grant the access that's required.
- Create a middle-tier service to manage users and their access to storage when there's an unacceptable risk of using a SAS.

## [Stored Access Policies](https://learn.microsoft.com/en-us/rest/api/storageservices/define-stored-access-policy)

Allow you to group SAS and set additional constraints like start time, expiry time, and permissions. Work on **container** level.

Use `SetAccessPolicy` on `BlobContainer` to apply an array containing a single `BlobSignedIdentifier` that has a configured `BlobAccessPolicy` for the `AccessPolicy` property.

```cs
BlobSignedIdentifier identifier = new BlobSignedIdentifier
{
    Id = "stored access policy identifier",
    AccessPolicy = new BlobAccessPolicy
    {
        ExpiresOn = DateTimeOffset.UtcNow.AddHours(1),
        Permissions = "rw"
    }
};

blobContainer.SetAccessPolicy(permissions: new BlobSignedIdentifier[] { identifier });
```

```sh
az storage container policy create \
    --name <stored access policy identifier> \
    --container-name <container name> \
    --start <start time UTC datetime> \
    --expiry <expiry time UTC datetime> \
    --permissions <(a)dd, (c)reate, (d)elete, (l)ist, (r)ead, or (w)rite> \
    --account-key <storage account key> \
    --account-name <storage account name> \
```

To cancel (revoke) a policy, you can either delete it, rename it, or set its expiration time to a past date.

To remove all access policies from the resource, call the `Set ACL` operation with an empty request body.

## Working with SAS

Gist:

- Service SAS and Account SAS use `StorageSharedKeyCredential`; User delegation SAS use `DefaultAzureCredential` or similar Entra ID
- Service SAS and User delegation SAS use `BlobSasBuilder`; Account SAS uses `AccountSasBuilder`
- Set permissions: `BlobSasPermissions` for user and service; `AccountSasPermissions` for account
- Obtaining URI:
  - User delegation SAS: Use key generated from `BlobServiceClient.GetUserDelegationKeyAsync` as first param of `BlobSasBuilder.ToSasQueryParameters(key, accountName)`; pass it to `BlobUriBuilder(BlobClient.Uri).Sas`
  - Service SAS: `BlobClient.GenerateSasUri(BlobSasBuilder)`
  - Account SAS: `BlobSasBuilder.ToSasQueryParameters(sharedKeyCredential)` and construct Uri from it at root level (`https://{accountName}.blob.core.windows.net?{sasToken}`)

```cs
// Using StorageSharedKeyCredential with account name and key directly for authentication.
// This key has full permissions to all operations on all resources in your storage account.
// Works for all SAS types, but less secure.
var credential = new StorageSharedKeyCredential("<account-name>", "<account-key>");

// Using DefaultAzureCredential with Entra ID. More secure, but doesn't work for Service SAS.
// TokenCredential credential = new DefaultAzureCredential();

var serviceClient = new BlobServiceClient(new Uri("<account-url>"), credential);
var blobClient = serviceClient.GetBlobContainerClient("<container-name>").GetBlobClient("<blob-name>");

// Create a SAS token for the blob resource that's also valid for 1 day
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = blobClient.BlobContainerName,
    BlobName = blobClient.Name,
    Resource = "b", // HINT: in case of missing BlobName property, then Resource = "c"
    StartsOn = DateTimeOffset.UtcNow,
    ExpiresOn = DateTimeOffset.UtcNow.AddDays(1)
};
sasBuilder.SetPermissions(BlobSasPermissions.Read | BlobSasPermissions.Write);

////////////////////////////////////////////////////
// User Delegation SAS
////////////////////////////////////////////////////

// Request the user delegation key
// UserDelegationKey is used to sign the SAS token and has its own validity (can be used for multiple SAS)
UserDelegationKey userDelegationKey = await serviceClient.GetUserDelegationKeyAsync(
    DateTimeOffset.UtcNow,
    DateTimeOffset.UtcNow.AddDays(1));
var userSas = sasBuilder.ToSasQueryParameters(userDelegationKey, serviceClient.AccountName);
// Add the SAS token to the blob URI
BlobUriBuilder uriBuilder = new BlobUriBuilder(blobClient.Uri) { Sas = userSas };
var blobClientSASUserDelegation = new BlobClient(uriBuilder.ToUri());

////////////////////////////////////////////////////
// Service SAS
////////////////////////////////////////////////////

Uri blobSASURIService = blobClient.GenerateSasUri(sasBuilder);
var blobClientSASService = new BlobClient(blobSASURIService);
```

Account SAS:

````cs
var sharedKeyCredential = new StorageSharedKeyCredential("<account-name>", "<account-key>");

// Create a SAS token that's valid for one day
var sasBuilder = new AccountSasBuilder()
{
    Services = AccountSasServices.Blobs | AccountSasServices.Queues,
    ResourceTypes = AccountSasResourceTypes.Service,
    ExpiresOn = DateTimeOffset.UtcNow.AddDays(1),
    Protocol = SasProtocol.Https
};
sasBuilder.SetPermissions(AccountSasPermissions.Read | AccountSasPermissions.Write);

// Use the key to get the SAS token
// NOTE: You can pass sharedKeyCredential to ToSasQueryParameters (also valid for Service SAS)
var sasToken = sasBuilder.ToSasQueryParameters(sharedKeyCredential).ToString();

// Create a BlobServiceClient object with the account SAS appended
var blobServiceURI = $"https://{accountName}.blob.core.windows.net";
var blobServiceClientAccountSAS = new BlobServiceClient(
    new Uri($"{blobServiceURI}?{sasToken}"));
    ```

```sh
# Assign the necessary permissions to the user
az role assignment create \
 --role "Storage Blob Data Contributor" \
 --assignee <email> \
 --scope "/subscriptions/<subscription>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account>"

# Account Level SAS: --account-name and --account-key
# Service Level SAS: --resource-types + same as above
# User Level SAS: --auth-mode login
# (applicable for Stored Access Policies as well)

# Generate a user delegation SAS for a container
az storage container generate-sas \
 --account-name <storage-account> \
 --name <container> \
 --permissions acdlrw \
 --expiry <date-time> \
 --auth-mode login \
 --as-user

# Generate a user delegation SAS for a blob
az storage blob generate-sas \
 --account-name <storage-account> \
 --container-name <container> \
 --name <blob> \
 --permissions acdrw \
 --expiry <date-time> \
 --auth-mode login \
 --as-user \
 --full-uri

# Revoke all user delegation keys for the storage account
az storage account revoke-delegation-keys \
 --name <storage-account> \
 --resource-group $resourceGroup
````

# [Azure Cache for Redis](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-overview)

Fully managed, in-memory Key/Value datastore. Used to complement database apps.
**Primary Goal:** To improve the performance and scalability of your applications by providing a low-latency, high-throughput store for frequently accessed data.

### Azure Cache for Redis - Patterns/Use Cases --> (will be referred to in implementation section)

| #   | Use Case                                         | Definition                                                                                                                                                        |
| :-- | :----------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Data Cache [cache-aside]**                     | Application loads frequently accessed data from the database into Redis on-demand, reducing database load and improving response times.                           |
| 2   | **Content Cache [static content]**               | Store static assets (images, CSS, JavaScript, HTML fragments) in Redis to reduce backend server load and accelerate page load times.                              |
| 3   | **Session Store [Instead of user cookies]**      | Store user session data in Redis to enable stateless web applications that can scale horizontally across multiple servers without losing user session state/data. |
| 4   | **Job and Message Queueing (Publish/Subscribe)** | Use Redis Pub/Sub or Lists to implement lightweight message brokers for real-time communication between microservices or background job processing.               |
| 5   | **Distributed Transactions**                     | Leverage Redis transactions (MULTI/EXEC) and Lua scripts to perform atomic operations across multiple keys, ensuring data consistency in distributed scenarios.   |

### Key Terms

- **Cache Hit:** Your application tries to get data from Redis and **finds it**. This is fast.
- **Cache Miss:** Your application tries to get data from Redis and **does not find it**. This is slow. The app must then fetch the data from the primary database (the "source of truth") and then add it to the cache for next time.
- **Eviction Policy:** What happens when the cache is full? The eviction policy defines which keys to remove (e.g., the "Least Recently Used" or **LRU** key) to make space for new data.
- **Serialization:** Redis stores strings. To store complex C# objects (like a `Product` or `User` class), you must **serialize** them into a string format (like JSON) before saving them, and **deserialize** them from JSON back into a C# object when you retrieve them.

## Tiers

| Tier                 | Key Features                                                                                                                                                                                                                                                                                                                                       | Use Case                                                                                                                                      |
| :------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------- |
| **Basic**            | • Single node<br/>• No SLA<br/>• Data Encryption<br/>• Network Isolation<br/>• Scaling                                                                                                                                                                                                                                                             | **Dev/Test only.** Never use in production.                                                                                                   |
| **Standard**         | • Two-node (Primary/Replica) for high availability<br/>• SLA<br/>• Data Encryption<br/>• Network Isolation<br/>• Scaling                                                                                                                                                                                                                           | **Production.** This is the baseline for reliable caching.                                                                                    |
| **Premium**          | All Standard features **PLUS**:<br/>• **Clustering** (HA and load distribution)<br/>• **Data Persistence** (RDB/AOF snapshots and backups)<br/>• **Zone Redundancy** (Availability zones)<br/>• **Geo-Replication** (DR, regional failover)<br/>• **Import/Export** (integrate with storage)<br/>• **VNET & Private Endpoint** (Network Isolation) | **Enterprise-grade production.** Required for high security (VNET), larger-than-memory datasets (persistence), or massive scale (clustering). |
| **Enterprise**       | All Premium features **PLUS**:<br/>• **RediSearch** (full-text search)<br/>• **RedisBloom** (probabilistic data structures)<br/>• **RedisTimeSeries** (time-series data)<br/>• **Active Geo-Replication** (multi-master, active-active)                                                                                                            | Specialized, high-performance scenarios requiring Redis modules and active-active replication.                                                |
| **Enterprise Flash** | All Enterprise features with:<br/>• **Flash Memory** (uses non-volatile memory to reduce costs)<br/>• Cost-optimized for large datasets                                                                                                                                                                                                            | Lower-cost enterprise option for scenarios where some performance trade-off is acceptable for significant cost savings.                       |

---

### How it works

Distributed Cache:

- Manage spices
- Cache aside [data]
- Static Content
- Geo positioning [like a CDN]

Session Store:

- 100000s of users
- reliability via data-replication
- shopping carts/cookies/login and session state/IoT Telemetry

Message Broker:

- Pub/Sub
- Queue

## Develop for Azure Cache for Redis

### Configuring at Azure

How to create a named instance at azure:

- Globally unique name: Numbers, letters, and `-` only. No start/end or consecutive `-` characters

### Redis Basic Commands

| Command                   | Description                                        |
| :------------------------ | :------------------------------------------------- |
| `Ping`                    | Returns `Pong` - used to test connection           |
| `SetString` / `GetString` | Set a string value / Get a string value            |
| `exists[key]`             | Check if a key exists (returns 1 or 0)             |
| `incr[key]`               | Increment the integer value of a key by 1          |
| `flushdb`                 | Delete all key/value pairs in the current database |

- Caching needs to have a Time To Live (TTL) to be valid. You can set the TTL to expire after a number of seconds.

### Creating & Managing with Azure CLI

```bash
# Create a resource group
az group create --name MyRedisRG --location "East US"

# Create a Standard C1 (1 GB) Redis cache
az redis create \
    --resource-group MyRedisRG \
    --name "az204-redis-cache" \
    --location "East US" \
    --sku Standard \
    --vm-size C1

# Create a Premium P1 cache with VNET
# First, you need a VNET and a dedicated subnet
# (Ensure the subnet has no other resources and no Service Endpoint)
az network vnet create --name MyVNET --resource-group MyRedisRG --subnet-name RedisSubnet

# Get the full Subnet ID
SUBNET_ID=$(az network vnet subnet show --name RedisSubnet --vnet-name MyVNET --resource-group MyRedisRG --query id -o tsv)

az redis create \
    --resource-group MyRedisRG \
    --name "az204-premium-cache" \
    --location "East US" \
    --sku Premium \
    --vm-size P1 \
    --subnet-id $SUBNET_ID

# Get the primary access key. Connection string is sth like: `az204-redis-cache.redis.cache.windows.net:6380,password=YOUR_KEY_HERE,ssl=True,abortConnect=False`
az redis list-keys \
    --resource-group MyRedisRG \
    --name "az204-redis-cache" \
    --query "primaryKey" -o tsv

```

## Using Redis in .NET

StackExchange.Redis client library to be added:

```bash
dotnet add package StackExchange.Redis
```

```cs
using StackExchange.Redis;

// Static class to manage the singleton connection
public static class RedisConnection
{
    // Use Lazy<T> for thread-safe, lazy initialization
    private static readonly Lazy<ConnectionMultiplexer> LazyConnection =
        new Lazy<ConnectionMultiplexer>(() =>
        {
            string cacheConnection = "YOUR_CONNECTION_STRING_FROM_AZURE";
            return ConnectionMultiplexer.Connect(cacheConnection);
        });

    public static ConnectionMultiplexer Connection => LazyConnection.Value;

    public static IDatabase GetDatabase()
    {
        return Connection.GetDatabase();
    }
}

```

### Patterns 1 & 2:

```cs
// Example object to cache
public class Product
{
    public string Id { get; set; }
    public string Name { get; set; }
    public double Price { get; set; }
}

public class ProductRepository
{
    private readonly IDatabase _cache;

    public ProductRepository()
    {
        _cache = RedisConnection.GetDatabase();
    }

    public async Task<Product> GetProductAsync(string productId)
    {
        // 1. Define the cache key
        string cacheKey = $"product:{productId}";

        // 2. Try to get the item from the cache (Cache Hit)
        RedisValue cachedProductJson = await _cache.StringGetAsync(cacheKey);

        if (cachedProductJson.HasValue)
        {
            // Found it! Deserialize and return
            return JsonSerializer.Deserialize<Product>(cachedProductJson);
        }

        // 3. Not found (Cache Miss) - Get from database
        // (Simulating a slow database call)
        Console.WriteLine("Cache Miss. Fetching from database...");
        await Task.Delay(1000);
        var product = new Product { Id = productId, Name = "Super Widget", Price = 99.99 };

        // 4. Serialize and add to cache for next time
        string productJson = JsonSerializer.Serialize(product);

        // Set an expiration time (e.g., 10 minutes)
        await _cache.StringSetAsync(cacheKey, productJson, TimeSpan.FromMinutes(10));

        return product;
    }
}

```

### Pattern 3

Add the session state NuGet package:

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

Configure services in Program.cs:

```cs
var builder = WebApplication.CreateBuilder(args);

// 1. Get the connection string from appsettings.json
var redisConnectionString = builder.Configuration.GetConnectionString("RedisCache");

// 2. Add Redis as the distributed cache
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = redisConnectionString;
    options.InstanceName = "YourAppPrefix_"; // Optional prefix for keys
});

// 3. Add and configure the session state provider to use Redis
builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(20); // Session timeout
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
});

// ... add controllers, etc.
var app = builder.Build();

// 4. IMPORTANT: Use session
app.UseSession();

// ... app.MapControllers(), etc.
app.Run();
```

Now in your controller, you can use HttpContext.Session:

```cs
[ApiController]
[Route("[controller]")]
public class MyController : ControllerBase
{
    [HttpGet]
    public string Get()
    {
        // Set a value in the session (stored in Redis)
        HttpContext.Session.SetString("UserName", "Alice");

        // Get a value
        string userName = HttpContext.Session.GetString("UserName");

        return $"User {userName} is in the session.";
    }
}
```

### Pattern 4:

This is great for decoupled, real-time communication.

**Publisher:**

```cs
using StackExchange.Redis;

// Get the subscriber interface from the connection
ISubscriber sub = RedisConnection.Connection.GetSubscriber();

// Define a channel name
RedisChannel channel = new RedisChannel("app:messages", RedisChannel.PatternMode.Literal);

Console.WriteLine("Publishing message...");

// Publish a message to the channel
await sub.PublishAsync(channel, "Hello, anyone there?");

Console.WriteLine("Message published.");
```

**Subscriber:**

```cs
using StackExchange.Redis;

// Get the subscriber interface
ISubscriber sub = RedisConnection.Connection.GetSubscriber();
RedisChannel channel = new RedisChannel("app:messages", RedisChannel.PatternMode.Literal);

Console.WriteLine("Subscribing to 'app:messages'...");

// Subscribe to the channel with a handler
await sub.SubscribeAsync(channel, (ch, message) =>
{
    Console.WriteLine($"Received message: {message}");
});

Console.WriteLine("Press Enter to quit...");
Console.ReadLine(); // Keep the subscriber alive
```

### **Information about keys**

- **Avoid long keys** - Keep keys short for better performance and memory efficiency
- **Maximum key size**: 512 MB

| Example Key                     | Structure                       | Description                                    |
| :------------------------------ | :------------------------------ | :--------------------------------------------- |
| `movie:scifi:title:Inception`   | object:category:attribute:value | A science fiction movie with title "Inception" |
| `user:1234:email`               | object:id:attribute             | Email address for user ID 1234                 |
| `product:electronics:laptop:99` | object:category:subcategory:id  | Laptop product with ID 99 in electronics       |
| `session:abc123:cart`           | object:id:attribute             | Shopping cart for session abc123               |

### **Persistence Options**

| Option                     | Description                                | How It Works                                                                         | Pros                                                      | Cons                                                                                        | Best For                                                                   |
| :------------------------- | :----------------------------------------- | :----------------------------------------------------------------------------------- | :-------------------------------------------------------- | :------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------- |
| **None**                   | Default setting - no data is saved to disk | If cache restarts or crashes, all data is lost                                       | Simple, no disk I/O overhead                              | Complete data loss on restart                                                               | Temporary cache where data can be easily rebuilt from primary database     |
| **RDB (Redis Database)**   | Takes periodic snapshots of entire dataset | Saves complete dataset to disk file at intervals (e.g., every 10 minutes)            | Efficient for backups, faster restarts, smaller file size | Can lose data between snapshots (e.g., 9 minutes of data if crash occurs between snapshots) | Scenarios where some data loss is acceptable, backup and disaster recovery |
| **AOF (Append Only File)** | Logs every write command as it happens     | Records each write operation (`SET`, `INCR`, etc.) to a log file, replays on restart | Much more durable, minimal data loss (1-2 seconds max)    | Larger log files, slightly slower due to constant disk writes                               | Critical data that cannot be lost, requires maximum durability             |

### **Transactions**

Utilize ServiceStack.Redis in .Net:

- CreateTransaction()
- QueueCommand()
- Commit()

Get the connection string:

```bash
rg=rg-name-here
redis_name=redis-cache-name-here

redis_key=$(az redis list-keys \
    --name "$redis_name" \
    --resource-group $rg \
    --query primaryKey \
    --output tsv)

echo $redis_key
echo "$redis_key"@"$redis_name".redis.cache.windows.net:6380?ssl=true
```

Use the connection string:

```c#
private static bool UtilizeTransactions(Dictionary<string, string> data, int expirationSeconds)
{
    bool transactionResult = false;

    var redisConnectionString = _configuration["Redis:RedisConnectionString"];
    using (RedisClient redisClient = new RedisClient(redisConnectionString))
    {
        using (var transaction = redisClient.CreateTransaction())
        {
            //Add multiple operations to the transaction
            foreach (var item in data)
            {
                transaction.QueueCommand(c => c.Set(item.Key, item.Value));
                if (expirationSeconds > 0)
                {
                    transaction.QueueCommand(c => ((RedisNativeClient)c).Expire(item.Key, expirationSeconds));
                }
            }

            //Commit and get result of transaction
            transactionResult = transaction.Commit();
        }
    }

    Console.WriteLine(transactionResult ? "Transaction committed" : "Transaction failed to commit");

    return transactionResult;
}
```

### **Expiration (TTL)**

```cs
using StackExchange.Redis;

// Get your connection (ideally as a singleton)
IDatabase cache = RedisConnection.Connection.GetDatabase();

string cacheKey = "product:123";
string productJson = "{ 'name': 'Super Widget', 'price': 99.99 }";

// Set the key with an absolute expiration of 10 minutes
TimeSpan absoluteExpiration = TimeSpan.FromMinutes(10);
await cache.StringSetAsync(cacheKey, productJson, absoluteExpiration);
```

### **Eviction policies and Memory Management**

| Title               | Purpose                                                                                                    |
| ------------------- | ---------------------------------------------------------------------------------------------------------- |
| **noeviction**      | No eviction - errors are thrown when you are out of memory                                                 |
| **allkeys-lru**     | Looks at all keys and removes the least-recently-used key                                                  |
| **allkeys-random**  | Looks at all keys and removes one at random                                                                |
| **allkeys-lfu**     | Looks at all keys and removes the least-frequently-used key                                                |
| **volatile-lru**    | Looks at all keys with expiration set and removes the least-recently-used of these keys                    |
| **volatile-ttl**    | Looks at all keys with expiration set and removes the key from these with the least remaining time to live |
| **volatile-random** | Looks at all keys with expiration set and removes a key from these at random                               |
| **volatile-lfu**    | Looks at all keys with expiration set and removes the least-frequently used key from these                 |

### **Expiration vs Eviction**

| Concept              | Description                                                                                                                                          | Trigger                      |
| :------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------- |
| **Expiration (TTL)** | Proactively telling Redis when to delete a specific key. The key is removed because its time is up.                                                  | Time-based (TTL expires)     |
| **Eviction**         | Reactive process that happens when cache is full and needs to make room for new data. Redis "evicts" (deletes) old keys based on an eviction policy. | Memory pressure (cache full) |

### **Message Order**

Queue functionality guarantees order:

```c#
var channel = multiplexer.GetSubscriber().Subscribe("messages");
channel.OnMessage(message =>
{
    Console.WriteLine((string)message.Message);
});
```

Whereas concurrent processing (Pub/Sub) has no order guarantee:

```c#
var channel = multiplexer.GetSubscriber().Subscribe("messages", (channel, message) => {
    Console.WriteLine((string)message);
});
```

# [Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/overview)

Newer, distinct service offering from Microsoft, separate from the classic "Azure Cache for Redis." It is designed to deliver the latest Redis innovations as a fully managed service, with a focus on hyperscale workloads, advanced data structures, and AI-driven applications.

## Key Differentiators vs. Azure Cache for Redis

<br>

| Feature             | Azure Cache for Redis (Classic)                                                           | Azure Managed Redis (New Service)                                                                       |
| :------------------ | :---------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------ |
| **Primary Focus**   | General-purpose caching (Basic/Standard) and enterprise features (Premium).               | Latest Redis innovations, AI/Vector search, advanced data structures.                                   |
| **Modules**         | Modules (RediSearch, etc.) are only available in the top **Enterprise Tiers**.            | Modules (RediSearch, RedisJSON, etc.) are integrated as core features.                                  |
| **Availability**    | Up to 99.9% (Standard/Premium).                                                           | Up to 99.999%.                                                                                          |
| **Geo-Replication** | **Passive Geo-Replication** (Premium): Asynchronous, single-master for disaster recovery. | **Active Geo-Replication**: Multi-master, enabling global read _and_ write operations.                  |
| **Dev/Test**        | **Basic Tier**: Single node, no SLA.                                                      | **Non-high availability configurations**: Offered as a cost-optimization option for dev/test.           |
| **Storage**         | RAM-only (Basic/Standard/Premium) or RAM + NVMe Flash (Enterprise Flash).                 | **Flash Optimized**: Uses NVMe storage to reduce costs for large datasets, similar to Enterprise Flash. |

<br>

## Components and Policies

<br>

| Component / Policy            | Key Characteristics                                                                                                                                                                                                                                                                                                                                      |
| :---------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Node Architecture**         | • Based on Redis Enterprise<br/>• Uses 2 VMs (nodes) with multiple Redis server processes (shards)<br/>• Primary and replica shards are distributed across both nodes<br/>• Includes a high-performance proxy for connection management<br/>• Uses a multi-threaded architecture for efficient vCPU use<br/>• Reserves ~20% memory for operations buffer |
| **OSS Cluster Policy**        | • Recommended for most applications<br/>• Implements the Redis Cluster API; clients connect directly to shards<br/>• Provides the best latency and throughput<br/>• Offers near-linear throughput scaling<br/>• Requires a client library that supports the Redis Cluster API                                                                            |
| **Enterprise Cluster Policy** | • Provides a single endpoint; acts as a proxy<br/>• Makes the cluster appear non-clustered to the client<br/>• Required for the RediSearch module<br/>• Only supports the NoEviction policy<br/>• Cannot be changed after the cache is created                                                                                                           |
| **Non-Clustered Policy**      | • No data sharding<br/>• Only available for caches ≤25 GB<br/>• Intended for migrating non-sharded environments or for heavy cross-slot command usage                                                                                                                                                                                                    |
| **Flash Optimized Tier**      | • 20% RAM (for all keys) and 80% NVMe Flash (for values)<br/>• Intelligently and automatically moves less-accessed data from RAM to Flash<br/>• Offers a trade-off: lower cost for reduced performance                                                                                                                                                   |
| **High Availability (HA)**    | • Can be disabled (not recommended for production)<br/>• Without HA: No replication, results in data loss during maintenance, and has lower performance                                                                                                                                                                                                  |

<br>

## Core Features & Built-in Modules

The primary advantage of Azure Managed Redis is its native support for advanced Redis modules.

- **RediSearch**: Transforms the cache into a powerful, low-latency search engine. It supports full-text search, complex queries, and **K-nearest neighbor (KNN) vector search**, which is essential for AI and RAG (Retrieval-Augmented Generation) applications.
- **RedisJSON**: Provides native support for storing, querying, and updating JSON documents _inside_ Redis. This avoids the need for client-side serialization/deserialization for complex objects and allows for atomic updates of specific JSON fields.
- **RedisTimeSeries**: A high-throughput database optimized for time-series data, such as IoT sensor readings, stock prices, or application metrics.
- **RedisBloom**: Adds support for probabilistic data structures like Bloom Filters and Cuckoo Filters. These are extremely memory-efficient for "might-contain" checks (e.g., checking if a username has already been taken).

<br>

## Key Features by Tier

<br>

| Feature                    | Memory Optimized | Balanced   | Compute Optimized | Flash Optimized |
| -------------------------- | ---------------- | ---------- | ----------------- | --------------- |
| Size Range                 | 12-1920 GB       | 0.5-960 GB | 3-720 GB          | 250-4500 GB     |
| Max Connections            | 15K-200K         | 15K-200K   | 30K-200K          | 75K-200K        |
| Replication & Failover     | ✓                | ✓          | ✓                 | ✓               |
| Network Isolation          | ✓                | ✓          | ✓                 | ✓               |
| Microsoft Entra ID Auth    | ✓                | ✓          | ✓                 | ✓               |
| Scaling (Preview)          | ✓                | ✓          | ✓                 | ✓               |
| Data Persistence (Preview) | ✓                | ✓          | ✓                 | ✓               |
| Zone Redundancy            | ✓                | ✓          | ✓                 | ✓               |
| Geo-Replication            | ✓\*              | ✓\*        | ✓                 | ✓               |
| Redis Modules              | ✓                | ✓          | ✓                 | ✓               |
| Import/Export              | ✓                | ✓          | ✓                 | ✓               |

**Tier Limitations:**

- Tiers >120 GB are in Public Preview
- Flash Optimized is in Public Preview
- \*Balanced B0/B1 don't support active geo-replication
- Cannot scale between in-memory and Flash tiers

<br>

## Clustering Policies

<br>

| Policy                                 | Description                                                | Connection Model                     | Performance                                                 | Requirements/Limitations                                                                             | Best For                                                                                                |
| :------------------------------------- | :--------------------------------------------------------- | :----------------------------------- | :---------------------------------------------------------- | :--------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------ |
| **OSS Cluster Policy** _(Recommended)_ | Implements Redis Cluster API                               | Clients connect directly to shards   | Best latency and throughput; Near-linear scaling with vCPUs | Requires client library support for Redis Cluster API                                                | Most production applications requiring high performance and scalability                                 |
| **Enterprise Cluster Policy**          | Single endpoint acts as proxy, routing requests internally | Single endpoint for all connections  | Good performance with proxy overhead                        | • Required for RediSearch module<br/>• NoEviction policy only<br/>• Cannot be changed after creation | Applications using RediSearch or those requiring simple connection model                                |
| **Non-Clustered Policy**               | No data sharding - single instance                         | Direct connection to single instance | Limited by single instance capacity                         | • Only for caches ≤25 GB<br/>• No horizontal scaling                                                 | • Migrating from non-sharded environments<br/>• Extensive cross-slot command usage<br/>• Small datasets |

<br>

## Developer Focus: Using Redis Modules in .NET

Since `StackExchange.Redis` does not have strongly-typed methods for every module (like `JSON.SET`), you often use the `IDatabase.Execute()` method to send raw Redis commands.

## Code Sample: Using RedisJSON (Server-Side JSON)

This pattern is different from your existing "Cache-Aside" sample. Instead of serializing a C# object into a string, we send the JSON string to Redis and let _Redis_ store and understand it as a JSON object.

```cs
using StackExchange.Redis;
using System.Text.Json;

public class JsonProductService
{
    private readonly IDatabase _cache;

    public JsonProductService()
    {
        _cache = RedisConnection.GetDatabase();
    }

    // Use JSON.SET to store the object, telling Redis to parse it
    public async Task SetProductAsJsonAsync(string productId)
    {
        var product = new { Id = productId, Name = "Super Widget", Price = 99.99, InStock = true };
        string productJson = JsonSerializer.Serialize(product);

        // The key is "product:json:123"
        // The path "$" means the root of the JSON document
        await _cache.ExecuteAsync("JSON.SET", $"product:json:{productId}", "$", productJson);

        Console.WriteLine("Stored product as a native JSON object in Redis.");
    }

    // Use JSON.GET to retrieve the full object or a sub-path
    public async Task GetProductPartsAsync(string productId)
    {
        string cacheKey = $"product:json:{productId}";

        // Get the full JSON object
        RedisResult fullObject = await _cache.ExecuteAsync("JSON.GET", cacheKey, "$");
        Console.WriteLine($"Full Object: {fullObject}");

        // Get only the 'Name' and 'Price' properties from the JSON doc
        RedisResult partialObject = await _cache.ExecuteAsync("JSON.GET", cacheKey, "$.Name", "$.Price");
        Console.WriteLine($"Partial Object: {partialObject}");
    }

    // Use JSON.NUMINCRBY to atomically update a number inside the JSON
    public async Task IncrementPriceAsync(string productId, double amount)
    {
        await _cache.ExecuteAsync("JSON.NUMINCRBY", $"product:json:{productId}", "$.Price", amount);
        Console.WriteLine($"Incremented price by {amount}.");
    }
}

```

## Code Sample: Using RediSearch (Text Search)

This sample demonstrates creating a search index and querying it.

```cs

using StackExchange.Redis;

public class ProductSearchService
{
    private readonly IDatabase _cache;

    public ProductSearchService()
    {
        _cache = RedisConnection.GetDatabase();
    }

    // 1. Create a search index
    public async Task CreateProductIndexAsync()
    {
        try
        {
            // Creates an index named "idx:products"
            // It indexes all Hashes prefixed with "product:hash:"
            // It indexes the 'Name' field as TEXT and 'Price' as NUMERIC
            await _cache.ExecuteAsync("FT.CREATE", "idx:products",
                "ON", "HASH",
                "PREFIX", "1", "product:hash:",
                "SCHEMA", "Name", "TEXT", "SORTABLE",
                          "Price", "NUMERIC", "SORTABLE");

            Console.WriteLine("Search index 'idx:products' created.");
        }
        catch (RedisServerException ex) when (ex.Message.Contains("Index already exists"))
        {
            Console.WriteLine("Search index already exists.");
        }

        // Add some data (using Hashes, as specified in the index)
        await _cache.HashSetAsync("product:hash:1", new HashEntry[] { new("Name", "Red Widget"), new("Price", 10) });
        await _cache.HashSetAsync("product:hash:2", new HashEntry[] { new("Name", "Blue Widget"), new("Price", 20) });
        await _cache.HashSetAsync("product:hash:3", new HashEntry[] { new("Name", "Red Chair"), new("Price", 50) });
    }

    // 2. Search the index
    public async Task SearchProductsAsync(string query)
    {
        // Use FT.SEARCH to query the index
        // This query finds items with "Red" in the Name, and Price between 0 and 60
        string redisQuery = $"@Name:{query} @Price:[0 60]";

        RedisResult searchResult = await _cache.ExecuteAsync("FT.SEARCH", "idx:products", redisQuery);

        Console.WriteLine($"Search results for: '{redisQuery}'");
        Console.WriteLine(searchResult.ToString());

        // Note: The result from ExecuteAsync is complex.
        // For production, you would use a client library like NRediSearch
        // to parse these results into objects.
    }
}

// --- How you might run this ---
// var searchService = new ProductSearchService();
// await searchService.CreateProductIndexAsync();
// await searchService.SearchProductsAsync("Red");

```

<br>

## Migration Strategies

<br>

| Strategy                      | Key Characteristics                                                                                                                                                                  | Best For                                         |
| :---------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :----------------------------------------------- |
| **1. Recreate and Rehydrate** | • Create new Azure Managed Redis instance<br/>• Update application configuration<br/>• Delete old instance<br/>• Simplest approach for look-aside caches                             | Scenarios where data loss is acceptable          |
| **2. RDB Snapshot Migration** | • Export RDB snapshot from source<br/>• Upload to Azure Storage<br/>• Import into Azure Managed Redis<br/>• For Premium tier caches only<br/>• Requires same or higher Redis version | Data preservation with minimal downtime          |
| **3. Dual-Write Pattern**     | • Write to both old and new caches<br/>• Continue reading from old cache<br/>• Switch reads to new cache when populated<br/>• Delete old cache<br/>• Application changes required    | Zero downtime migrations                         |
| **4. RIOT-X Tool**            | • Redis Input/Output Tool<br/>• Full control and customization<br/>• Open-source solution<br/>• Requires development effort                                                          | Custom migration scenarios requiring flexibility |
| **5. Active Geo-Replication** | • Set up replication between instances<br/>• Gradual traffic migration<br/>• Eventual deletion of old instance                                                                       | Geo-distributed applications                     |

<br>

## Migration Considerations

<br>

| Category                       | Key Considerations                                                                                                                                           |
| :----------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Network Configuration**      | • Private endpoint parity required<br/>• VNet peering setup<br/>• DNS and firewall rules update<br/>• Connection string changes                              |
| **Clustering Differences**     | • OSS vs Enterprise vs Non-Clustered policies<br/>• Client library compatibility<br/>• Multi-key command limitations<br/>• CROSSSLOT exception handling      |
| **Client Application Updates** | • Connection configuration only (typically)<br/>• No code changes for compatible libraries<br/>• Endpoint format changes<br/>• Authentication method updates |
| **Data Compatibility**         | • RDB file version support<br/>• Redis version matching<br/>• Page blob alignment (512-byte boundary)<br/>• Module compatibility                             |

---
