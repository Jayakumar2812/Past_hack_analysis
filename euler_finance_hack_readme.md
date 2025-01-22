# Euler Finance Hack: Expanded Overview of the Protocol
**Euler Finance** is a decentralized lending protocol designed to provide users with advanced features, enhanced capital efficiency, and risk management tools. It operates on Ethereum and supports a wide variety of tokens, enabling users to lend, borrow, and earn interest in a fully decentralized and permissionless environment.

## Key Features

### Permissionless Listing
- Unlike traditional lending platforms, Euler allows any ERC-20 token to be listed as long as it meets minimum criteria.
- Tokens are categorized into three tiers based on their risk levels:
  - **Collateral Tier**: Can be used as collateral for borrowing.
  - **Cross Tier**: Can be borrowed but not used as collateral.
  - **Isolation Tier**: Restricted to lending only.

### Risk Management
- **Risk-Adjusted Lending and Borrowing**:
  - Uses a risk framework to determine collateral requirements and borrowing capacity for each token.
- **Interest Rate Models**:
  - Dynamic interest rates based on utilization rates, with custom rate curves for each asset.
- **Liquidation Incentives**:
  - Introduces a Dutch auction mechanism for liquidations to minimize the risk of cascading failures.

### Uniswap Integration
- Euler integrates with Uniswap V3 to provide on-chain price feeds for token valuations, enabling real-time risk assessment.

### Tokenized Debt
- Borrowed positions are represented as ERC-20 tokens, making them composable with other DeFi protocols.

### Sophisticated Lending Features
- **Leverage**:
  - Users can leverage their positions by borrowing tokens and reinvesting them in the protocol.
- **Short Selling**:
  - Euler allows short selling by borrowing tokens and selling them on external platforms.

### Governance and Community
- Governed by a decentralized autonomous organization (DAO) where EUL token holders can vote on protocol changes and updates.

### Efficiency
- Gas-optimized architecture to reduce costs for users compared to traditional lending protocols.

## Workflow of Euler Protocol

### 1. Lending
- Users deposit ERC-20 tokens into the protocol to earn interest.
- Interest is generated from borrowers who pay fees based on dynamic interest rates.

### 2. Borrowing
- Borrowers provide collateral to take out loans.
- The collateralization ratio depends on the token's risk tier and the interest rate model.
- Borrowers receive tokenized debt positions (ERC-20 tokens) representing their loan.

### 3. Risk Isolation
- High-risk assets are isolated to prevent systemic risks.
- Tokens in the **Isolation Tier** are restricted to lending only, ensuring they do not impact other assets in the protocol.

### 4. Liquidations
- If a borrower's collateral falls below the required threshold, their position becomes liquidatable.
- Liquidators participate in a **Dutch auction mechanism** to repay the borrower's debt and claim their collateral at a discounted price.

### 5. Leverage and Short Selling
- Users can leverage their positions by borrowing additional tokens and reinvesting them in the protocol.
- Short selling is facilitated by borrowing tokens and selling them on external platforms, enabling users to profit from price declines.

### 6. Governance
- EUL token holders participate in governance to propose and vote on protocol updates.
- Governance decisions include risk parameter adjustments, new token listings, and upgrades to the protocol.


## Detailed Mechanics of Euler Protocol

### Deposits and ETokens
- When users deposit assets into the Euler protocol, they receive **ETokens** representing their collateral.
- As the deposited tokens earn interest, the value of the ETokens increases compared to the underlying asset.
- Almost any asset can be deposited on Euler, provided a Uniswap V3 pool exists.

### Layered Leverage
- Users can mint up to **10 times their collateral value** in ETokens to gain leverage, but this also mints **DTokens** representing their debt.
- Euler allows users to use loans directly as collateral, enabling **layered leverage**:
  - Newly minted ETokens can be used as collateral to borrow additional assets.
  - This amplifies potential profits but also increases risk by reducing the health score.

### Overcollateralized Loans
- All loans are technically overcollateralized to ensure the borrower can repay the loan plus interest.

### Health Score and Liquidations
- The **health score** determines how close a borrower is to liquidation:
  
  
  **Health Score = Max LTV / Current LTV**


- Euler's Maximum LTV values:
  - **Regular Loans**: 75% (collateral token and loaned token are distinct).
  - **Self-Collateralized Loans**: 95% (collateral and loaned tokens are the same).

- If a borrower's health score falls below **1**, they can be liquidated.

### Soft Liquidations
- Unlike fixed penalties, Euler employs **soft liquidation**:
  - The penalty starts at **0%** and increases by **1% for every 0.1 decrease** in the health score.
  - The maximum penalty is **20%**.
