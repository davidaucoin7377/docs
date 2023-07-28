# OnChain Credential Status

On-chain identity verification requires the ability to check whether a claim has been revoked using a smart contract. To accomplish this, the claim must include information about the smart contract, such as its address, blockchain, and network. This information can be stored in the `CredentialStatus` structure.

The `CredentialStatus` structure should contain the following fields:

```golang
type CredentialStatus struct {
	ID                 string               `json:"id"`
	Type               CredentialStatusType `json:"type"`
	RevocationNonce    uint64               `json:"revocationNonce"`
}
```

The `ID` field is a composite field that contains encoded information in the following format:

`did:[did-method]:[blockchain]:[network]:[id]/credentialStatus?(revocationNonce=value)&(contractAddress={chainID}:{contractAddress})`

`type` - credential type

`revocationNonce` - unique revocation nonce

Example:

```json
{
"id": "did:polygonid:polygon:main:2qCU58EJgrEMWhziKqC3qNXJkZPY8XCxDSBM4mqPkM/credentialStatus?revocationNonce=1234&contractAddress=1:0xf3bB959314B5D1e4587e1f597ccc289216608ac5",
"type": "Iden3OnchainSparseMerkleTreeProof2023",
"revocationNonce": "1234"
}
```

The `revocationNonce` and `contractAddress` parameters inside the ID field are optional. Here's an example without them:

```json
{
"id": "did:polygonid:polygon:main:2qCU58EJgrEMWhziKqC3qNXJkZPY8XCxDSBM4mqPkM/credentialStatus",
"type": "Iden3OnchainSparseMerkleTreeProof2023",
"revocationNonce": "1234"
}
```

## Process ID field

In the context of `OnChainCredentialStatus`, the `id` field can contain two additional optional parameters: `revocationNonce` and `contractAddress`.
`revocationNonce` is a credential revocation `nonce`, while `contractAddress` is the address of a smart contract that implements an on-chain issuer interface. The `contractAddress` field is composed by two parts: `chainID` and `contractAddress`. Use `chainID` to select the correct network.

If `contractAddress` is not set, find the default contract address by parsing DID [extracting the on-chain issuer contract address](https://github.com/iden3/go-iden3-core/blob/014f51e92da5c0c89c95c31e42bfca1652d2ad14/did.go#L345-L354) and getting `chainID` from the DID network. Use the blockchain name, network and contract address from the DID to make on-chain revocation request. If the DID doesn't have a contract address inside and `contractAddress` parameter is empty, this VC document should be considered invalid.

If `revocationNonce` is not set, the `revocationNonce` value from the struct will be used instead.

## Workflow

Example of how to build a `non-revocation` proof with the `Iden3OnchainSparseMerkleTreeProof2023` credential status type:

1. Extract the `credentialStatus` object from the verifiable credential.
1. Parse core.DID from credentialStatus.id field with [js](https://github.com/iden3/js-iden3-core/blob/baa0ead8a3e2340bb4d78132ec63e6e24d806da9/src/did/did.ts#L160)|[go](https://github.com/iden3/go-iden3-core/blob/014f51e92da5c0c89c95c31e42bfca1652d2ad14/w3c/did_w3c.go#L165)
1. Extract core.Id fome core.DID with [js](https://github.com/iden3/js-iden3-core/blob/baa0ead8a3e2340bb4d78132ec63e6e24d806da9/src/did/did.ts#L160)|[go](https://github.com/iden3/go-iden3-core/blob/014f51e92da5c0c89c95c31e42bfca1652d2ad14/did.go#L184)
1. Use the DID from step two to extract the on-chain issuer contract address:

    a. If the `contractAddress` parameter is not empty, use this address to build the non-revocation proof.

    b. If the `contractAddress` is empty, extract the contract address from the `id` field (refer to [this code snippet](https://github.com/iden3/go-iden3-core/blob/014f51e92da5c0c89c95c31e42bfca1652d2ad14/did.go#L345-L354)).

    c. If the `id` doesn't have the `contractAddress` parameter, and you are not allowed to extract the contract address from the `DID`, consider this VC document invalid.

1. Extract `chainID` from `contractAddress` parameter. If `chainID` does not exist - try to extract `chainID` from DID. If both empty - return an error.
1. Parse the `id` to obtain the `revocationNonce`:

    a. You can extract the `revocationNonce` from the `id` parameter `revocationNonce`.

    b. If the `id` doesn't have the `revocationNonce`, you can get the `revocationNonce` from the `revocationNonce` field.

    c. If the parameter doesn't exist and the `revocationNonce` field is empty, consider this VC document invalid.

1. Generate revocation proof call method `getRevocationStatus` from the issuer smart contract using the information you received earlier.
    
```golang 
const response = await this.onchainContract.getRevocationStatus(id, nonce);  
```
    
- Use this ABI to make getRevocationStatus call.
    
```json
[
    {
      "inputs": [
        {
          "internalType": "uint256",
          "name": "id",
          "type": "uint256"
        },
        {
          "internalType": "uint64",
          "name": "nonce",
          "type": "uint64"
        }
      ],
      "name": "getRevocationStatus",
      "outputs": [
        {
          "components": [
            {
              "components": [
                {
                  "internalType": "uint256",
                  "name": "state",
                  "type": "uint256"
                },
                {
                  "internalType": "uint256",
                  "name": "claimsTreeRoot",
                  "type": "uint256"
                },
                {
                  "internalType": "uint256",
                  "name": "revocationTreeRoot",
                  "type": "uint256"
                },
                {
                  "internalType": "uint256",
                  "name": "rootOfRoots",
                  "type": "uint256"
                }
              ],
              "internalType": "struct IOnchainCredentialStatusResolver.IdentityStateRoots",
              "name": "issuer",
              "type": "tuple"
            },
            {
              "components": [
                {
                  "internalType": "uint256",
                  "name": "root",
                  "type": "uint256"
                },
                {
                  "internalType": "bool",
                  "name": "existence",
                  "type": "bool"
                },
                {
                  "internalType": "uint256[]",
                  "name": "siblings",
                  "type": "uint256[]"
                },
                {
                  "internalType": "uint256",
                  "name": "index",
                  "type": "uint256"
                },
                {
                  "internalType": "uint256",
                  "name": "value",
                  "type": "uint256"
                },
                {
                  "internalType": "bool",
                  "name": "auxExistence",
                  "type": "bool"
                },
                {
                  "internalType": "uint256",
                  "name": "auxIndex",
                  "type": "uint256"
                },
                {
                  "internalType": "uint256",
                  "name": "auxValue",
                  "type": "uint256"
                }
              ],
              "internalType": "struct IOnchainCredentialStatusResolver.Proof",
              "name": "mtp",
              "type": "tuple"
            }
          ],
          "internalType": "struct IOnchainCredentialStatusResolver.CredentialStatus",
          "name": "",
          "type": "tuple"
        }
      ],
      "stateMutability": "view",
      "type": "function"
    }
  ]
```
  
Also, you can use the signature of getRevocationStauts `0xeb62ed0e` instead of the ABI.

