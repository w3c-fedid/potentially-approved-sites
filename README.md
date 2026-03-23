- Introduction and context: https://github.com/samuelgoto/agentic-federated-login
- Goal: provide a federated login tool for agentic browsers
- Challenge:
    - it is important for agentic browsers to tell the difference between signing-up vs signing-in
    - sign-up vs sign-in requires knowing what the `client_id` is
    - `client_id` isn't available on page load
- Proposal
    - Introduce `potentially_approved_site_hashes` and `site_salt` that allows the IdP to return potentially approved sites, rather than `client_id`s.
    - Can contain false positives and false negatives

```json
{
  "approved_clients" : [...],
  "potentially_approved_site_hashes": [...],
  "site_salt": "...",
}
```

