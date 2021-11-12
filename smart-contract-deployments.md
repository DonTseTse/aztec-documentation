# Aztec smart contract deployments

Before diving deeper, a few clarifications:
- the smart-contract deployment scripts use the standards of the Truffle toolkit
- the ACE's note registry behavior relies on an external "factory" contract of which there are 2 types: a base factory adapted for convertible zk assets and an adjustable factory for adjustable zk assets. Factory contracts are upgradeable and have been timestamped by the protocol authors. To handle the upgradability and the fact that a single ACE may be used by both convertible and adjustable zk assets, there's a way to "route" interactions with the note registry to the right factory (details shown in the deployment script comments)
- each of the joinsplit, mint, etc. operations is associated to a numeric value, like an ID, which allows the ACE to route incoming proofs to the right proof validator contract

## Convertible zk asset
Supposes a ERC20 compliant contract, called `ERC20.sol`. 
```js
const ACE = artifacts.require('./ACE.sol');
const JoinSplitProofValidator = artifacts.require('./JoinSplit.sol');
const BaseFactory = artifacts.require('./noteRegistry/epochs/201907/base/FactoryBase201907');
const ZkAsset = artifacts.require('./ZkAsset.sol');
const ERC20 = artifacts.require('./ERC20.sol');

const utils = require('@aztec/dev-utils');
const bn128 = require('@aztec/bn128');

const {proofs: {JOIN_SPLIT_PROOF}} = utils;

module.exports = async (deployer, network) => {
    await deployer.deploy(ACE);
    await deployer.deploy(JoinSplitProofValidator);
    let ace = await ACE.deployed();
    await deployer.deploy(BaseFactory, ace.address);
    await ace.setCommonReferenceString(bn128.CRS);
    let joinSplitProofValidator = await JoinSplitProofValidator.deployed();
    ace.setProof(JOIN_SPLIT_PROOF, joinSplitProofValidator.address);
    let baseFactory = await BaseFactory.deployed();
    ace.setFactory(compressFactorySettings(1, 1, 1), baseFactory.address);
    // compressFactorySettings: see function below 
    await deployer.deploy(ERC20);
    let erc20 = await ERC20.deployed();
    await deployer.deploy(ZkAsset, ace.address, erc20.address, 1);     // 3rd constructor parameter is the scaling factor
};

// - epochId: counter that may be incremented to upgrade a factory
// - cryptoSystemId: may be used to run several proof validator sets within the same epoch
// - assetType: a convention which defines convertible zk asset = 1, adjustable zk asset = 2
// The 256 powers allow to shift the epoch ID and crypto system ID to the correct bits
function compressFactorySettings(epochId, cryptoSystemId, assetType){
    return epochId * 256 ** 2 + cryptoSystemId * 256 + assetType;
}
```
This migration consumes gas amounting to a bit less than 0.2 Eth (ERC20 deployment excluded). 

## Adjustable zk asset
```js
const ACE = artifacts.require('./ACE.sol');
const JoinSplitProofValidator = artifacts.require('./JoinSplit.sol');
const JoinSplitFluidProofValidator = artifacts.require('./JoinSplitFluid.sol');
const AdjustableFactory = artifacts.require('./noteRegistry/epochs/201907/adjustable/FactoryAdjustable201907.sol');
const ZkAssetAdjustable = artifacts.require('./ZkAssetAdjustable.sol');

const utils = require('@aztec/dev-utils');
const bn128 = require('@aztec/bn128');

const {proofs: {JOIN_SPLIT_PROOF, MINT_PROOF, BURN_PROOF}} = utils;

module.exports = async (deployer, network) => {
    await deployer.deploy(ACE);
    await deployer.deploy(JoinSplitProofValidator);
    await deployer.deploy(JoinSplitFluidProofValidator);
    let ace = await ACE.deployed();
    await deployer.deploy(AdjustableFactory, ace.address);
    await ace.setCommonReferenceString(bn128.CRS);
    let joinSplitProofValidator = await JoinSplitProofValidator.deployed();
    let joinSplitFluidProofValidator = await JoinSplitFluidProofValidator.deployed();
    ace.setProof(JOIN_SPLIT_PROOF, joinSplitProofValidator.address);
    ace.setProof(MINT_PROOF, joinSplitFluidProofValidator.address);
    ace.setProof(BURN_PROOF, joinSplitFluidProofValidator.address);
    let adjustableFactory = await AdjustableFactory.deployed();
    ace.setFactory(compressFactorySettings(1, 1, 2), adjustableFactory.address);
    // compressFactorySettings: see function and parameter documentation in the example above 
    await deployer.deploy(ZkAssetAdjustable, ace.address, '0x0000000000000000000000000000000000000000', 1, 0, []);
    // the ZkAssetAdjustable constructor 2nd to 5th parameter mean there's no linked ERC20, a scaling factor of 1, no initial mint proof (0) nor mint details ([])
};
```
This migration consumes gas amounting a bit below 0.24 Eth. 