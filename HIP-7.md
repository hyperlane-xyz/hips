| hip | title                                           | status | author     | created    |
| --- | ----------------------------------------------- | ------ | ---------- | ---------- |
| 7   | Draft multisig transactions on long-tail chains | Draft  | aroralanuk | 2024-01-10 |

### **Brief Summary / Abstract**

> "Uh, open up the safe, bitches got a lot to say" - Iggy Azalea

### **Motivation**

When deploying Hyperlane to a long-tail chain, one often runs into the question of ownership of critical roles like proxyAdmin owner, mailbox owner or warp route owner. On fat head chains, the Gnosis Safe SDK's governor tooling is present to support multisig operations but their lack of supprt on the long-tail means we need to deploy the safe SDK adhoc as we're doing the bridge deployment. This is the not the best experience from the deployer perspective and risky as we may need to blind sign and manually submit the transactions.

### **Definition**

- Chain A: chain where the multisig tooling is adequately supported (like Ethereum Mainnet) which has n multisig participants with public keys pk_1, pk_2, pk_3, ..., pk_n and aggregation public key K_agg . The relevant contracts are mailbox_A, proxyAdmin_A, hypToken_A, etc or referred to ownable_A generally.
- Chain B: chain where the multisig tooling isn't supported like HarryPotterObamaSonicInu chain which the core hyperlane contract deployments. The relevant contracts are mailbox_B, proxyAdmin_B, hypToken_B, etc or referred to ownable_B generally.

### **Goal**

- minimize f(x), f(x) = Σ||sig_i(AUTH_CALL) - sig_0(AUTH_CALL)||_0, ie minimize the number of actions or signatures taking place outside your home chain.
- simplicity, minimizing the number of additional deployment and maintainence burden
- (nice to have) 
    - subject to equal soundness property: if the multsig participants cannot sign on chain A, then they shouldn't be able to sign on chain B either.
    - liveness: ∀ T ∈ valid transactions, ∃ subset S' of S such that |S'| >= n and ∀ signer ∈ S', eventually signer signs T, then T is eventually executed.

**Stakeholders**

- Internal operations for transferring ownership, upgrading contracts, etc.
- Long-tail chain teams like Manta, HarryPotterObamaSonicInu chains you would like to have ownable without the hassle.

### **Options**

#### Use `InterchainAccount`

Procedure

- We deploy ICARouter_A on chain A and ICARouter_B on chain B. We then deploy a `InterchainAccount` on chain B with the following parameters:
  - origin: chainA, owner: k_agg, router: ICARouter_B, ism: ISM_B
- The ownership of ownable_B will be transferred to K_agg.localICA() after being instantiated on ICARouter_B.handle()
- K_agg will call ICARouter_A.callRemote({(dest_B, callParams_b),(dest_C, callParams_C)}) with batch of transactions which will be delivered to chain B, chain C, etc with their respective callParams.
  - For sending multiple destinations in the same callRemote call, we'll need the adapt the callRemote to loop through the destinations and send the transactions one by one.
- Sign the address is derived from ISM_B, changing the ISM will mean ownership change of all subseqeunt ownable_B contracts
- ISM_B cannot contain PausableISM as it will be impossible to unpause the contract
- A malicious majority of signers can sign the transfer ownership of ownable_B to a malicious address (security is dependent on the ISM_B which will multisig on chain B)
  - If we can alleviate this by using the native bridges or zkLightClient like zeth

Pros:

- Easier lift, need to update the ICADeployer and add it to the CLI (with some minor changes to the contract to support msg.value) and needs no relayer changes
- Good example of dogfooding our own middleware and experience the pain points first hand

Cons:

- Security is dependent on the ISM_B which may be sufficient for Abacus Works but not for other projects who don't want to use and maintain their validator set but don't want to rely on AW's ISM. Further, if the validator set is down or not synced, the multisig transactions will be stuck in the queue.

#### Use `SafeISM`

- The multisig signers pk_1, pk_2, pk_3, ..., pk_n will serve as the signers for the SafeISM on chain B and we can queue the transactions and check the ecdsa signatures in the SafeISM contract. The recipient will be ICA account which will call the intended governable function on the ownable_B contract (msg.sender is ICA_B).
- Each signer key pk_i will sign the ({B, ICA_B.call()[]}, {C, ICA_B.call()[]}) and submit it to Hyperlane transaction services which the queries by the SafeISM which is CCIPReadIsm and uses the following:

  - msg: {B,ICA_B.call()[]}, metadata: ({C, ICA_B.call()[]}, signatures: {sig_1, sig_2, sig_3, ..., sig_t})
  - compute digest H(msg, metadata) and pk_i == ecrecover(digest, sig_i)

- If the multsig participant pk_i is not EOA, we need EIP1271 compatible ICA_B_i account authorized to sign messages on behalf of pk_i.

Pros:

- Only relies on the security of the multisig signer set and not an additional validator set.

Cons:

- Needs an additional offchain Hyperlane Safe Services which will store the signatures for the multisig calls and gets the per-destination-chain message and metadata when fetched from the respective ISM through CCIP read interface.

#### Use AA-style solutions
(Privy, Pimlico, ZeroDev, Capsule, Portal, etc)

Uses shamir secret sharing to split the private key of the multisig participants and use the threshold signature scheme to sign the transactions. The threshold signature scheme will be implemented in the SSSISM contract.

- Most of the AA wallet style solutions are ERC-4337 compatible which has a lot of bells and whistles which we don't need like GasPaymaster API, Bundle API, etc.
- For access control, it's meant to be more hierarchical like 2/3 but the two are the two different devices and a third is a backup from their servers which is an additional dependency. Not ideal for our use since it may be stored in iframe.

### **Scope**

- Contract changes (ICA)
- ICA deployer (sdk+cli)
- checker/governor queue transactions for the safe 

### **Further questions**

- Handling non-EVM environments