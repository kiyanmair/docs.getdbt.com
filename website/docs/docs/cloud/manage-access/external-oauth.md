---
title: "Set up external Oauth"
id: external-oauth
description: "Configuration instructions for dbt Cloud and external Oauth connections"
sidebar_label: "Set up external Oauth"
pagination_next: null
pagination_prev: null
---

# Set up external Oauth <Lifecycle status="beta, enteprise" />

:::note Beta feature

External OAuth for authentication is available in a limited beta. If you are interested in joining the beta, please contact your account manager. 

This feature is currently only available for the Okta identity provider and Snowflake connections. Only available to Enterprise accounts.

:::


dbt Cloud Enterprise supports [OAuth authentication](https://docs.snowflake.net/manuals/user-guide/oauth-intro.html) with external providers. When External OAuth is enabled, users can authorize their Development credentials using single sign-on (SSO) via the identity provider (IdP).  This grants users authorization to access multiple applications, including dbt Cloud, without their credentials being shared with the service. Not only does this make the process of authenticating for development environments easier on the user, it provides an additional layer of security to your dbt Cloud account. 

## Getting started

The process of setting up external Oauth will require a little bit of back-and-forth between your dbt Cloud, Okta, and Snowflake accounts, and having them open in multiple browser tabs will help speed up the configuration process:

- **dbt Cloud:** You’ll primarily be working in the **Account Settings** —> **Integrations** page. You will need [proper permission](/docs/cloud/manage-access/enterprise-permissions) to set up the integration and create the connections.
- **Snowflake:** Open a worksheet in an account that has permissions to [create a security integration](https://docs.snowflake.com/en/sql-reference/sql/create-security-integration).
- **Okta:** You’ll be working in multiple areas of the Okta account, but you can start in the **Applications** section. You will need permissions to [create an application](https://help.okta.com/en-us/content/topics/security/custom-admin-role/about-role-permissions.htm#Application_permissions) and an [authorization server](https://help.okta.com/en-us/content/topics/security/custom-admin-role/about-role-permissions.htm#Authorization_server_permissions).

If the admins that handle these products are all different people, it’s better to have them coordinating simultaneously to reduce friction.

### Snowflake commands

The following is a template for creating the Oauth configurations in the Snowflake environment:

```sql

create security integration your_integration_name
type = external_oauth
enabled = true
external_oauth_type = okta
external_oauth_issuer = ''
external_oauth_jws_keys_url = ''
external_oauth_audience_list = ('')
external_oauth_token_user_mapping_claim = 'sub'
external_oauth_snowflake_user_mapping_attribute = 'email_address'
external_oauth_any_role_mode = 'ENABLE'

```

The `external_oauth_token_user_mapping_claim` and `external_oauth_snowflake_user_mapping_attribute` can be modified based on the your organizations needs. These values point to the claim in the users’ token. In the example, Snowflake will look up the Snowflake user whose `email` matches the value in the `sub` claim. 

**Note:** The Snowflake default roles ACCOUNTADMIN, ORGADMIN, or SECURITYADMIN, are blocked from external Oauth by default and they will likely fail to authenticate. See the [Snowflake documentation](https://docs.snowflake.com/en/sql-reference/sql/create-security-integration-oauth-external) for more information. 

## 1. Initialize the dbt Cloud settings

1. In your dbt Cloud account, navigate to **Account settings** —> **Integrations**. 
2. Scroll down to **Custom integrations** and click **Add integrations**
3. Leave this window open. You can set the **Integration type** to Okta and make a note of the **Redirect URI** at the bottom of the page. Copy this to your clipboard for use in the next steps.

<Lightbox src="/img/docs/dbt-cloud/callback-uri.png" width="60%" title="Copy the callback URI at the bottom of the integration page in dbt Cloud" />

## 2. Create the Okta app

1. From the Okta dashboard, expand the **Applications** section and click **Applications.** Click the **Create app integration** button.
2. Select **OIDC** as the sign-in method and **Web applications** as the application type. Click **Next**.

<Lightbox src="/img/docs/dbt-cloud/create-okta-app.png" width="60%" title="The Okta app creation window with OIDC and Web Application selected" />

3. Give the application an appropriate name, something like “External Oauth app for dbt Cloud” that will make it easily identifiable. 
4. In the **Grant type** section, enable the **Refresh token** option.
5. Scroll down to the **Sign-in redirect URIs** option. Here, you’ll need to paste the redirect URI you gathered from dbt Cloud in step 1.3.

<Lightbox src="/img/docs/dbt-cloud/configure-okta-app.png" width="60%" title="The Okta app configuration window with the sign-in redirect URI configuredd to the dbt Cloud value" />

6. Save the app configuration. You’ll come back to it, but for now move on to the next steps. 

## 3. Create the Okta API

1. From the Okta sidebar menu, expand the **Security** section and clicl **API**. 
2. On the API screen, click **Add authorization server**. Give the authorizations server a name (a nickname for your Snowflake account would be appropriate). For the **Audience** field, copy and paste your Snowflake login URL (for example, https://abdc-ef1234.snowflakecomputing.com). Give the server an appropriate description and click **Save**.

<Lightbox src="/img/docs/dbt-cloud/create-okta-api.png" width="60%" title="The Okta API window with the Audience value set to the Snowflake URL" />

3. On the authorization server config screen, open the **Metadata URI** in a new tab. You’ll need information from this screen in later steps. 

<Lightbox src="/img/docs/dbt-cloud/metadata-uri.png" width="60%" title="The Okta API settings page with the metadata URI highlighted" />

<Lightbox src="/img/docs/dbt-cloud/metadata-example.png" width="60%" title="Sample output of the metadata URI" />

4. Click on the **Scopes** tab and **Add scope**. In the **Name** field, add `session:role-any`. (Optional) Configure **Display phrase** and **Description** and click **Create**.

<Lightbox src="/img/docs/dbt-cloud/add-api-scope.png" width="60%" title="API scope configured in the Add Scope window" />

5. Open the **Access policies** tab and click **Add policy**. Give the policy a **Name** and **Description** and set **Assign to** as **The following clients**. Start typing the name of the app you created in step 2.3 and you’ll see it autofill. Select the app and click **Create Policy**.

<Lightbox src="/img/docs/dbt-cloud/add-api-assignment.png" width="60%" title="Assignment field autofilling the value" />

6. On the **access policy** screen, click **Add rule**.

<Lightbox src="/img/docs/dbt-cloud/add-api-rule.png" width="60%" title="API Add rule button highlighted" />

7. Give the rule a descriptive name and scroll down to **token lifetimes**. Configure the **Access token lifetime is**, **Refresh token lifetime is**, and **but will expire if not used every** settings according to your organizational policies. We recommend the defaults of 1 hour and 90 days. Stricter rules increases the odds of your users having to re-authenticate. 

<Lightbox src="/img/docs/dbt-cloud/configure-token-lifetime.png" width="60%" title="Toke lifetime settings in the API rule window" />

8. Navigate back to the **Settings** tab and leave it open in your browser. You’ll need some of the information in later steps.

## 4. Create the Oauth settings in Snowflake

1. Open up a Snowflake worksheet and copy/paste the following:

```sql

create security integration your_integration_name
type = external_oauth
enabled = true
external_oauth_type = okta
external_oauth_issuer = ''
external_oauth_jws_keys_url = ''
external_oauth_audience_list = ('')
external_oauth_token_user_mapping_claim = 'sub'
external_oauth_snowflake_user_mapping_attribute = 'email_address'
external_oauth_any_role_mode = 'ENABLE'

```

2. Change `your_integration_name` to something appropriately descriptive. For example, `dev_OktaAccountNumber_okta`. Copy the `external_oauth_issuer` and `external_oauth_jws_keys_url` from the metadate URI in step 3.3. Use the same Snowflake URL that you entered in step 3.2 as the `external_oauth_audience_list`. 

Adjust the other settings as needed to meet your organizations configurations in Okta and Snowflake.

<Lightbox src="/img/docs/dbt-cloud/gather-uris.png" width="60%" title="The issuer and jws keys URIs in the metadata URL" />

3. Run the steps to create the integration in Snowflake.

## 5. Configuring the integration in dbt Cloud

1. Navigate back to the dbt Cloud **Account settings** —> **Integrations** page you were on at the beginning. It’s time to start filling out all of the fields. 
    1. `Integration name`: Give the integration a descriptive name that includes identifying information about the Okta environment so future users won’t have to guess where it belongs. 
    2. `Client ID` and `Client secrets`: Retrieve these from the Okta application page.
    <Lightbox src="/img/docs/dbt-cloud/gather-clientid-secret.png" width="60%" title="TThe client ID and secret highlighted in the Okta app" />
    3. Authorize URL and Token URL: Found in the metadata URI.
    <Lightbox src="/img/docs/dbt-cloud/gather-authorization-token-endpoints.png" width="60%" title="The authorize and token URLs highlighted in the metadata URI" />

2. **Save** the configuration

## 6. Create a new connection in dbt Cloud

1. Navigate the **Account settings** and click **Connections** from the menu. Click **Add connection**. 
2. Configure the `Account`, `Database`, and `Warehouse` as you normally would and for the `Oauth method` select the external Oauth you just created.

<Lightbox src="/img/docs/dbt-cloud/configure-new-connection.png" width="60%" title="The new configuration window in dbt Cloud with the External Oauth showing as an option" />

3. Scroll down to the **External Oauth** configurations box and select the config from the list.

<Lightbox src="/img/docs/dbt-cloud/select-oauth-config.png" width="60%" title="The new connection displayed in the External Oauth Configurations box" />

4. **Save** the connection and you have now configured External Oauth with Okta and Snowflake!