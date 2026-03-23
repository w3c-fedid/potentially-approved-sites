# Potentially Approved Sites

# Background

https://github.com/samuelgoto/agentic-federated-login

# Challenge

  - it is important for agentic browsers to tell the difference between signing-up vs signing-in
  - sign-up vs sign-in requires knowing what the `client_id` is
  - `client_id` isn't available on page load

# Proposal

  - Introduce a `potentially_approved_site_hashes` and `site_salt` to the [accounts endpoint](https://w3c-fedid.github.io/FedCM/#idp-api-accounts-endpoint) that allows the IdP to return potentially approved sites, rather than `client_id`s.
  - Can contain false positives and false negatives (hence the same prefixed with `potentially_`)

Example:

```json
{
  "approved_clients" : [...],
  "potentially_approved_site_hashes": [...],
  "site_salt": "...",
}
```

