# Identity Access Management - POC

## Purpose

POC to provide an authentication mechanism to manage users logging into to 3rd party applications that required authentication and authorization services.

## Run Locally

```bash
source local.env && docker-compose -f docker-compose.yml -f docker-compose.test.yml build
source local.env && docker-compose -f docker-compose.yml -f docker-compose.test.yml up
```

Optional - Create Secrets if you want to `docker stack deploy`.

```
docker secret create auth_keycloak_user ./keycloak_user_file.txt
docker secret create auth_keycloak_password ./keycloak_password_file.txt
docker secret create auth_keycloak_postgres_password ./keycloak_postgres_password.txt
```

## Using phpLDAPadmin
The container exposes port 80 as target so find the actual port docker has mapped to it by using the following command and then finding the entry for the service
```
docker service ls
```
The output from docker should be similar to the following:
```
9l0mdxp4s5xr        keyk_phpldapadmin   replicated          1/1                 osixia/phpldapadmin:0.9.0           *:30002->80/tcp
```
In this case the port is 30002 and the URL to get to the admin ui would be: http://<hostname>:30002

To login goto the published port using a browser and use the following creadentials:
1. For the Login DN: use ```cn=admin,dc=wpms,dc=test```
1. For the password: use ```password```

## Configuration

Configure KeyCloak

* Go to `http://localhost/auth`
* Open KeyCloak Admin console
* Log in with `admin` `password`
* Configure a client
  * Go to "Configure" > "Clients" on the left panel
  * Select "Create", to create a new client
  * In the "Client ID" field, enter `client1`
  * In "Client Protocol", select `openid-connect`
  * Click "Save"
* Configure a user
  * Go to "Manage" > "Users" on the left panel
  * Select "Add User"
  * In `Username` field, enter "user1"
  * Click "Save"
  * Select the "Credentials" selection header
  * In the "New Password" and "Password Confirmation" fields, enter `password`
  * Turn "Temporary" to "Off"
  * Click "Reset Password"

## Example Requests

This will get an auth token, assuming there is a client called `client` and a user `user1` with password `password`:

```bash
curl -i -XPOST -d "grant_type=password&username=user1&password=password&client_id=client1" http://localhost/auth/realms/master/protocol/openid-connect/token
```

This will make the request to the service. Traefik will pass the auth token to the auth service to verify the token, and the request will be forwarded to the internal service.

```bash
curl -i http://localhost/svc1/secure -H "Authorization: Bearer $(AUTH_TOKEN)"
```

In one fell swoop.

This example uses [jq](https://stedolan.github.io/jq/download/) to parse the JSON to extract the token.

```bash
curl -XPOST -d "grant_type=password&username=user1&password=password&client_id=client1" http://localhost/auth/realms/master/protocol/openid-connect/token |\
jq '.access_token' |\
cut -d "\"" -f 2 |\
curl -i http://localhost/svc1/secure -H "Authorization: Bearer $(cat -)"
```
