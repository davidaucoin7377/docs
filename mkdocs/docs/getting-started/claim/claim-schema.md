# Credential Schema

The reusability of credentials across platforms and services is guaranteed by [Credential Schema](https://docs.iden3.io/protocol/claim-schema/) consistency. 

Polygon ID use [JSON-LD documents](https://json-ld.org/learn.html) to represent Credential Schemas.

As an issuer it is advised to check if any of the [existing credential schemas](https://github.com/0xPolygonID/schemas/tree/main/jsonld) can accommodate the type of information you are interested to issue.

If not, here's the guide to create a new credential schema. Let us create a shared and reusable credential schema of type **ProofOfDaoMembership**.

1.**Define the value to be included in the schema.**

The ProofOfDaoMembership credential should attest that a person executes a role inside a specific DAO.

Information such as the identifier of the DAO or the identifier of the subject of the credential don't need to be encoded inside one of the four data slots allocated for claim information (i_2,i_3, v_2, v_3): 

- The information about the specific DAO can be inferred from the credential issuer identifier
- The information about the individual subject of the claim is already stored in the i_1 or v_1 data slot of the Claim.

A further information that must be included in the claim is the *role* of the individual inside a DAO. This will be the added inside one of the data slots (i_2,i_3,v_2,v_3). 

Remember that a claim can only store numeric data so each DAO role should be encoded as a number.

2.**Decide where to store this information, should it be inside index data slots or value data slots?**

Claim's index determines its uniqueness inside the issuer's claims tree. There cannot be more than one claim with the same index. If it is decided to store the identifier of the subject of the claim inside i_1 and leave the other index data slots empty, it means that there can only be one claim issued to a specific identity inside the tree.

In this case, the question is whether to store the information with type *role* inside i_2 or v_2.

- Storing the role inside i_2 means that the uniqueness inside the tree is determined by the combination "person identifier + role"
- Storing the role inside v_2 means that the uniqueness inside the tree is only determined the person identifier

Considering the possibility that a DAO member covers more than one role, it makes more sense to store the role inside i_2. This choice allow the DAO to issue subsequent claims to the same individual attesting a different role.

3.**Describe the vocabulary of the schema**

Create a markdown file in your repository to describe the vocabulary used in the claim. This should contain a description of the key type *role* and its possible values:

```js
# role

Describes the role covered by an individual inside a specific DAO

1: Contributor
2: Guild Coordinator
3: Team Member
```

4.**Create the JSON-LD document**

Add a file inside your repository with extension .json-ld and populate it.

The `@id` key should contain the identifier of the Schema Type "ProofOfDaoMembership", in this case the unique url to the JSON-LD document.
The `proof-of-dao-vocab` key should contain the url that describes the vocabulary of the claim schema.

```json
{
  "@context": [{
    "@version": 1.1,
    "@protected": true,
    "id": "@id",
    "type": "@type",
    "ProofOfDaoMembership": {
      "@id": "https://raw.githubusercontent.com/iden3/tutorial-examples/main/claim-schema/proof-of-dao-membership.json-ld#ProofOfDaoMembership",
      "@context": {
        "@version": 1.1,
        "@protected": true,
        "id": "@id",
        "type": "@type",
        "proof-of-dao-vocab": "https://github.com/iden3/tutorial-examples/blob/main/claim-schema/proof-of-dao.md#",
        "serialization": "https://github.com/iden3/claim-schema-vocab/blob/main/credentials/serialization.md#",
        "type": {
          "@id": "proof-of-dao-vocab:role",
          "@type": "serialization:IndexDataSlotA"
        },
      }
    }
  }]
}
```

5.**Generate the schema hash**

The [Schema Hash](https://docs.iden3.io/protocol/claim-schema/#schema-hash) has to be added inside claim's index.

The schema hash is generated by hashing together `schemaBytes` (the JSON-LD document in byte format) and `credentialType` (in this case "ProofOfDaoMembership"). In this case:
    
```go
package main

import (
    "fmt"
    "os"

    core "github.com/iden3/go-iden3-core"
    "github.com/iden3/go-iden3-crypto/keccak256"
)

func main() {

  schemaBytes, _ := os.ReadFile("./tutorial-examples/claim-schema/proof-of-dao-membership.json-ld")

    var sHash core.SchemaHash
    h := keccak256.Hash(schemaBytes, []byte("ProofOfDaoMembership"))

    copy(sHash[:], h[len(h)-16:])

    sHashHex, _ := sHash.MarshalText()

    fmt.Println(string(sHashHex))
    // 4f6bbcb133bfd4e9ebdf09b16a0816c8
}
```

> The executable code can be found [here](https://github.com/0xPolygonID/tutorial-examples/tree/main/claim-schema)