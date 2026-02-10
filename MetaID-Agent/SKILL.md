---
name: MetaID-Agent
description: Create and manage MetaID wallets and accounts. This skill handles wallet creation, MetaID registration, node creation, and sending Buzz messages on the MVC network. Use when users want to: (1) Create a new MetaID Agent/robot/proxy with a wallet, (2) Register a MetaID account, (3) Create MetaID nodes, (4) Send Buzz messages to the MVC network. Requires Node.js >= 18.x.x and TypeScript. Dependencies: @scure/bip39, @metalet/utxo-wallet-service, bitcoinjs-lib, ecpair, @bitcoinerlab/secp256k1, crypto-js, meta-contract.
---

# MetaID-Agent

MetaID-Agent skill provides comprehensive wallet and MetaID account management capabilities for the MVC network. It handles the complete lifecycle from wallet creation to MetaID registration and Buzz message sending.

## Core Capabilities

1. **Wallet Creation** - Generate mnemonic phrases and derive addresses for MVC, BTC, and DOGE chains
2. **MetaID Registration** - Register new MetaID accounts with gas subsidies (including MVC init rewards)
3. **MetaID Node Creation** - Create MetaID nodes with custom usernames on MVC and DOGE chains
4. **Buzz Messaging** - Send Buzz messages to the MVC network using simpleBuzz protocol

## Prerequisites

Before using this skill, ensure:
- Node.js >= 18.x.x is installed
- TypeScript is installed globally or available in the project
- All required dependencies are installed (see Dependencies section)

Run `scripts/check_environment.sh` to verify the environment.

## Dependencies

This skill requires the following npm packages with specific versions:
- `@scure/bip39@1.6.0`
- `@metalet/utxo-wallet-service@0.3.33-beta.5`
- `bitcoinjs-lib@6.1.7`
- `ecpair`
- `@bitcoinerlab/secp256k1@1.2.0`
- `crypto-js`
- `meta-contract`
- `sharp` - 头像压缩（Node.js 环境）

Install dependencies with:
```bash
npm install
```

## Workflow Overview

The MetaID-Agent workflow consists of three main phases:

1. **Wallet Creation** - Generate mnemonic, derive addresses, save to `account.json` (project root)
2. **MetaID Registration** - Claim gas subsidy, create MetaID node with username
3. **Buzz Creation** - Send initial Buzz message to the network

## Usage

### Trigger Detection

The skill activates when user prompts contain keywords like:
- "我要创建一个 MetaID Agent"
- "我要创建一个代理"
- "我要创建一个机器人"
- "创建 MetaID 钱包"
- "注册 MetaID"

### Wallet Selection Logic

1. **New Wallet Creation**: Triggered when keywords indicate creation intent
2. **Existing Wallet Selection**: 
   - If root `account.json` exists with accounts, match by username/address from user prompt
   - If no match found, use accountList[0] as default
   - New wallets are unshifted to the front of accountList

### Main Execution Flow

Execute the main script:
```bash
ts-node scripts/main.ts "<user_prompt>"
```

The script will:
1. Check environment prerequisites
2. Parse user prompt for username and buzz content
3. Determine if wallet creation or selection is needed
4. Create/select wallet and save to root `account.json`
5. Register MetaID if userName is empty
6. Create MetaID node with username
7. Fetch user info by address to get globalMetaId and update root `account.json`
8. Send Buzz message if content is provided

## Account Management

Account data is stored in `account.json` at the **project root** (MetaApp-Skill/) with the following structure:

```json
{
  "accountList": [
    {
      "mnemonic": "word1 word2 ... word12",
      "mvcAddress": "MVC address",
      "btcAddress": "BTC address",
      "dogeAddress": "DOGE address",
      "publicKey": "hex public key",
      "userName": "username or empty string",
      "path": "m/44'/10001'/0'/0/0",
      "globalMetaId": "global metaid (optional, fetched after MetaID registration)",
      "metaid": "metaid (optional, synced from getUserInfoByAddressByMs)",
      "avatarPinId": "txid+i0 (optional, created when static/avatar has image)",
      "avatar": "https://file.metaid.io/metafile-indexer/api/v1/files/accelerate/content/${avatarPinId} (optional)",
      "chatPublicKey": "hex (optional, ECDH pubkey for private chat)",
      "chatPublicKeyPinId": "txid+i0 (optional)",
      "llm": [
        {
          "provider": "deepseek",
          "apiKey": "",
          "baseUrl": "https://api.deepseek.com",
          "model": "DeepSeek-V3.2",
          "temperature": 0.8,
          "maxTokens": 500
        }
      ]
    }
  ]
}
```

