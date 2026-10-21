# Client API

This section defines the client API.

## Description

The client API is the API that will be used by the SDK to interact with the network.
A client defines a concrete network connection to a specific network with a specific operator account.

### External signing

A `TransactionSigner` signs transactions with a key the SDK does not hold — a cloud KMS (AWS KMS,
Google Cloud KMS, Azure Key Vault), HashiCorp Vault transit, a PKCS#11 HSM, or a hardware wallet.
Signing with such a key is I/O, so `signTransaction` is `@@async`. One key contributes one
signature per target node (see `consensusnode.transactions`), so the SDK calls the signer once per
`NodeBody`.

Every `TransactionSigner` implementation must follow this contract:

- **Input.** `body.bytes` is the serialized `TransactionBody` addressed to `body.node`, exactly as
  returned by `PackedTransaction.signableBodies()`. The signer signs these bytes and nothing else.
- **Digest.** The signer applies the digest of its key algorithm itself: Ed25519 signs the raw body
  bytes; ECDSA secp256k1 signs the Keccak-256 digest of the body bytes. Backends that cannot
  compute Keccak-256 (no cloud KMS offers it) receive the 32-byte digest computed locally by the
  signer.
- **Output encoding.** `NodeSignature.signature` is 64 bytes for both algorithms. For ECDSA
  secp256k1 that is `r || s` (32 bytes each, big-endian) with `s` in the lower half of the curve
  order; DER-encoded or high-`s` signatures returned by a backend must be converted by the signer.
- **Identity.** `NodeSignature.node` equals `body.node`, and `NodeSignature.publicKey` equals the
  signer's `publicKey`.

## API Schema

```
namespace consensusnode.client
requires {AccountId, Network} from ledger
requires {NetworkSetting} from ledger.config
requires {PrivateKey, PublicKey} from keys
requires {NativeTokenUnit} from nativeToken

// Definition of an account that signs and pays for requests
Account {
    @@immutable accountId: AccountId // the account id of the operator
    @@immutable privateKey: PrivateKey // the private key of the operator
}

// A serialized TransactionBody for one target consensus node — the exact bytes a signer
// must sign. Bytes differ per node because each body carries the target node's nodeAccountID.
type NodeBody {
    @@immutable node: AccountId
    @@immutable bytes: bytes
}

type NodeSignature {
      @@immutable node: AccountId       // the consensus node's fee account
      @@immutable publicKey: PublicKey
      @@immutable signature: bytes      // 64 bytes; see "External signing" for the encoding
}

// Signs transactions with a key the SDK does not hold (KMS, HSM, Vault, hardware wallet).
// Implementations must follow the contract in "External signing" above.
abstraction TransactionSigner {

  @@immutable publicKey: PublicKey // the public key matching every signature this signer produces

  // Signs the TransactionBody addressed to body.node. Called once per target node.
  @@async NodeSignature signTransaction(body: NodeBody)
}

// Common base for anything that is handed off to the consensus node network and
// produces a typed result — both queries (consensusnode.queries.Query) and
// packed transactions (consensusnode.transactions.PackedTransaction) extend this.
// Carries the shared retry-tuning knobs (max attempts, backoff window, per-attempt
// timeout) and the single submit() entry point. The SDK applies these settings
// when selecting a consensus node, retrying transient gRPC failures, and bounding
// the total wait.
abstraction Submittable<$$Result> {

    @@nullable maxAttempts: int32
    @@nullable maxBackoff: int64
    @@nullable minBackoff: int64
    @@nullable attemptTimeout: int64

    // Hand off to the network and return the typed result. Node selection,
    // retry, and any operation-specific protocol details (e.g. cost discovery
    // and payment for PaidQuery) are handled by the SDK transparently.
    @@async $$Result submit(client: HieroClient)
}

// The client API that will be used by the SDK to interact with the network
HieroClient<$$Unit extends NativeTokenUnit> {
    @@immutable operatorAccount: Account // the operator account
    @@immutable network: Network<$$Unit> // the network to connect to
    @@immutable transactionSigner: TransactionSigner // by default the operator account is used, but this allows to use an external signer for transactions
    // TO_BE_DEFINED_IN_FUTURE_VERSIONS
}

// factory methods of `HieroClient` that should be added to the namespace in the best language dependent way

@@static HieroClient<ANY> createClient(networkSettings: NetworkSetting, operatorAccount: Account)
@@static HieroClient<ANY> createClient(networkSettings: NetworkSetting, operatorAccount: Account, transactionSigner: TransactionSigner)
```

## Examples

The following example shows how to create a `HieroClient` instance:

```
AccountId accountId = ...;
PrivateKey privateKey = ...;
Account operatorAccount = new Account(accountId, privateKey);

NetworkSetting networkSettings = ...;

HieroClient client = HieroClient.createClient(networkSettings, operatorAccount);
```

The following example shows a `TransactionSigner` backed by an ECDSA secp256k1 key in a cloud KMS:

```
KmsSigner implements TransactionSigner {
    publicKey = createPublicKey(SPKI_WITH_DER, kms.getPublicKey(keyId))  // fetched once

    @@async NodeSignature signTransaction(body: NodeBody) {
        bytes digest = keccak256(body.bytes)                 // KMS cannot hash with Keccak-256
        bytes der = await kms.sign(keyId, digest, DIGEST)    // one KMS call per target node
        return new NodeSignature(body.node, publicKey, toCompactLowS(der))
    }
}
```

## Questions & Comments

- [@Jexsie](https://github.com/Jexsie): The `TransactionSigner` contract above was validated with a
  PoC that implements this API on top of the V2 JavaScript SDK, with AWS KMS and HashiCorp Vault
  signers (ported from [hiero-remote-signer](https://github.com/Jexsie/hiero-remote-signer)). Open
  points it surfaced:
  - **`Account` still requires a `PrivateKey`.** An operator whose key is in a KMS has no
    `PrivateKey`, so callers need both an `Account` and a separate `transactionSigner`. The PoC
    models `Account { accountId, signer: TransactionSigner }`, where an in-memory key is one
    `TransactionSigner` implementation. That removes `HieroClient.transactionSigner`, the
    three-argument `createClient`, and the `sign(payerId, signer, nodes)` /
    `signForBatch(payerId, signer, batchKey)` tiers. Same problem as
    [missing-features.md §6.1](../../missing-features.md); probably needs an ADR, since it also
    touches ADR-0001 and ADR-0002.
  - **Signing cost scales with the number of target nodes.** A remote signer makes one billed,
    rate-limited, audit-logged backend call per target node and key. Should `HieroClient` own node
    selection (callers of `pack(payer, nodes)` currently have no API to obtain `nodes`), and should
    `TransactionSigner` offer an optional batch method such as
    `@@async list<NodeSignature> signTransactions(bodies: list<NodeBody>)`?
  - **Lifecycle.** A PKCS#11 signer holds a session that must be closed. Should
    `TransactionSigner` be closeable, or is that left to implementations?
  - **Wire format.** Out-of-process signing ships `NodeBody` values to a signing service and
    `NodeSignature` values back, but neither has a defined serialization (unlike
    `PackedTransaction.toBytes()`), so every pipeline invents its own.