- Liquidators can repay the borrower's debt and seize their collateral at a discount equal to the penalty percentage.

## Euler Finance Attack Steps

## Attacker's Initial Balance
- **Attacker balance before exploit**: 0 DAI

## Steps of the Attack

### 1. Flash Loan Acquisition
- Borrowed **30 million DAI** using a flash loan.

### 2. Deployment of Attack Contracts
The attacker deployed two contracts:
- **The Violator**: To perform the attack using the flash loan.
- **The Liquidator**: To liquidate the Violator’s account.

---

### Using the Violator Contract

### 3. Initial Deposit
- Deposited **20 million DAI** to Euler using the `EToken::deposit` function.
- Received approximately **19.5 million ETokens** representing the collateral.

#### After Deposit (Violator):
- **Collateral (eDAI)**: 19,568,124.414447288391400399
- **Debt (dDAI)**: 0

### 4. Borrowing ETokens and DTokens
- Borrowed **195.6 million ETokens** and **200 million DTokens** using the `EToken::mint` function.
- The borrowed-to-collateral ratio resulted in an **LTV of 93%** (maximum allowed was 95% for self-collateralized loans) with a health score of **1.02**.

#### After Minting (Violator):
- **Collateral (eDAI)**: 215,249,368.558920172305404396
- **Debt (dDAI)**: 200,000,000
- **Health Score**: 1.040263157894736842

### 5. Partial Debt Repayment
- Repaid **10 million DAI** to Euler using the `DToken::repay` function.
- Approximately **10 million ETokens** were burned, leaving the EToken balance unchanged.
- This decreased the debt compared to the collateral, improving the health score.

#### After Repayment (Violator):
- **Collateral (eDAI)**: 215,249,368.558920172305404396
- **Debt (dDAI)**: 190,000,000
- **Health Score**: 1.089473684210526315

### 6. Additional Borrowing
- Borrowed **195.6 million eDAI** and **200 million ETokens** again using the `EToken::mint` function.
- This decreased the health score but allowed the attacker to increase the borrowed amount, maximizing their profits.

#### After Minting (Violator):
- **Collateral (eDAI)**: 410,930,612.703393056219408393
- **Debt (dDAI)**: 390,000,000
- **Health Score**: 1.020647773279352226

### 7. Donation to Reserves
- Donated **100 million EToken** to Euler using the `EToken::donateToReserves` function.

#### Vulnerability:
- The `donateToReserves()` function did not perform liquidity checks to ensure that the user’s health score remained above **1** after the donation.

    ```
    /// @notice Donate eTokens to the reserves
    /// @param subAccountId 0 for primary, 1-255 for a sub-account
    /// @param amount In internal book-keeping units (as returned from balanceOf).
    function donateToReserves(uint subAccountId, uint amount) external nonReentrant {
    (address underlying, AssetStorage storage assetStorage, address proxyAddr, address msgSender) = CALLER();
    address account = getSubAccount(msgSender, subAccountId);

    updateAverageLiquidity(account);
    emit RequestDonate(account, amount);

    AssetCache memory assetCache = loadAssetCache(underlying, assetStorage);

    uint origBalance = assetStorage.users[account].balance;
    uint newBalance;

    if (amount == type(uint).max) {
        amount = origBalance;
        newBalance = 0;
    } else {
        require(origBalance >= amount, "e/insufficient-balance");
        unchecked { newBalance = origBalance - amount; }
    }

    assetStorage.users[account].balance = encodeAmount(newBalance);
    assetStorage.reserveBalance = assetCache.reserveBalance = encodeSmallAmount(assetCache.reserveBalance + amount);

    emit Withdraw(assetCache.underlying, account, amount);
    emitViaProxy_Transfer(proxyAddr, account, address(0), amount);

    logAssetStatus(assetCache);
    }
    ```
### Using the Liquidator Contract

### 8. Liquidation Check
- The Liquidator contract checks the liquidation using the `Liquidation::checkLiquidation` function to determine the yield and repayment values (corresponding to the collateral and debt transferred to the liquidator).

#### Key Mechanism:
- The `computeLiqOpp()` function ensures that the amount of EToken sent to the liquidator does not exceed the borrower’s available collateral.
- If the collateral does not satisfy the expected repayment yield, the remaining collateral is used by default.
- Liquidators only ever incur debt equal to the collateral they acquire at the discount dictated by the soft-liquidation mechanism.

#### Assumption:
- This mechanism assumes that the violator’s collateral is never lower than their debt, a condition that can fail due to the donation exploit.

