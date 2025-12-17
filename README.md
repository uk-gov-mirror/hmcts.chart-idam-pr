# chart-idam-pr

DEPRECATED - CFT IdAM changed the mechanism for handling PR urls several years ago and this chart is no longer required.

Note that the IdAM /testing-support/services endpoint that this chart uses does nothing since IdAM 8.2 (Feb 2023).
Continuing to include the chart is harmless, but the chart itself does nothing.

## Original motivation

Redirect URLs are a critical part of the OAuth2 authorization flow that IDAM implements and Reform Services use to log 
users into their applications. After a user successfully authenticates with IDAM, IDAM then redirects the user back to 
the originating application with either an authorization code in the URL. Because the redirect URL contains sensitive 
information, it is critical that IDAM does not redirect the user to arbitrary locations.

The best way to ensure the user is only directed to appropriate locations is to require IDAM admins and developers to 
register one or more redirect URLs for their services. Most services are set up with the correct list of URLs in IDAM 
Admin Console by the IDAM system owner. This works well for Production, however makes developers lives hard if they 
want to use application URLs that are not static (e.g. `https://moneyclaims.aat.platform.hmcts.net/receiver`). 

This chart is intended for managing whitelisted redirect URLs that are dynamic, e.g. PR style URLs in AKS: 
`https://cmc-citizen-frontend-pr-849.service.core-compute-preview.internal/receiver`.

The chart includes Kubernetes Jobs with Helm hooks that are run:
- post-install: whitelisted URL added to IDAM
- post-delete: whitelisted URL removed from IDAM

Both jobs use curl to send a PATCH request to IDAM:

```bash
curl -X PATCH \
  https://idam-api.aat.platform.hmcts.net/testing-support/services/Money%20Claims%20-%20Citizen \
    -H 'Accept: application/json' \
    -H 'Content-Type: application/json' \
    -d '[{
      "operation": "add",
      "field": "redirect_uri",
      "value": "https://cmc-citizen-frontend-pr-849.service.core-compute-preview.internal/receiver"
    }]'
```

## Example configuration

requirements.yaml
```yaml
dependencies:
  # other deps
  - name: idam-pr
    version: ~2.1.3
    repository: '@hmctspublic'
    tags:
      - idam-pr
```

values.preview.template.yaml
```yaml
tags:
  idam-pr: true

# other services yaml

idam-pr:
  redirect_uris:
    Money Claims - Citizen:
      - https://${SERVICE_FQDN}/receiver
```
*Notes*: 
- idam-pr.redirect_uris.service: name of the service as configured in IDAM. It is unique per service per environment and will 
match the label your service appears with in IDAM Admin Console. If you are not sure what it needs to be set to, 
you can try and find your service in the list of services in IDAM AAT by going to: https://idam-api.aat.platform.hmcts.net/services. Look for the label value that matches your service, then use that as `idam-pr.service.name`. If still unsure, please contact IDAM team at #sidam-team.
- idam-pr.redirect_uris.name.{value}: this is the application callback URL where IDAM will send back the authentication code. If you want to find out what redirect URIs are currently registered for your service in AAT, you can do it by going to https://idam-api.aat.platform.hmcts.net/agents/[service_oauth2_client_id] (you can find `service_oauth2_client_id` in the response of the get-services request above).
- SERVICE_FQDN - is injected by Jenkins and values file templated before passing to Helm.

## Multiple whitelisting URLs

You must use `releaseNameOverride` to avoid Kubernetes resource name clashes. An example is where the CCD chart uses two web ui components and both need whitelisting:

requirements.yaml
```yaml
  - name: idam-pr
    version: ~2.1.3
    repository: '@hmctspublic'
    tags:
      - ccd-idam-pr
```

values.preview.template.yaml
```yaml
tags:
  idam-pr: true

idam-pr:
  releaseNameOverride: ${SERVICE_NAME}-ccd-idam-pr
  redirect_uris:
    CCD:
    - https://case-management-web-${SERVICE_FQDN}/oauth2redirect
    CCD Admin:
    - https://admin-web-${SERVICE_FQDN}/oauth2redirect
```

## Default values.yaml

```yaml
api:
  url: https://idam-api.aat.platform.hmcts.net

web_public:
  url: https://idam-web-public.aat.platform.hmcts.net

redirect_uris:
  test-public-service:
  - http://localhost/oauth2/receiver

memoryRequests: "512Mi"
cpuRequests: "25m"
memoryLimits: "1024Mi"
cpuLimits: "2500m"
```
