## ADR 3: User Profile

### Context

Currently, OTRv3 supports previous versions by allowing the user to configure
their version policy to allow previous versions or not: when a user wants to
have a conversation with someone, they should send a query message telling which
versions they support. This message is sent as plaintext at the beginning of the
protocol and can be intercepted and changed by a MITM in a rollback attack.

With this attack, how can OTR ensure that participants will talk using the
highest version supported by both parties? OTRv4 seeks to solve this problem.

To address version rollback attacks on OTR, it was important for both parties to
exchange verified version information without compromising their participation
deniability and while keeping backwards compatibility with OTRv3.

### Decision

We keep the use of query message for OTRv3 compatibility but it will work as a
ping message.

We introduce a user profile to OTRv4 that includes:

1. Public key used for verifying a profile signature. Users must check whether
   they trust this key or not.
2. Versions supported in the form of a Query Message string
2. Expiration date of the User Profile
4. Profile signature of the above 3 parts
5. Transition Signature (optional): A signature of the profile excluding Profile
   Signatures and itself signed by the user's OTRv3 DSA key.

The user profile must be published publicly and updated before it expires. The
main reason for doing this is that the publication allows two parties to send
the signed User Profile during the DAKE. Since this signed profile is public
information, it does not damage participation deniability for the conversation.
As a side affect, it is possible for the receiver of a Query Message that
contains versions lower than four to check for a User Profile and thus detect a
downgrade attack for older versions. Requesting a user profile from the server
is not necessary for online conversations using OTRv4. It's important to note
that the absence of a user profile is not proof that a user doesn't support
OTRv4.

Although it is possible to check for a version publication, this does not stop
an attacker from spoofing responses about whether the profile exists or not. So
in the case where an attacker spoofs the Query Message to contain versions lower
than four and the response to a request for a User Profile, a version downgrade
attack is possible. On the other hand, the user profile will protect against
version rollback attacks in OTRv4 and higher.

If more than one valid user profile is available from the server, the one with
the latest expiry will take priority.

The signature should be generated using the Cramer-Shoup secret value `z` and a
generator of Ed448 (`G`) according to Mike Hamburg's Ed448 paper [\[1\]](#references).

We chose this curve because we are using the Ed448 in the rest of OTRv4 and
there are at least two implementations available for different platforms.

#### Protecting from rollback in OTRv4

Rollback from v4 protocol to v3 protocol can't be detected by OTRv4.
Bob's DH-Commit message does not contain a User Profile. After the AKE finishes,
Alice could contact the Profiles server and ask for Bob's User Profile to
validate if Bob really does not support 4, but this put the trust on the server.

```
Alice                        Malory                         Bob
 ?OTRv43  ---------------->   ?OTRv3  --------------------->
          <---------------------------------  DH-Commit (v3)
 The DAKE continues.
```

Rollback from vX (released after 4) protocol to v4 protocol can be
detected by OTRv4:

- For OTRvX (released after OTRv4), the known state machine versions are:
  X, ..., 4.
- After receiving a User Profile, an OTRvX client must make sure the highest
  version supported by both participants is used:
  - For every known version that's present in the received User Profile, check
    if the version used to receive this message is the highest you support.

```
Alice                               Malory                                Bob
 ?OTRvX4  ----------------------->  ?OTRv4  ---------------------------->
          <-------------------------------  Identity Message (v4)
                                            + User Profile (versions "X4")
 Detects the rollback and notifies the user. Should also abort the DAKE.
```

Notice the following case is not a rollback because "X" is not a known version
from Alice's perspective. Also notice that the list of known versions for OTRv4
is (4, 3) - and 3 does not support User Profiles. In this case, the only check
you need to perform in OTRv4 is making sure "4" is in the received User Profile.

```
Alice                               Malory                                Bob
 ?OTRv43  ----------------------->  ?OTRv4  ---------------------------->
          <-------------------------------  Identity Message (v4)
                                            + User Profile (versions "X43")
 The DAKE continues.
```

### Consequences

As OTRv4 upgrades do not fix any known security issue on OTRv3, it is acceptable
for users to chat using version 3, but is preferable to use 4 if both parties
support it.

We will support a conversation on version 3 if we don't find any user profile
and the client allows it.

Because of the decision to use the Ed448 signature algorithm, OTRv4 will use the
`z` value both for the NIZKPK (`Auth()`) in the DAKE and for the Ed448
signature. Using the same key material for two different purposes is potentially
unsafe and we aknowledge this. Nevertheless having two different keys would
increment the number of keys to be managed and would introduce the need of
binding them together.

### References

1. https://mikehamburg.com/papers/goldilocks/goldilocks.pdf "Ed448-Goldilocks, a new elliptic curve"
