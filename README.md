# Keycloak SSO
This role provides a means to install Keycloak on your system and manage it's configuration as code.

It will:
  * Install and configure Keycloak with a PostgreSQL database
  * Install and configure a Apache based reverse proxy

## Podman
This role only sets up a minimal environment for Podman, if you need to configure registries etc,
you could use the `thulium-drake.podman` role, which provides a means to do so.
