# Titicloud Kubernetes Infrastructure

## Servarr Authentication Quirk
When using ForwardAuth (Authentik) with Prowlarr, Sonarr, or Radarr, environment variables (e.g., `PROWLARR__AUTH__METHOD`) may be ignored if a `config.xml` file already exists. 

To ensure ForwardAuth works correctly, an **init container** is used to patch the `config.xml` file at startup:
- Sets `AuthenticationMethod` to `External`.
- Sets `AuthenticationRequired` to `DisabledForLocalAddresses`.

**IMPORTANT NOTE FOR FRESH INSTALLS**: 
The `initContainer` runs *before* the application starts. On the very first run, `config.xml` does not exist yet, so the patch cannot be applied. The application will start and create a default `config.xml` with Basic Auth. 
**You MUST restart the pod once (`kubectl rollout restart deployment <app> -n media`) after the initial deployment** so the `initContainer` can run again and successfully patch the now-existing config file.

Refer to `values/media/*.yaml.gotmpl` for implementation details.
