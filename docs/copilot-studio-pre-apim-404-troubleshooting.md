# Copilot Studio Pre-APIM 404 Troubleshooting

This guide covers intermittent connection prompts and `404` errors that occur before a
request reaches API Management or the App Service adapter.

## Confirmed deployment context

- The adapter is deployed to Azure App Service.
- API Management fronts the deployed App Service.
- Dev Tunnel is not part of this request path and is excluded from this diagnosis.
- Connector consent-card bypass is enabled for the Copilot Studio agent.
- During the reported failures, no corresponding request appears in APIM or App Service
  logs.

The expected specialist call is:

```text
Copilot Studio orchestrator
  -> A2A connector and user connection
  -> APIM specialist operation
  -> App Service adapter
  -> Copilot Studio specialist
```

If APIM receives nothing, the failure is before the APIM boundary:

```text
Copilot Studio orchestrator
  -> connector, operation, or connection resolution fails
  X no outbound HTTP request
```

## What consent-card bypass changes

Consent-card bypass suppresses the conversational card normally shown when a connector
is first used on behalf of a user. It does not:

- Create a connection for every user.
- Repair a `Not Connected`, `Expired`, `Stale`, or deactivated connection.
- Replace a deleted connection or connection reference.
- Bypass Microsoft Entra Conditional Access or MFA.
- Repair a connector client secret or OAuth configuration.
- Update a connector operation cached by an already published agent.
- Fix a connected-agent reference that points to another environment or an old agent.

Admin consent can remove delegated permission approval, but a delegated call still needs
a valid user connection and a resolvable connector operation.

## Most likely causes

### 1. Stale connection reference

Copilot Studio keeps separate records for the custom connector, the agent's connection
reference, and each user's authenticated connection. The agent can continue to reference
an old internal connection ID after a connector or connection is recreated.

This commonly follows:

- Deleting and recreating a connection.
- Recreating or reimporting a custom connector.
- Importing the agent into another Power Platform environment.
- Replacing or upgrading a solution.
- Changing connector authentication.
- Switching between generic OAuth and the experimental Entra OBO configuration.

Power Platform can return an internal `404` while resolving that record. No HTTP request
is sent to APIM.

### 2. User connection is missing or unusable

Microsoft documents connection states including `Connected`, `Not Connected`,
`Deactivated`, `Expired`, and `Stale`. The maker can have a valid connection while a
non-maker user has no usable connection.

An expired or stale connection can explain both symptoms:

1. Power Platform cannot resolve or refresh the connection.
2. Copilot Studio presents a connection experience even though the connector consent
   card itself is bypassed.

### 3. Wrong connector or environment

Two connectors can have the same display name but different internal IDs. The published
agent might reference a connector or connection from another environment, solution
version, or import.

Verify the environment ID, agent ID, connector ID, connection reference ID, and published
agent version rather than relying on display names.

### 4. Stale connector operation

The agent can retain an old operation reference after the connector operation was deleted,
renamed, or recreated. Power Platform then fails before it has an HTTP operation to invoke.

This is especially likely when the operation has the same display name but a different
internal `operationId`.

### 5. OBO conversion was reverted

The native A2A connected-agent UI does not expose the repository's experimental Entra OBO
conversion. Editing or saving the connected agent or its generated connector can revert
the connector to generic OAuth.

Download the effective connector after every relevant edit and verify:

- Authentication provider.
- `IsOnbehalfofLoginSupported`.
- `ResourceUri`.
- Client ID and delegated scope.
- Host, base path, operation path, and `operationId`.

Authentication failures normally produce `401` or `403`, but a missing connection or
connector record can fail earlier as an internal platform `404`.

### 6. Invalid connected-agent reference

The orchestrator can retain an invalid reference if the specialist or connected-agent
definition was deleted, recreated, unpublished, moved to another environment, or removed
and added again. The orchestrator then fails while resolving the tool, before calling its
connector.

### 7. Specialist tool was not selected

No APIM request is expected when the orchestrator decides that the specialist is not
needed. Test with a prompt that unambiguously requires the specialist. This explanation is
less likely when the conversation contains an explicit `connectorRequestFailure`.

## Distinguish an internal and external 404

Inspect the complete error, including the failing URL or resource identifier.

| Evidence | Interpretation |
| --- | --- |
| Error identifies the expected APIM hostname | Power Platform attempted an external request. Confirm that this exact APIM instance and time range are being monitored. |
| Error identifies a different hostname | The effective connector points somewhere else. |
| Error refers to `connections`, `connectors`, `apis`, `environments`, `bots`, or `operations` | Power Platform could not resolve one of its internal resources; APIM is not expected to receive a request. |
| No error and no APIM request | The orchestrator might not have selected the specialist tool. |

## Investigation procedure

Use one affected non-maker user, one new conversation, and one exact UTC timestamp.

1. In the published orchestrator, open **Settings** > **Connection Settings**.
2. Inspect the A2A/custom connector entry and record its status, connector, connection
   owner, and environment.
3. Confirm that the affected user's connection is `Connected`; do not use the maker's
   connection as proof.
4. Open the connected specialist/tool configuration and record the selected specialist,
   connector, operation, and connection reference.
5. Test the custom connector operation independently using the same connection.
6. Confirm whether that independent test produces an APIM request.
7. Download the effective connector and compare its host, base path, operation path,
   `operationId`, authentication provider, `ResourceUri`, and OBO setting with the
   intended configuration.
8. Verify that the orchestrator and specialist are published and shared with the affected
   user in the same Power Platform environment.
9. Save and republish the orchestrator only after the connector and connection are known
   to be correct.
10. Start a new conversation and send a prompt that must invoke the specialist.

## How to interpret the result

| Result | Likely fault | Next action |
| --- | --- | --- |
| Connection is `Not Connected`, `Expired`, or `Stale` | User connection lifecycle | Reauthenticate or select a known-good connection, then republish and retest. |
| Independent connector test does not reach APIM | Connector or connection configuration | Inspect the connector host, operation, authentication, and connection reference. |
| Independent connector test reaches APIM, but the orchestrator does not | Agent tool or connected-agent binding | Rebind the current connector operation and connection in the orchestrator, then republish. |
| Downloaded connector shows generic OAuth | Experimental OBO conversion was reverted | Restore and verify the Entra OBO configuration before republishing. |
| Error contains an internal connection or operation resource | Stale Power Platform record | Bind a new or known-good connection/operation; remove the stale object only after validation. |
| APIM receives the request | The failure is no longer pre-APIM | Continue with APIM operation, App Service route, token, and downstream diagnostics. |

## Safe recovery sequence

Do not delete or redeploy APIM for a failure that never reaches APIM.

1. Prove that the current connector operation works independently.
2. Create or select a known-good user connection.
3. Rebind the orchestrator tool to that connection and the current connector operation.
4. Save and publish the orchestrator.
5. Test in a new conversation as a non-maker user.
6. Remove stale connections or references only after the replacement works.

## References

- [Microsoft Learn: Create and manage connections](https://learn.microsoft.com/microsoft-copilot-studio/authoring-connections)
- [Microsoft Learn: Bypass connector consent cards for an agent](https://learn.microsoft.com/microsoft-copilot-studio/admin-connector-consent-bypass)
- [Repository connection setup and OBO notes](./copilot-studio-a2a-connections.md)
- [Authentication and OBO sequence diagrams](./auth-obo-sequence-diagram.md)
