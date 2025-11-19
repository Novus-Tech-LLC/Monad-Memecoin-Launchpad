[Back to Support index](./SUPPORT.md)

## MNad.Fun Support Guide

This document explains how to get help, diagnose common issues, and contribute safely to the MNad.Fun smart contract system.

---

## Table of Contents

- [Getting Help](#getting-help)
- [Common Issues](#common-issues)
- [Frequently Asked Questions (FAQ)](#frequently-asked-questions-faq)
- [Troubleshooting](#troubleshooting)
- [Reporting Bugs](#reporting-bugs)
- [Feature Requests](#feature-requests)
- [Contributing](#contributing)
- [Contact Information](#contact-information)
- [Additional Resources](#additional-resources)

---

## Getting Help

If you need assistance with MNad.Fun, use the following channels:

1. **Documentation**  
   Review the main documentation in [`README_EN.md`](./README_EN.md) for an architectural overview and usage details.

2. **GitHub Issues**  
   - Search existing issues to check whether your question has already been answered.
   - Open a new issue if you do not find an existing one that matches your case.

3. **GitHub Discussions**  
   - Use Discussions for open‑ended questions, design feedback, and community support.

4. **Direct Contact (for sensitive topics)**  
   - For security or private matters, use the contact methods listed in [Contact Information](#contact-information).

---

## Common Issues

### Contract Deployment Issues

**Symptom**: Contracts fail to deploy or initialize.

**Checklist**:

- Ensure your account has sufficient gas to cover deployment costs.
- Verify that all required constructor or initialization parameters are provided correctly.
- Confirm that the factory contract is **deployed and initialized** before depending contracts are used.
- Check that the deploying address has the appropriate permissions and ownership (if required).

---

### Transaction Failures

**Symptom**: Transactions revert with an error.

**Common causes**:

- Insufficient balance (WMon or token).
- Slippage constraints triggered (min/max amount not satisfied).
- Deadline parameter has expired.
- Insufficient allowance for token spending.
- Bonding curve is currently locked.

**What to check**:

- Confirm your WMon and token balances before trading.
- If using `protectBuy` / `protectSell`, relax slippage slightly and re‑test.
- Use a deadline such as `block.timestamp + 300` (5 minutes) instead of a past timestamp.
- Ensure you have called `approve()` or are using a permit-enabled function.
- Query curve state to verify whether it is locked.

---

### Fee Calculation Issues

**Symptom**: Transactions revert due to fee validation.

**Checklist**:

- Validate that the fee configuration (numerator/denominator) matches the expected bonding curve settings.
- Confirm that you are using the same fee parameters in off‑chain simulations and on‑chain calls.
- Ensure the inequality `fee >= (amount * denominator) / numerator` holds where expected.

---

### Permit / Approval Issues

**Symptom**: EIP‑2612 `permit` calls or approvals fail.

**What to verify**:

- Signature parameters (`v`, `r`, `s`) are computed correctly and not reused.
- The `deadline` in the signed data has not expired.
- The domain separator used off‑chain matches the on‑chain contract’s domain separator.
- The `nonce` matches the current on‑chain nonce for that owner and token.

---

## Frequently Asked Questions (FAQ)

### General Questions

**Q: What is WMon?**  
**A:** WMon is the wrapped version of the native Monad token, providing an ERC20‑compatible interface for trading and protocol operations.

**Q: How do bonding curves work in MNad.Fun?**  
**A:** Bonding curves use a **constant product formula** (\\(k = x \* y\\)) to determine price. As more tokens are bought, the price increases; as tokens are sold back, the price decreases. Virtual reserves are used to shape the initial pricing curve.

**Q: What happens when a bonding curve locks?**  
**A:** Once the configured lock target is reached, the curve transitions to a **locked** state. Trading against that curve stops, and the position can be migrated to a DEX via the `listing()` flow.

**Q: Can I create multiple tokens?**  
**A:** Yes. Each call to `createBc` deploys a new `Token` and `BondingCurve` pair.

**Q: How are protocol fees calculated?**  
**A:** Fees are determined by the bonding curve’s configuration (numerator/denominator). For buys, fees are deducted from the **input** amount; for sells, fees are deducted from the **output** amount.

---

### Trading Questions

**Q: What is the difference between `buy` and `protectBuy`?**  
**A:** `buy` executes immediately at the current curve price. `protectBuy` adds a minimum output parameter and reverts if the realized price would yield fewer tokens than `amountOutMin`.

**Q: Do I need to approve tokens before selling?**  
**A:** Yes. You must either:

- Approve tokens using the `approve()` function, or  
- Use permit-based functions such as `sellPermit` or `protectSellPermit` for gasless approvals.

**Q: What is the deadline parameter used for?**  
**A:** The deadline guarantees transaction freshness. It must be a future timestamp (for example, `block.timestamp + 600`). Transactions with an expired deadline revert.

**Q: Can I buy an exact amount of tokens?**  
**A:** Yes. Use `exactOutBuy` to specify the exact token amount you want to receive, and the contract computes the required WMon input.

**Q: Can I sell for an exact amount of WMon?**  
**A:** Yes. Use `exactOutSell` to specify the exact WMon output you require, and the contract computes the required token input.

---

### Technical Questions

**Q: What Solidity version do the contracts use?**  
**A:** The contracts target **Solidity ^0.8.13**.

**Q: What testing framework is used?**  
**A:** The project uses **Foundry**. See `test/README.md` for additional details and examples.

**Q: What is the difference between virtual and real reserves?**  
**A:** Virtual reserves exist purely in the pricing formula and do not map to actual balances. Real reserves are the on‑chain balances held by the bonding curve contract and must satisfy the invariant enforced by the curve logic.

**Q: What happens during DEX listing?**  
**A:** When `listing()` is called:

- WMon and tokens are transferred into a Uniswap V2–compatible pair.
- Liquidity is added according to the configured parameters.
- LP tokens are typically burned or controlled per the design, ensuring transparent liquidity.

---

## Troubleshooting

### Test Failures

If tests are failing locally:

1. **Check Foundry version**

   ```bash
   forge --version
   ```

2. **Update dependencies**

   ```bash
   forge update
   ```

3. **Clean and rebuild**

   ```bash
   forge clean
   forge build
   ```

4. **Isolate failing tests**

   ```bash
   forge test --match-test testName
   ```

5. **Increase verbosity**

   ```bash
   forge test -vvv
   ```

---

### Compilation Errors

If contracts fail to compile:

1. **Verify Solidity version**
   - Ensure your tooling is configured for `^0.8.13`.

2. **Check import paths**
   - Confirm that all `import` statements reference valid paths.

3. **Install dependencies**

   ```bash
   forge install
   ```

4. **Validate remappings**
   - Ensure `remappings.txt` reflects the correct library paths.

---

### Runtime Errors

If a transaction reverts at runtime:

1. **Read the revert message or error selector** – it is designed to be descriptive.  
2. **Inspect emitted events** – events often carry state context for the failing call.  
3. **Check contract state** – verify that preconditions (e.g., unlocked curve, sufficient reserves) hold.  
4. **Verify permissions** – ensure the caller has the appropriate role or ownership.  
5. **Confirm gas settings** – underpriced gas can cause silent failures at the node level.

---

## Reporting Bugs

When opening a bug report, please include:

1. **Description** – Clear, concise summary of the issue.  
2. **Steps to Reproduce** – Exact sequence of steps or transactions that trigger the bug.  
3. **Expected Behavior** – What you expected to happen.  
4. **Actual Behavior** – What actually occurred (including revert messages or error logs).  
5. **Environment**:
   - Solidity version  
   - Foundry version (if relevant)  
   - Network (local, testnet, mainnet, etc.)  
6. **Error Messages / Logs** – Full stack traces or transaction hashes if possible.  
7. **Minimal Reproduction** – A minimal contract or script that demonstrates the issue.  
8. **Screenshots** – Where visual context helps (e.g., block explorer screenshots).

### Bug Report Template

```markdown
## Bug Description
Brief description of the bug

## Steps to Reproduce
1. Step one
2. Step two
3. Step three

## Expected Behavior
What should happen

## Actual Behavior
What actually happens

## Environment
- Solidity Version:
- Foundry Version:
- Network:

## Error Messages
Paste error messages or transaction hashes here

## Additional Context
Any additional context, logs, or configuration details
```

---

## Feature Requests

Feature requests are welcome. To help evaluate them effectively, please include:

1. **Clear Description** – What you want to add or change.  
2. **Use Case / Motivation** – Why this feature matters (user story, business case).  
3. **Potential Implementation** – Any ideas or sketches for how it might be implemented.  
4. **Alternatives Considered** – Other approaches you explored and why they were rejected.

Well-scoped, rationale-backed proposals have a significantly higher chance of being accepted.

---

## Contributing

Contributions are highly appreciated. A typical workflow looks like:

1. Fork the repository.  
2. Create a topic or feature branch.  
3. Implement your changes with clear, focused commits.  
4. Add or update tests covering the new behavior.  
5. Run the full test suite and ensure everything passes.  
6. Open a pull request with:
   - A clear description of the change.
   - Any migration or deployment notes.

If a CONTRIBUTING guide is present in the repository, please follow it as the source of truth.

---

## Contact Information

### Say hello / Reach the maintainer

- **Whatsapp**: `https://wa.me/https://wa.me/14105015750`  
- **Telegram**: `https://t.me/novustch`  
- **Discord**: `https://discordapp.com/users/985432160498491473`  
- **Twitter**: `https://x.com/novustch`

For **security disclosures**, please use email or a direct private channel rather than opening a public issue.

---

## Additional Resources

- [`README_EN.md`](./README_EN.md) – Main technical documentation.  
- `test/README.md` – Testing documentation and examples.  
- GitHub repository – Source code and issue tracker.

---

**Security Note**  
If you believe you have found a vulnerability, please follow responsible disclosure practices and contact the maintainers privately first.

# Support Documentation

[English](./SUPPORT.md)

## Table of Contents

- [Getting Help](#getting-help)
- [Common Issues](#common-issues)
- [Frequently Asked Questions (FAQ)](#frequently-asked-questions-faq)
- [Troubleshooting](#troubleshooting)
- [Reporting Bugs](#reporting-bugs)
- [Feature Requests](#feature-requests)
- [Contributing](#contributing)
- [Contact Information](#contact-information)

## Getting Help

If you need help with MNad.Fun, here are the best ways to get support:

1. **Documentation**: Check the [README.md](./README.md) for comprehensive documentation
2. **Issues**: Search existing issues on GitHub to see if your question has already been answered
3. **New Issue**: Create a new issue if you can't find an answer
4. **Discussions**: Use GitHub Discussions for general questions and community help

## Common Issues

### Contract Deployment Issues

**Issue**: Contracts fail to deploy or initialize

**Solutions**:
- Ensure you have sufficient gas for deployment
- Check that all required parameters are provided correctly
- Verify that factory contracts are initialized before use
- Ensure proper contract permissions and ownership

### Transaction Failures

**Issue**: Transactions revert with error messages

**Common Causes**:
- Insufficient balance (WMon or tokens)
- Slippage protection triggered
- Deadline expired
- Insufficient allowance for token spending
- Contract is locked (bonding curve)

**Solutions**:
- Check your WMon balance before trading
- Increase slippage tolerance if using protected functions
- Use a future deadline (block.timestamp + X)
- Approve tokens before selling operations
- Wait until the bonding curve unlocks or check its lock status

### Fee Calculation Errors

**Issue**: Fee validation fails

**Solutions**:
- Verify the fee amount matches the bonding curve configuration
- Check fee denominator and numerator values
- Ensure the fee is calculated correctly: `fee >= (amount * denominator) / numerator`

### Permit/Approval Issues

**Issue**: Permit or approval fails

**Solutions**:
- Verify signature parameters (v, r, s) are correct
- Check that the deadline hasn't expired
+- Ensure the domain separator matches the contract
- Verify the nonce is correct

## Frequently Asked Questions (FAQ)

### General Questions

**Q: What is WMon?**  
A: WMon is the wrapped version of the native Monad token, providing ERC20 compatibility for trading operations.

**Q: How do bonding curves work?**  
A: Bonding curves use a constant product formula (k = x * y) to determine prices. As more tokens are bought, the price increases. Virtual reserves are used for initial price calculation.

**Q: What happens when a bonding curve locks?**  
A: When the locked token target is reached, the bonding curve locks and trading stops. The curve can then be listed on a DEX.

**Q: Can I create multiple tokens?**  
A: Yes, each call to `createBc` creates a new token and bonding curve pair.

**Q: How are fees calculated?**  
A: Fees are calculated based on the bonding curve's fee configuration (denominator/numerator). For buys the fee is taken from the input amount, and for sells it is taken from the output amount.

### Trading Questions

**Q: What's the difference between `buy` and `protectBuy`?**  
A: `buy` executes immediately at the current price. `protectBuy` includes slippage protection and reverts if the price moves beyond `amountOutMin`.

**Q: Do I need to approve tokens before selling?**  
A: Yes, you need to either:
- Approve tokens using the `approve()` function, or
- Use permit-based functions (`sellPermit`, `protectSellPermit`, etc.)

**Q: What is the deadline parameter?**  
A: The deadline ensures transaction freshness. It must be a future timestamp (block.timestamp + duration). Transactions with expired deadlines will revert.

**Q: Can I buy an exact amount of tokens?**  
A: Yes, use `exactOutBuy` to specify the exact number of tokens you want to receive.

**Q: Can I sell for an exact amount of WMon?**  
A: Yes, use `exactOutSell` to specify the exact amount of WMon you want to receive.

### Technical Questions

**Q: What Solidity version is used?**  
A: Contracts are written in Solidity ^0.8.13.

**Q: What testing framework is used?**  
A: Foundry is used for testing. See [test/README.md](./test/README.md) for more testing information.

**Q: How do virtual reserves differ from real reserves?**  
A: Virtual reserves are used for initial price calculation and do not reflect actual balances. Real reserves track the actual token balances in the bonding curve contract.

**Q: What happens during DEX listing?**  
A: When `listing()` is called, tokens and WMon are transferred to a Uniswap V2 compatible pair, liquidity is provided, and LP tokens are burned.

## Troubleshooting

### Test Failures

If tests are failing:

1. **Check Foundry version**: Ensure you're using a compatible Foundry version
   ```bash
   forge --version
   ```

2. **Update dependencies**: Pull the latest dependencies
   ```bash
   forge update
   ```

3. **Clean build**: Clean and rebuild
   ```bash
   forge clean
   forge build
   ```

4. **Run specific test**: Isolate the failing test
   ```bash
   forge test --match-test testName
   ```

5. **Verbose output**: Get more information
   ```bash
   forge test -vvv
   ```

### Compilation Errors

If contracts won't compile:

1. **Check Solidity version**: Ensure the version matches (^0.8.13)
2. **Verify imports**: Check that all import paths are correct
3. **Install dependencies**: Ensure all dependencies are installed
   ```bash
   forge install
   ```
4. **Check remappings**: Verify that `remappings.txt` is correct

### Runtime Errors

If transactions fail at runtime:

1. **Read error messages**: Error messages are designed to be descriptive
2. **Check event logs**: Events can provide additional context
3. **Verify state**: Ensure contracts are in the expected state
4. **Check permissions**: Verify the caller has the required permissions
5. **Gas estimation**: Ensure sufficient gas is provided

## Reporting Bugs

When reporting bugs, please include:

1. **Description**: Clear description of the issue
2. **Steps to Reproduce**: Detailed steps to reproduce the bug
3. **Expected Behavior**: What should happen
4. **Actual Behavior**: What actually happens
5. **Environment**:
   - Solidity version
   - Foundry version (if applicable)
   - Network (if applicable)
6. **Error Messages**: Full error messages or logs
7. **Code Samples**: Minimal code to reproduce (if applicable)
8. **Screenshots**: For visual issues (if applicable)

### Bug Report Template

```markdown
## Bug Description
Brief description of the bug

## Steps to Reproduce
1. Step one
2. Step two
3. Step three

## Expected Behavior
What should happen

## Actual Behavior
What actually happens

## Environment
- Solidity Version: 
- Foundry Version: 
- Network: 

## Error Messages
Paste error messages here

## Additional Context
Any additional context or information
```

## Feature Requests

We welcome feature requests! When submitting:

1. **Clear Description**: Describe the feature clearly
2. **Use Case**: Explain why this feature would be useful
3. **Potential Implementation**: If you have ideas, share them
4. **Alternatives**: Discuss any alternatives you've considered

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests for new functionality
5. Ensure all tests pass
6. Submit a pull request

For more details, see the contribution guidelines (if available).

## Contact Information

- [Whatsapp](https://wa.me/14105015750)
- [Telegram](https://wa.me/14105015750)
- [Discord](https://discordapp.com/users/985432160498491473)
- [Twitter](https://x.com/novustch)

## Additional Resources

- [README.md](./README.md) - Main documentation
- [README_EN.md](./README_EN.md) - Alternate README file
- [test/README.md](./test/README.md) - Testing documentation
- [GitHub Repository](https://github.com/Novus-Tech-LLC/Monad-Memecoin-Launchpad) - Source code

---

**Note**: For security vulnerabilities, please use responsible disclosure and contact maintainers privately rather than opening a public issue.