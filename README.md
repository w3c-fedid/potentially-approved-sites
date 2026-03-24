# Potentially Approved Sites

TL;DR; This is a proposal to add two extra properties (`potentially_approved_site_hashes` and `site_salt`) to the FedCM [accounts endpoint](https://w3c-fedid.github.io/FedCM/#idp-api-accounts-endpoint), along with the [`approved_clients`](https://w3c-fedid.github.io/FedCM/#dom-identityprovideraccount-approved_clients) to allow IdPs to tell the browser which sites the specific acccount is connected to.

# Introduction

There is a good introduction of the space here:

https://github.com/w3c-fedid/agentic-federated-login

# Problem

The typical user journey that we want to enable is the following:

1) User is using an agentic browser, navigates to `example.com`.
2) User asks for their agentic browser for a task that involves logging in, for example "Hey browser, could you please check my orders on this website?"
3) The agentic browser is happy to assist the user and look for ways to login to `example.com`.
4) The agentic browser finds the login page (using browser actuation) and runs into the login options
5) The agentic browser checks if there is a password form and adds that as a structured login option
6) The agentic browser checks if there is a pending FedCM passive call, if the IdP says that the user already has an account on this website, via the [`approved_clients`](https://w3c-fedid.github.io/FedCM/#dom-identityprovideraccount-approved_clients) property of accounts, the agentic browser adds that specific account as a structured login option (since the user asked to "login" rather than "create an account")
7) The agentic browser prompts the user for permission to login to the website and offers the mechanisms available (the username/password options as well as the federated accounts that are available)
8) Upon user account selection, use the authentication mechanism that the user selected (e.g. fill a form with the password, resolve the FedCM passive mode request or click on the "Sign-in with X button")

The problem is that, in step (4) and (6), looking at pending FedCM passive calls doesn't cover enough ground. Even though almost [3% of page loads](https://chromestatus.com/metrics/feature/timeline/popularity/4166), the majority of ways in which users use federation on the web isn't accounted for yet.

For example, it is common that, in step (4) the website offers what's called server-to-server flows: social login "Sign-in with X" buttons that are opaque to the browser.

The problem is: in step (4), if a FedCM passive mode call isn't made in step (6), how does the agentic browser offer to the user the option to login to the website with their federated account?

# Background

Identity Providers tell the browser when their users are logging in and out with the [Login Status API](https://github.com/w3c-fedid/login-status), so the browser is aware of the federated accounts that are available.

In each federated [accounts](https://w3c-fedid.github.io/FedCM/#idp-api-accounts-endpoint), the browser is also able to access the [`approved_clients`](https://w3c-fedid.github.io/FedCM/#dom-identityprovideraccount-approved_clients) property of each account, which represents which website (by [`clientId`](https://w3c-fedid.github.io/FedCM/#dom-identityproviderconfig-clientid)) the user has already approved a connection before.

However, because of how OAuth and SAML is constructed (to support native apps), `clientIds` aren't `origin`s, they represent "clients".

When an agentic browser is helping a user in a browser tab, the website is loaded as an URL, but there is no correspondence between URLs (say, `https://pinterest.com`) and clientIds (say, `1234567890-abc123def456.apps.googleusercontent.com`).

Because of that, the browser can't test if the URL that it is on is in the `approved_clients` lists or not: the browser doesn't know how to compute `clientId` given a URL.

> As we said earlier, when the website calls into the FedCM API call on page load, it reveals what its clientId is, and so makes this whole process easier. But as also said earlier, not every website in the world call into the FedCM API.

# Alternatives Considered

## Ask Websites to expose their `clientId`

The first alternative that is worth noting is that none of this would be a problem if the website told the browser what their `clientId` was, say, in a `<meta>` tag or `.well-known` file somewhere. We ruled that out quickly because there are hundreds of thousands of websites, and asking each one of them one by one would be insurmountable.

## Deploy using JS SDKs

There is a variation of the last alternative that is a bit more plausible which is to rely on JS SDKs that IdPs control on the website's page. We think this might be a viable option (e.g. via the [`<login>`](https://github.com/fedidcg/login-element) element), but won't cover all of the ground, so insufficient.

## Fake click the "Sign-in with X" buttons to compute `clientId`

The second alternative that we considered was to have the agentic browser "click" on the "Sign-in with X" button and follow that navigation. Because the navigation is well formed (e.g. OAuth or SAML), the browser would be able to figure out what the `clientId` was by parsing the URL query parameters.

This is challenging because the agent has to (a) find the "Sign-in with X" button and (b) click it. Step (a) isn't that hard, but step (b) is hard to get right: it has to be done in a different browser context (say, an incognito window), because otherwise it can have an actual effect on the user journey.

# Proposal

None of the alternatives we considered were great. 

This proposal isn't perfect isn't, and we think that the right long term solution is the [`<login>`](https://github.com/fedidcg/login-element).

However, because we can't rely on redeploying every website in the world, this proposal has the property that we expect it to work (suboptimally) for every website.

The proposal is to introduce a `potentially_approved_site_hashes` and `site_salt` to the [accounts endpoint](https://w3c-fedid.github.io/FedCM/#idp-api-accounts-endpoint) to allow the IdP to return potentially approved sites, rather than `client_id`s.

For example:

```json
{
  "accounts": [{
    "id": "...",
    "name": "...",
    "email": "...",
    "picture": "...",
    "approved_clients": ["..."],
    "potentially_approved_site_hashes": ["..."]
  }],
  "site_salt": "..."
}
```

When the browser receives these accounts, it now is able to test whether the user has an account on a site or not, and hence make an offer to the user in the account chooser.

Unfortunately, we believe that IdPs don't have a direct 1:1 map between `clientId` and `origin` either, so we expect `potentially_approved_site_hashes` to have both false positives (sites that are in this list that the user doesn't have an account on) and false negatives (sites that the user has an account on but isn't in the list).

This isn't great, but together with the [IdP-Initiated FedCM](https://github.com/fedidcg/idp-initiated), we believe that we can make the browser handle both of those unfortunate cases gracefully:

- For false positives, the user goes through account choosing and then goes through a browser-mediated FedCM flow (through the mechanism described in [IdP-Initiated FedCM](https://github.com/fedidcg/idp-initiated)), which allows the agentic browser to give control to the user if they selected an account that was supposed to be a returning one but isn't
- For false negatives, the user would simply not see an account in the account chooser (causing a nuisance, but not a direct harm)

We don't know yet, quantifiably, the rate of false positives and false negatives, so while we think this is theoretically possible, we don't know how often it will happen. As we learn more from our production experience we'll adjust how we think about alternatives.

We fully acknowledge that this proposal isn't neither elegant nor ideal, but we think it is necessary to retrofit agentic browsers to the current web.

We are developing the [`<login>`](https://github.com/fedidcg/login-element) hoping it will be a more sustainable long term end state.
