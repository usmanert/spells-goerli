# spells-goerli

![Build Status](https://github.com/makerdao/spells-goerli/actions/workflows/.github/workflows/tests.yaml/badge.svg?branch=master)

Staging repo for MakerDAO's Goerli executive spells.

## Instructions

### Getting Started

```bash
$ git clone git@github.com:makerdao/spells-goerli.git
$ dapp update
```

### Adding Collaterals to the System

If the weekly executive needs to onboard a new collateral:

1. Update the `onboardNewCollaterals()` function in `Goerli-DssSpellCollateral.sol`
2. Update the values in `src/test/config.sol`
3. Add `onboardNewCollaterals();` in the `actions()` function in `DssSpellAction`


### Collateral Onboarding

In order to onboard new collateral to the Maker protocol, the following must be done before the spell is prepared:

- Deploy a GemJoin contract
    - Rely the `MCD_PAUSE_PROXY` address
- Deploy a Clip contract
    - Rely the `MCD_PAUSE_PROXY` address
- Deploy an Abacus pricing contract
    - Initialize values (or use initialization function in DssExecLib)
    - Rely the `MCD_PAUSE_PROXY` address
- Deploy a Pip contract (oracles team-managed)
    - Rely the `MCD_PAUSE_PROXY` address

Once these actions are done, add the following code (below is an example) to the `execute()` function in the spell. The `setChangelogAddress` function calls are required to add the collateral to the on-chain changelog. They must follow the following convention:
- GEM: `TOKEN`
- JOIN: `MCD_JOIN_TOKEN`
- CLIP: `MCD_CLIP_TOKEN`
- CALC: `MCD_CLIP_CALC_TOKEN`
- PIP: `PIP_TOKEN`
- VAL: `VAL_TOKEN`

```js
pragma solidity 0.6.12;
import "dss-exec-lib/DssExec.sol";
import { CollateralOpts } from "dss-exec-lib/CollateralOpts.sol";
import "dss-interfaces/dss/IlkRegistryAbstract.sol";
import "dss-exec-lib/DssAction.sol";
uint256 internal constant TWO_FIVE_PCT_RATE   = 1000000000782997609082909351;
uint256 constant THOUSAND   = 10 ** 3;
uint256 constant MILLION    = 10 ** 6;
// --- DEPLOYED COLLATERAL ADDRESSES ---
address internal constant XMPL                 = 0x9884522d356F8bae29874035888482B35ECD0Dbd;
address internal constant VAL_XMPL             = 0xb64Aee69FBc71A1b61b331DCf7a332de507a0327;
address internal constant PIP_XMPL             = 0xB686bB63Ae3db79Ebf5CB24863d10b5f58Bd6f98;
address internal constant MCD_JOIN_XMPL_A      = 0xF6Fd400f15F995d4BF56Fe86aDb55D6a43091eBb;
address internal constant MCD_CLIP_XMPL_A      = 0x6b7Aaa71412aDBD4513AC230158c4C7Fb7E1Cb70;
address internal constant MCD_CLIP_CALC_XMPL_A = 0xad5287705Be426869Af60211cA22bB5F26425656;
address internal constant ETH_FROM = 0x125fC0CcCDee5ac474062F6358d4d056b0430b84;

// Initialize the pricing function with the appropriate initializer
CollateralOpts memory XMPL_A = CollateralOpts({
    ilk:                   "XMPL-A",
    gem:                   XMPL,
    join:                  MCD_JOIN_XMPL_A,
    clip:                  MCD_CLIP_XMPL_A,
    calc:                  MCD_CLIP_CALC_XMPL_A,
    pip:                   PIP_XMPL,
    isLiquidatable:        true,
    isOSM:                 true,
    whitelistOSM:          true,
    ilkDebtCeiling:        3 * MILLION,
    minVaultAmount:        100,         // 100 Dust
    maxLiquidationAmount:  50000,       // 50,000 Dai
    liquidationPenalty:    1300,        // 13% penalty
    ilkStabilityFee:       1000000000705562181084137268,
    startingPriceFactor:   13000,       // 1.3x multiplier
    auctionDuration:       6 hours,
    permittedDrop:         4000,        // 40% drop before reset
    liquidationRatio:      15000        // 150% collateralization ratio
});
// add new collateral from dss-exec-lib 
DssExecLib.addNewCollateral(XMPL_A);

DssExecLib.setChangelogAddress("XMPL",          0xCE4F3774620764Ea881a8F8840Cbe0F701372283);
DssExecLib.setChangelogAddress("PIP_XMPL",      0x9eb923339c24c40Bef2f4AF4961742AA7C23EF3a);
DssExecLib.setChangelogAddress("MCD_JOIN_XMPL-A", 0xa30925910067a2d9eB2a7358c017E6075F660842);
DssExecLib.setChangelogAddress("MCD_CLIP_XMPL-A", 0x9daCc11dcD0aa13386D295eAeeBBd38130897E6f);
DssExecLib.setChangelogAddress("MCD_CLIP_CALC_XMPL-A", xmpl_calc);
DssExecLib.setStairstepExponentialDecrease(MCD_CLIP_CALC_XMPL_A, 130 seconds, 9900);// args: (address calc , step , cut)
DssExecLib.setIlkAutoLineParameters("XMPL-A", 20 * MILLION, 3 * MILLION, 8 hours);// (bytes32 _ilk, uint256 _amount, uint256 _gap, uint256 _ttl)
// Updates the ilk-registry 
IlkRegistryAbstract(DssExecLib.reg()).update("XMPL-A");
// Denying the ETH_FROM Address from following contracts.
Authorizable(VAL_XMPL).deny(ETH_FROM);
Authorizable(PIP_XMPL).deny(ETH_FROM);
Authorizable(MCD_JOIN_XMPL_A).deny(ETH_FROM);
Authorizable(MCD_CLIP_XMPL_A).deny(ETH_FROM);
Authorizable(MCD_CLIP_CALC_XMPL_A).deny(ETH_FROM);

```
#### CollateralOpts:
- `ilk`:                  Collateral type
- `gem`:                  Address of collateral token
- `join`:                 Address of GemJoin contract
- `clip`:                 Address of Clip contract
- `calc`:                 Address of Abacus pricing contract
- `pip`:                  Address of Pip contract
- `isLiquidatable`:       Boolean indicating whether liquidations are enabled for collateral
- `isOsm`:                Boolean indicating whether pip address used is an OSM contract
- `whitelistOsm`:         Boolean indicating whether median is src in OSM.
- `ilkDebtCeiling`:       Debt ceiling for new collateral
- `minVaultAmount`:       Minimum DAI vault amount required for new collateral
- `maxLiquidationAmount`: Max DAI amount per vault for liquidation for new collateral
- `liquidationPenalty`:   Percent liquidation penalty for new collateral [ex. 13.5% == 1350]
- `ilkStabilityFee`:      Percent stability fee for new collateral       [ex. 4% == 1000000001243680656318820312]
- `startingPriceFactor`:  Percentage to multiply for initial auction price. [ex. 1.3x == 130% == 13000 bps]
- `auctionDuration`:      Total auction duration before reset for new collateral
- `permittedDrop`:        Percent an auction can drop before it can be reset.
- `liquidationRatio`:     Percent liquidation ratio for new collateral   [ex. 150% == 15000]

### Removing Collaterals from the System

If the weekly executive needs to offboard collaterals:

1. Update the `offboardCollaterals()` function in `Goerli-DssSpellCollateral.sol`
2. Update the values in `src/test/config.sol`
3. Add `offboardCollaterals();` in the `actions()` function in `DssSpellAction`

### Build

```bash
$ make
```

### Test (DappTools without Optimizations)

Set `ETH_RPC_URL` to a Goerli node.

```bash
$ export ETH_RPC_URL=<Goerli URL>
$ make test
```

### Test (Forge without Optimizations)

#### Prerequisites
1. [Install](https://www.rust-lang.org/tools/install) Rust.
2. [Install](https://github.com/gakonst/foundry#forge) Forge.

#### Operation
Set `ETH_RPC_URL` to a Goerli node.

```bash
$ export ETH_RPC_URL=<Goerli URL>
$ make test-forge
```

### Deploy

Set `ETH_RPC_URL` to a Goerli node and ensure `ETH_GAS` is set to a high enough number to deploy the contract.

```bash
$ export ETH_RPC_URL=<Goerli URL>
$ export ETH_GAS=8000000
$ export ETH_GAS_PRICE=$(seth --to-wei 3 "gwei")
$ make deploy
```
