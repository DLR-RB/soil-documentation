# Security

## Component Authentication

- Provided by the transport layer.
- Could also be implemented over `SOIL` communication (write nonces/signatures to specific paths on connect)

## Message authentication

- Handled on the transport layer
- No defined format to be handled by `SOIL` (message encoding is not standardized)
- Storage of keys and other data may be based on `SOIL`

## Access control

Can be handled by `SOIL`, see the [access control documentation](Access%20control.md).
Note that if the data tree must be restricted appropriately if authentication information is also stored there.

## Denial of service

There are currently no dedicated provisions to mitigate denial-of-service attacks on SOIL components.
Suitable logic may be implemented for clients, which should not change the general principles of operation.

## Timing attacks

Timing attacks can be used to infer system state via repeated requests measuring differences in response times.
The main mantra of [SOIL access control](Access%20control.md) is "Restricted areas should be invisible to the client".
Depending on how this is implemented, clients that perform timing analysis may infer additional information about restricted information, e.g. the existence of additional paths.
This may be due to tree traversal and lookups that take longer for existing paths.
Susceptability to these attacks is implementation-dependent.

## Security vulnerabilities

As with any software (concept), there may be security flaws present in both the concept, as well as specific implementations.
`SOIL` aims to minimize these issues by:
- being as simple as possible
- never treating data as code
- performing strict validations on inputs
- providing good documentation to minimize implementation errors
