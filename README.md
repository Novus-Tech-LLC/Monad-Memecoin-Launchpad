[English (detailed guide)](./README_EN.md)

# MNad.Fun Smart Contract

This file provides a concise overview of the MNad.Fun protocol. For a more comprehensive and highly detailed description of the architecture and flows, see `README_EN.md`.

## Contact 

| Platform | Link |
|----------|------|
| 📱 Telegram | [t.me/novustch](https://t.me/novustch) |
| 📲 WhatsApp | [wa.me/14105015750](https://wa.me/14105015750) |
| 💬 Discord | [discordapp.com/users/985432160498491473](https://discordapp.com/users/985432160498491473)

<div align="left">
    <a href="https://t.me/novustch" target="_blank"><img alt="Telegram"
        src="https://img.shields.io/badge/Telegram-26A5E4?style=for-the-badge&logo=telegram&logoColor=white"/></a>
    <a href="https://wa.me/14105015750" target="_blank"><img alt="WhatsApp"
        src="https://img.shields.io/badge/WhatsApp-25D366?style=for-the-badge&logo=whatsapp&logoColor=white"/></a>
    <a href="https://discordapp.com/users/985432160498491473" target="_blank"><img alt="Discord"
        src="https://img.shields.io/badge/Discord-7289DA?style=for-the-badge&logo=discord&logoColor=white"/></a>
</div>

Feel free to reach out for implementation assistance or integration support.

## Overview

MNad.Fun is a smart contract system on the Monad blockchain for launching and trading tokens backed by bonding curves. It provides:

- A router contract that coordinates token creation and trading (`GNad`).
- A factory that deploys dedicated bonding curves and token contracts for each new asset.
- A wrapped native token (`WMon`) for seamless ERC20-based trading.
- A multisig fee vault to secure protocol revenue.

## Where to Start

- **Developers / Auditors**: Read the full documentation in `README_EN.md`.  
- **Users / Integrators**: Review the support guide in `SUPPORT_EN.md` for operational and troubleshooting details.

## Testing

Run the test suite with Foundry:

```bash
forge test
forge test -vv
forge test --gas-report
```

## Support

For questions, integrations, or issues:

- Open an issue in this repository.
- Refer to the detailed [Support Guide](./SUPPORT_EN.md).
