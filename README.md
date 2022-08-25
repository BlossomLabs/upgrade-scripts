# Upgrade Scripts

Scripts to automate keeping track of active deployments and upgrades. Allows for:
- automatic contract deployments and proxy upgrades if the source has changed
- keeping track of all latest deployments and having one set-up for unit-tests, deployments and interactions
- storage layout compatibility checks on upgrades

These scripts use [ERC1967Proxy](https://github.com/0xPhaze/UDS/blob/master/src/proxy/ERC1967Proxy.sol).

## Example Usage using Anvil

First, make sure [Foundry](https://book.getfoundry.sh) is installed.

Clone the repository:
```sh
git clone https://github.com/0xPhaze/upgrade-scripts
```

Navigate to the example directory and install the dependencies
```sh
cd upgrade-scripts/example
forge install
```

Spin up a local anvil node **in a second terminal**.
```sh
anvil
```

Read through [deploy.s.sol](./example/script/deploy.s.sol) before running random scripts from the internet using `--ffi`.

Run
```sh
UPGRADE_SCRIPTS_DRY_RUN=true forge script deploy --rpc-url http://127.0.0.1:8545 --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 -vvvv --ffi
```
in the example project root
to go through a "dry-run" of the deploy scripts and make sure it runs correctly.
This connects to your running anvil node using the default account's private key.

Add `--broadcast` and `--ffi` to the command to actually broadcast the transactions on-chain.
```sh
forge script deploy --rpc-url http://127.0.0.1:8545 --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 -vvvv --broadcast --ffi
```

After a successful run, it should have created the file `./example/deployments/31337/deploy-latest.json` which keeps track of your up-to-date deployments. It also saves the contracts *creation code hash* and its *storage layout*.

Try running the command again. 
It will detect that no implementation has changed and thus not create any new transactions.

## Upgrading a Proxy Implementation

If any registered contracts' implementation changes, this should be detected and the corresponding proxies should automatically get updated on another call.
Try changing the implementation by, for example, uncommenting the line in `tokenURI()` in [ExampleNFT.sol](./example/src/ExampleNFT.sol) and re-running the script.

```solidity
contract ExampleNFT {
    ...
    function tokenURI(uint256 id) public view override returns (string memory uri) {
        // uri = "abcd";
    }
}
```

After a successful upgrade, running the script once more will not broadcast any additional transactions.

## Detecting Storage Layout Changes

A main security-feature of these scripts is to detect storage-layout changes.
Try uncommenting the following line in [ExampleNFT.sol](./example/src/ExampleNFT.sol).

```solidity
contract ExampleNFT is UUPSUpgrade, ERC721UDS, OwnableUDS {
    // uint256 public contractId = 1;
    ...
}
```

This adds an extra variable `contractId` to the storage of `ExampleNFT`.
If the script is run again (note that `--ffi` needs to be enabled),
it should notify that a storage layout change has been detected:
```diff
  Storage layout compatibility check [0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0 <-> 0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9]: fail
  
Diff:
  [...]

  If you believe the storage layout is compatible,
  add `if (block.chainid == 31337) isUpgradeSafe[0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0][0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9] = true;` to `run()` in your deploy script.
```

Note that, this can easily lead to false-positives, for example, when any variable is renamed
or when, like in this case, a variable is appended correctly to the end of existing storage.
Thus any positive detection here requires manually review.

Another peculiarity to account for is that, since dry-run uses `vm.prank` instead of `vm.broadcast`, there might be some differences when calculating the addresses of newly deployed contracts. Thus, sometimes, the scripts need to be run without a dry-run to get the correct address to be marked as "upgrade-safe".

Since we know it is safe, we can add the line
```solidity
if (block.chainid == 31337) isUpgradeSafe[0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0][0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9] = true;
```
to the start of `run()` in [deploy.s.sol](./example/script/deploy.s.sol).
If we re-run the script now, it will deploy a new implementation, perform the upgrade for our proxy and update the contract addresses in `deploy-latest.json`.

## Extra Notes

### Running on Mainnet
If not running on a testnet, adding `CONFIRM_DEPLOYMENT=true CONFIRM_UPGRADE=true forge ...` might be necessary. This is an additional safety measure. 

### Testing with Upgrade Scripts

In order to keep the deployment as close to the testing environment, 
it is generally helpful to share the same contract set-up scripts.

To disable any additional checks or logs that are not necessary when running `forge test`,
the function `__upgrade_scripts_init()` can be overridden to
include `__UPGRADE_SCRIPTS_BYPASS = true;`. This can be seen in [ExampleNFT.t.sol](./example/test/ExampleNFT.t.sol).
This bypasses all checks and simply deploys the contracts.

### Interacting with Deployed Contracts

To be able to interact with deployed contracts, the existing contracts can
be "attached" to the current environment (instead of re-deploying).
An example of how this can be done in order to mint an NFT from a deployed
address is shown in [mint.s.sol](./example/script/mint.s.sol). 
This requires the previous steps to be completed.

The script can then be run via:
```sh
forge script mint --rpc-url http://127.0.0.1:8545 --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 -vvvv --broadcast
```

### Contract Storage Layout Incompatibility Example

This is an example of what a contract storage layout change could look like:

```diff
"t_enum(DISTRICT_STATE)40287": {                                     |     "t_enum(DISTRICT_STATE)40450": {
"t_enum(Gang)40280": {                                               |     "t_enum(Gang)40443": {
"t_enum(PLAYER_STATE)40295": {                                       |     "t_enum(PLAYER_STATE)40458": {
"t_mapping(t_enum(Gang)40280,t_mapping(t_uint256,t_uint256))": {     |     "t_mapping(t_enum(Gang)40443,t_mapping(t_uint256,t_uint256))": {
  "key": "t_enum(Gang)40280",                                        |       "key": "t_enum(Gang)40443",
                                                                     >     "t_mapping(t_uint256,t_bool)": {
                                                                     >       "encoding": "mapping",
                                                                     >       "key": "t_uint256",
                                                                     >       "label": "mapping(uint256 => bool)",
                                                                     >       "numberOfBytes": "32",
                                                                     >       "value": "t_bool"
                                                                     >     },
"t_mapping(t_uint256,t_struct(District)40351_storage)": {            |     "t_mapping(t_uint256,t_struct(District)40514_storage)": {
  "value": "t_struct(District)40351_storage"                         |       "value": "t_struct(District)40514_storage"
"t_mapping(t_uint256,t_struct(Gangster)40314_storage)": {            |     "t_mapping(t_uint256,t_struct(Gangster)40477_storage)": {
  "value": "t_struct(Gangster)40314_storage"                         |       "value": "t_struct(Gangster)40477_storage"
```

Here, an additional `mapping(uint256 => bool)` (right side) was inserted in a storage struct at an invalid slot, which would have led to storage conflicts.

## Notes and disclaimers
These scripts do not replace manual review and caution must be taken when upgrading contracts
in any case.
Make sure you understand what the scripts are doing. I am not responsible for any damages created.

Note that, it currently is not possible to detect whether `--broadcast` is enabled.
Thus the script can't reliably detect whether the transactions are only simulated or sent
on-chain. For that reason, `UPGRADE_SCRIPT_DRY_RUN=true` must ALWAYS be set when `--broadcast` is passed in.
Otherwise this will update `deploy-latest.json` with addresses that haven't actually been deployed yet and will complain on the next run.

When `deploy-latest.json` was updated with incorrect addresses for this reason, just delete the file and the incorrect previously created `deploy-{latestTimestamp}.json` (containing the highest latest timestamp) and copy the correct `.json` (second highest timestamp) to `deploy-latest.json`.

If anvil is restarted, these deployments will also be invalid.
Simply delete the corresponding folder `rm -rf deployments/31337` in this case.