**Important Note**: 
- Account file location: **project root** `account.json` (shared with MetaID-Agent-Chat).
- If `MetaID-Agent/account.json` exists, it will be migrated to root on first run.
- Empty accounts (accounts with empty mnemonic) are automatically filtered out when writing.
- **llm**：数组格式，`llm[0]` 默认从 `.env` / `.env.local` 读取；用户可手动添加 `llm[1]`、`llm[2]` 等；未指定时默认使用 `llm[0]`；旧格式（对象）会自动迁移为 `[llm]`。
- **metaid**：创建 Agent 后调用 `getUserInfoByAddressByMs`，若返回 `metaId` 则同步到 account.json 和根目录 userInfo.json。
- **avatarPinId**：若 `MetaID-Agent/static/avatar` 下有图片文件（jpg/png/gif/webp/avif），创建 namePin 成功后自动创建 avatar pin 并写入。
- **avatar**：格式为 `https://file.metaid.io/metafile-indexer/api/v1/files/accelerate/content/${avatarPinId}`，用于前端展示。
- **chatPublicKey**：创建 Agent 时若 `userInfo.chatPublicKey` 为空则自动创建 `/info/chatpubkey` 节点；也可通过 `create_chatpubkey.ts` 为已有 Agent 单独创建。
- **Agent 人设（character / preference / goal / masteringLanguages / stanceTendency / debateStyle / interactionStyle）**：在创建完 **name 节点** 时同时写入 `account.json` 的该账户下。若用户在提示词中指定（如「帮我生成一个 MetaID Agent，名字叫 Sam，性格 XXX，爱好 XXX，目标 XXX，擅长语言 XXX，立场 XXX，辩论风格 XXX，互动风格 XXX」），则按提示词填充；未指定则用系统默认策略 `getRandomItem` 分配。这些人设会在所有涉及 LLM 调用的场景中作为 config 传入（不限于群聊）。选项见 `utils.ts` 中的 `CHARACTER_OPTIONS`、`PREFERENCE_OPTIONS`、`GOAL_OPTIONS`、`LANGUAGE_OPTIONS`、`STANCE_OPTIONS`、`DEBATE_STYLE_OPTIONS`、`INTERACTION_STYLE_OPTIONS`。
- **path**：每个账户的推导路径，存于 `account.json` 的 `path` 字段。`wallet.ts` 的 `getPath(options?)` 会从**要操作的** accountList 项读取 path；新建 agent/新建钱包时使用 `getPath({ defaultPath: DEFAULT_PATH })`，默认值为 `m/44'/10001'/0'/0/0`。当用户在自然语言中指定「钱包路径使用1」或「钱包路径使用 m/44'/10001'/0'/0/1」时，主流程会通过 `extractWalletPathFromPrompt` 解析出对应 path/index，并据此设置新建账户的 `path` 与 `addressIndex`。
- **addressIndex**：`getCurrentWallet`、`getAddress`、`getPublicKey`、`getCredential`、`getUtxos`、`createPin`、`getEcdhPublickey`、`createBuzz` 等所有涉及当前账户的接口均支持在 options 中传入可选属性 `addressIndex`（不传则使用 0）。从 `account.path` 解析索引请使用 `parseAddressIndexFromPath(account.path)`；对于仅给出索引（如「路径1」）的情况，可通过 `buildPathFromIndex(1)` 得到 path 并保持与 `addressIndex` 一致。业务在操作已有 account 时应传入对应 `addressIndex` 以与 path 一致。详见 `references/wallet-operations.md`。

## Error Handling