```
    // Limit yield to borrower's available collateral, and reduce repay if necessary
    // This can happen when borrower has multiple collaterals and seizing all of this one won't bring the violator back to solvency

    liqOpp.yield = liqOpp.repay * liqOpp.conversionRate / 1e18; 

    {
        uint collateralBalance = balanceToUnderlyingAmount(collateralAssetCache, collateralAssetStorage.users[liqLocs.violator].balance);

            // if collateral < debt then collateral will be < yield 
        if (collateralBalance < liqOpp.yield) {
            liqOpp.repay = collateralBalance * 1e18 / liqOpp.conversionRate;
            liqOpp.yield = collateralBalance;
        }
    }
```
### 9. Liquidation Execution
- Liquidated the Violator using the `Liquidation::liquidate` function.
- The violator’s health score dropped below **1**, triggering the soft-liquidation mechanism.
- The yield cannot exceed the available collateral, and the discount (penalty fee of **20%**) was maintained, as enforced in `computeLiqOpp()`.
- Since the Violator's EToken balance exceeded their DToken balance:
  - The entire EToken balance was transferred to the Liquidator.
  - A portion of the DToken balance remained in the Violator’s account.
- This created bad debt in the system while ensuring the Liquidator’s profit covered their debt.

#### After Liquidating (Liquidator):
- **Collateral (eDAI)**: 310,930,612.703393056219408392
- **Debt (dDAI)**: 259,319,058.477209877830400000000000000

#### After Liquidating (Violator):
- **Collateral (eDAI)**: 0.000000000000000001
- **Debt (dDAI)**: 135,765,628.943911884480000000000000000

---

### 10. Withdrawal of Liquidated Funds
- Withdrew the liquidated funds using `EToken::withdraw`.
- The exchange rate for EToken to the underlying token was skewed due to the artificially increased total borrows in the system.
- The attacker was able to withdraw more DAI for their ETokens.
- The Liquidator burned ~38 million ETokens to withdraw the entire contract balance of ~38.9 million DAI.

#### After Withdrawing (Liquidator):
- **Collateral (eDAI)**: 272,866,200.699670845275982401
- **Debt (dDAI)**: 259,319,058.477209877830400000000000000
- **EULER balance**: 0 DAI

### 11. Flash Loan Repayment
- Repaid the flash loan using the profits:
  - **30 million DAI** for the loan.
  - **27k DAI** in interest.
- Remaining profit: **8.88 million DAI**.

#### Attacker Balance After Exploit:
- **8,877,507 DAI**.

### 12. Repeated Attack
- Repeated the exploit on other liquidity pools, resulting in a total profit of approximately **$197 million**.
---

## Root Causes of the Attack

### 1. Lack of Health Check After Donation
- The `donateToReserves` function failed to check whether the user was in a state of undercollateralization after donating funds to the reserve.
- This oversight allowed the soft-liquidation mechanism to be triggered.
- Combined with self-collateralized layered leverage, the attacker could self-liquidate with the maximum penalty of **20%**.

### 2. Liquidator’s Profit Exceeded Their Debt
- The Violator’s health score dropped below **1** when the soft-liquidation logic was triggered due to their EToken balance exceeding their DToken balance post-donation.
- The `computeLiqOpp()` function ensured that:
  - The Liquidator’s health score remained above **1**.
  - The **20% discount** on the ETokens was maintained.
- This resulted in bad debt being locked in the Violator contract, while the Liquidator’s profit entirely covered their debt.
- The Liquidator could extract the obtained funds without requiring additional collateral.

### Result
This combination of factors enabled the attacker to drain the contract’s funds and profit significantly.

---

## Recommendations to Prevent Similar Hacks

## 1. Invariant Testing
- This attack could have been avoided if the **health score** was tested post-donation.
- The core invariant is that the health score should never go below **1** unless the value of the underlying changes.
- New logic and functions added to an existing codebase, such as the `donateToReserves()` function, should be thoroughly tested in the context of the entire protocol.


## 2. Comprehensive Auditing
- Multiple firms had previously audited Euler Finance; however, the `donateToReserves()` function was only audited once.
- The protocol was audited again after the code change, but the function was **out of scope**.
- Comprehensive audits are crucial to ensure that modifications to the protocol do not create vulnerabilities in the context of the entire system.
- The `donateToReserves()` function was never considered when used in the context of lenders, only for the use case where the EToken reserves needed to be increased.



## run POC for nomad hack 
   ```
      forge test --match-path test/Eulerfinance.sol
   ```