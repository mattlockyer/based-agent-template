# Based Agent Template

This is a monorepo template for deploying a Based Agent on NEAR and Phala Cloud.

The template has 2 main components to code and deploy: a NEAR contract and a NextJS docker application with API.

# Based Agents

Based Agents are multichain crypto AI agents, verifiable from their source code through to their onchain transactions.

Based Agents can:
- Sign transactions for any chain
- Custody cryptoassets
- Verifiably access off-chain LLMs, APIs and data
- Preserve privacy

<img width="691" alt="image" src="https://github.com/user-attachments/assets/9b8f3308-ae6d-4349-a312-9984e36e38c6" />

Components of a Based Agent are:
1. A verifiable TEE deployment on Phala Cloud
1. API layer as docker image (NextJS App)
1. NEAR Contract to verify a TEE's remote attestation and stack
1. NEAR's Chain Signatures for multichain key management 

## 1. TEE deployment on Phala Cloud 

Phala Cloud is a TEE as a service provider to deploy multiple docker images inside a trusted execution environment. This environment exposes their dStack SDK where one can access the remote attestation of the TEE and derive keys. dStack also provides information about the docker images running inside the TEE from the docker-compose.yaml.

Based Agents use the following information from the TEE Deployment on Phala Cloud:
1. The remote attestation
1. The unique key derived from the TEE's hardware
1. The docker-compose.yaml data which contains the SHA hash of the individual docker images running on the TEE

![image](https://github.com/user-attachments/assets/aae0db8a-cbcb-4a30-858b-3fcadc0f1f17)

## 2. API Layer

Based Agents have an API Layer that acts as a bridge between the LLMs and the NEAR Contract.

The template includes the following code for the API Layer:
1. The API will derive a unique key for this instance of the TEE (a NEAR Account). This key is in memory and inside the TEE, unable to be exfiltrated, verifiable by code inspection and the SHA hash of the docker image.
2. A call from the API layer to the NEAR contract to verify the TEE's remote attestation (so we know it's running in a TEE). This call includes the SHA hash of the docker images used and the checksum of the remote attestation to be viewed on the Phala Cloud explorer.
3. A call to `get_tee` which returns the SHA hashes and checksum of the TEE's instance.

Subsequent API layer code and calls to the NEAR Contract are left to the developer and their custom smart contract development for their based agent.

What to use the API layer for (remember it's verified!):
- providing offchain (but online) data to the LLM
- preprocessing LLM inference into specific smart contract method calls
- arbitrary offchain compute

## 3. NEAR Contract

This template provides a NEAR Contract that can do the following:
1. Registration - verify a TEE's remote attestation and store the TEE data
1. Method Calls - made from the TEE's NEAR Account, follow up method calls virtually zero cost

## 4. NEAR Chain Signatures

Using a path offset from the NEAR Contract, we can have contract controlled accounts for any chain.

To derive accounts we use...

To get signatures for transactions on these accounts we call the NEAR Chain Signatures contract from our contract with the path offset of the account we'd like to receive a signature for.

# What can be built with Based Agents?

*LLMs as docker images in the Based Agent stack = verifiable inference*

LLMs deployed as docker images can easily be leveraged by other applications deployed in the same docker stack. With Based Agents, an LLM deployed as a docker image is used by the API Layer in the same docker stack on the Trusted Execution Environment (TEE). Based Agents can use 1 or more LLMs to provide the inference for the API layer.

LLMs could be used to:
- interpret API calls and provide reasoning and inference that translates these to more specific smart contract method calls
- digest realtime sentiment about particular cryptocurrencies and provide higher order recommendations for trading
- reason about multiple price sources and provide a more robust price oracle vs. multi party computation

# How a Based Agent is Verified?

Based Agents have an API Layer that is running inside a trusted execution environment (TEE) and a NEAR Contract deployed onchain. Here are the following steps that the API Layer takes to verify itself onchain, so that subsequent calls from the API Layer can be trusted.

## 1. Ephemeral key for a NEAR Account

The API Layer derive route (`/pages/api/derive`) creates a key derived from the TEE's hardware KMS and additional entropy. *If the TEE is rebooted, or redeployed a new account would be created and need to be funded*. Loss of funds from these accounts can be mitigated by deploying minimal funds and using NEAR's Meta Transactions to fund the smart contract calls.

## 2. Remote Attestation (RA) quote, quote collateral and docker image hashes

The API Layer's verify route (`/pages/api/verify`) gets the remote attestation quote in hex, gets the necessary quote collateral through a Phala API call that is verified through a PCCS_URL from Intel, and lastly the `docker_compose.yaml` data for the TEE's stack is returned.

All of these arguments are made to the NEAR Contract's verify method. The contract checks the quote using Phala's [dcap-qvl](https://github.com/Phala-Network/dcap-qvl) Rust crate and a verified report about the TEE is returned.

Once a TEE is verified, the NEAR Account of the TEE is registered in the contract for future calls.

Also stored for each TEE is:
- the SHA hash of the API Layer docker image so that the code running is verifiable
- the checksum of the RA quote which can be verified on Phala Cloud's [TEE Attesation Explorer](https://proof.t16z.com/)

# API Layer Local Development

The API Layer template is a NextJS application that offers both an API via the `/pages/api` routes and an optional UI under `/pages`.

The UI found at `index.js` is intended to guide you through the first steps of verifying your template against the NEAR Contract template. *Note: this is only possible when deployed on Phala Cloud.* In order to pass the verification in the NEAR Contract, your API Layer must be deployed on Phala Cloud and running on TEE hardware to get a valid remote attestation (RA) quote, that can be verified by the NEAR Contract.

However, this does not stop you from developing your API Layer. You can add your own API routes, call external APIs, work with data, and prepare your NEAR Contract calls. In fact, you can skip the verification step in your NEAR Contract, and make calls with the assumption that you will add in the proper logic to "allow list" these calls only by verified based agents when the NEAR Contract is deployed live.

To develop locally, you need only use:
- `yarn`
- `yarn dev`

For making calls to the NEAR Contract it's recommended you provide a local `.env.development.local` file with the following fields:
```bash
NEXT_PUBLIC_accountId=[YOUR_NEAR_DEV_ACCOUNT_ID]
NEXT_PUBLIC_secretKey=[YOUR_NEAR_DEV_ACCOUNT_SECRET_KEY]
```

This dev account will be used to call the NEAR Contract. It won't be used when you deploy to Phala Cloud. The template will create an ephemeral NEAR Account and in the UI example will ask for funds before verification and subsequent calls to the contract can be made.

# NEAR Contract Local Development

Local development of the NEAR Contract has a few dependencies. You will need to have Rust installed and you will also need [cargo near](https://github.com/near/cargo-near).

The contract provides only the basic methods to verify the API Layer's remote attesation (RA) quote coming from the TEE deployed on Phala Cloud.

This is handled by the `verify` method in `lib.rs`. See the API Layer route in `pages/api/verify` for more information on where the data is gathered for this call.
