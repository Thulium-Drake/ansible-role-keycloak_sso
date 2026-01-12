# Keycloak SSO
This role provides a means to install Keycloak on your system and manage it's configuration as code.

It will:
  * Install and configure Keycloak with a PostgreSQL database
  * Install and configure a Apache based reverse proxy
  * Configure Keycloak aspects:
    * Realms
    * User federations
    * Realm roles, optionally map them against a _single_ IdM group (which can be mapped via the User Federation)
    * Clients
      * Client roles, optionally mapped against a _single_ IdM group

This role has been developed with an environment in mind that uses Red Hat Identity Management (IdM) or FreeIPA that serves as the centralized IdM solution. It might also be useable in scenarios with e.g. AD, but this role is not tested in such a setting.

## Podman
This role only sets up a minimal environment for Podman, if you need to configure registries etc,
you could use the `thulium-drake.podman` role, which provides a means to do so.

## Configuration
In order to make a configuration for the different aspects in keycloak, please refer to the Keycloak module documentation in `community.general`. Below are some snippets that can serve as an example:

  * Realms
```
keycloak_realms:
  - name: 'lab-sso'
    display_name: 'Lab SSO'
    enabled: true
    state: 'present'
```

  * User federations:
```
keycloak_user_federations:
  - name: 'lab-idm'
    realm: 'lab-sso'
    provider_id: 'ldap'
    config:
      # General settings
      editMode: 'READ_ONLY'
      importEnabled: true
      vendor: 'rhds'
      allowKerberosAuthentication: true
      trustEmail: true
      # LDAP settings
      connectionUrl: "ldaps://{ ipa_server }}:636"
      usersDn: "{{ ipa_user_full_dn }}"
      authType: 'simple'
      bindDn: "{{ ipa_readonly_user_dn }}"
      bindCredential: "{{ ipa_readonly_password }}"
      userObjectClasses: 'person, organizationalPerson'
      rdnLDAPAttribute: 'uid'
      uuidLDAPAttribute: 'ipaUniqueID'
      usernameLDAPAttribute: 'uid'
      # Kerberos settings
      kerberosRealm: "{{ ipaclient_domain | upper }}"
      serverPrincipal: "HTTP/{{ keycloak_service_hostname }}@{{ ipaclient_domain | upper }}"
      keyTab: '/opt/keycloak/keycloak.keytab'  # The role will configure the keytab at this location
      updateProfileFirstLogin: true
      krbPrincipalAttribute: 'krbPrincipalName'
    mappers:
      - name: 'username'
        providerId: 'user-attribute-ldap-mapper'
        config:
          ldap.attribute: 'uid'
          user.model.attribute: 'username'
          read.only: 'true'
          always.read.value.from.ldap: 'false'
          is.mandatory.in.ldap: 'true'
      - name: 'first name'
        providerId: 'user-attribute-ldap-mapper'
        config:
          ldap.attribute: 'givenName'
          user.model.attribute: 'firstName'
          read.only: 'true'
          always.read.value.from.ldap: 'true'
          is.mandatory.in.ldap: 'false'
      - name: 'last name'
        providerId: 'user-attribute-ldap-mapper'
        config:
          ldap.attribute: 'sn'
          user.model.attribute: 'lastName'
          read.only: 'true'
          always.read.value.from.ldap: 'true'
          is.mandatory.in.ldap: 'false'
      - name: 'email'
        providerId: 'user-attribute-ldap-mapper'
        config:
          ldap.attribute: 'mail'
          user.model.attribute: 'email'
          read.only: 'true'
          always.read.value.from.ldap: 'false'
          is.mandatory.in.ldap: 'false'
      - name: 'groups'
        providerId: 'group-ldap-mapper'
        config:
          mode: 'READ_ONLY'
          drop.non.existing.groups.during.sync: true
          groups.dn: "{{ ipa_group_full_dn }}"
          group.name.ldap.attribute: 'cn'
          group.object.classes: 'ipausergroup'
          groups.path: '/'
          ignore.missing.groups: false
          membership.attribute.type: 'DN'
          membership.ldap.attribute: 'member'
          membership.user.ldap.attribute: 'uid'
          preserve.group.inheritance: true
          user.roles.retrieve.strategy: 'LOAD_GROUPS_BY_MEMBER_ATTRIBUTE'
```

* Realm roles
```
keycloak_realm_roles:
  - name: 'client-manager'
    realm: 'lab-sso'
    description: 'Allows editing clients and retrieving client secrets'
    composite: true
    idm_group: 'keycloak_client_manager'
    composites:
      - name: 'create-client'
        client_id: 'realm-management'
      - name: 'view-clients'
        client_id: 'realm-management'
      - name: 'manage-clients'
        client_id: 'realm-management'
      - name: 'query-clients'
        client_id: 'realm-management'
    state: 'present'
```

* Clients
```
keycloak_clients:
  - name: 'forgejo-git'
    realm: 'lab-sso'
    description: 'Lab Git'
    root_url: 'https://git.example.nl'
    base_url: 'https://git.example.nl'
    protocol: 'openid-connect'
    public_client: false
    redirect_uris: 'https://git.example.nl/user/oauth2/lab-sso/callback'
    roles:
      - name: 'forgejo-access'
        idm_group: 'forgejo-access'
      - name: 'forgejo-admins'
        idm_group: 'forgejo-admins'
```