All errors are logged to `log/error.md` with:
- Error message
- Method/function where error occurred
- Timestamp
- Execution context

## Scripts

- `check_environment.sh` - Validates Node.js and TypeScript installation
- `wallet.ts` - Wallet creation and address derivation functions (supports MVC, BTC, DOGE)
- `api.ts` - API functions for fetching UTXOs, broadcasting transactions, and claiming rewards
- `metaid.ts` - MetaID registration and node creation functions (supports MVC and DOGE chains)
- `doge.ts` - DOGE chain specific inscription and transaction building functions
- `buzz.ts` - Buzz message creation and sending
- `avatar.ts` - 头像压缩与处理（从 static/avatar 读取图片，Node.js 环境使用 sharp）
- `chatpubkey.ts` - ECDH 生成 chatpubkey（供私聊用）
- `env-config.ts` - 从 .env / .env.example 读取 LLM 等配置
- `create_agents.ts` - 批量创建 MetaID Agent（支持头像、metaid 同步、chatpubkey、llm）
- `create_chatpubkey.ts` - 为指定 Agent 创建 chatpubkey 节点
- `main.ts` - Main orchestration script

### 头像设置

将图片文件（jpg/png/gif/webp/avif）放入 `MetaID-Agent/static/avatar` 目录，执行 `create_agents.ts` 创建 Agent 时会自动创建 avatar pin 并写入 `avatarPinId`。

**为已有 Agent 单独设置头像**：`npm run create-avatar -- "AI Eason"`（需先确保 `static/avatar` 下有图片）

- **文件大小**：需小于 1MB，超过时提示「上传头像文件超过1 MB，请使用小于1MB图片文件设置头像」并跳过头像创建
- **无图片**：若无图片则提示并跳过头像创建

### 单独创建 chatpubkey

```bash
npm run create-chatpubkey -- "AI Eason"
# 或 npx ts-node scripts/create_chatpubkey.ts <userName|mvcAddress|metaid>
```

### 同步 metaid 信息

```bash
npm run sync-agent-metaid -- "AI Eason"
```

### LLM 配置（数组格式）

- `llm` 为数组：`llm[0]` 默认来自 .env，创建 Agent 时自动写入
- 用户可手动添加 `llm[1]`、`llm[2]` 等
- 调用时若未指定使用哪一个，默认使用 `llm[0]`
- 工具函数 `getAccountLLM(account, index?)`：获取指定下标的 LLM 配置，index 默认 0
- 旧格式（`llm: {...}`）读写时自动迁移为 `llm: [{...}]`
- 为已有账户补充 llm[0]：`npm run sync-account-llm`

## Cursor 人机交互与一键执行规范

当用户在 Cursor 对话中提出需要**执行某条命令或脚本**的需求时（例如「让 AI Eason 监听群聊信息并且当监听到新消息之后对其进行回复」），**不应在输出推荐方案后结束任务**，而应：

1. **先说明推荐方案**：简要说明已按需求接好的能力，以及推荐的一种或多种用法（如专用脚本、通用脚本等）。
2. **继续代为执行推荐方案**：在给出命令示例后，**不要停止**，应通过 Cursor 的终端或执行能力，**直接帮用户把推荐方案执行起来**（例如执行 `cd MetaID-Agent-Chat && ./scripts/run_ai_eason_listener.sh --no-open`）。
3. **便于用户一键确认**：执行方式应设计为「用户可在对话中通过接受/确认（如点击接受、确认运行）即可完成执行」，相当于**一键替用户跑起来**，而不是仅把命令贴在对话框里让用户自己复制到终端执行。

**原则**：当输出中包含「请执行以下命令」「在项目根目录下执行」「推荐方式如下」等让用户去跑某脚本的指引时，任务不应在此结束；应继续把该推荐命令实际执行起来，让用户只需在 Cursor 中确认/接受即可一键完成执行。

## References

For detailed implementation details, see:
- `references/wallet-operations.md` - Wallet creation and management (MVC, BTC, DOGE)
- `references/metaid-protocol.md` - MetaID registration and node creation
- `references/buzz-protocol.md` - Buzz message protocol
