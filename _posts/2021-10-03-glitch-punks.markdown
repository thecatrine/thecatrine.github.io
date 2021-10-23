---
layout: post
title:  "Building a Solana NFT"
date:   2021-10-03 12:34:01 -0500
categories: update solana rust
---

I recently spent a weekend getting up to speed on the Solana ecosystem by launching an NFT.

[Solana](https://solana.com/) is a blockchain that promises low fees, fast transaction times, and uses Rust for their on-chain programs. A lot of new projects, especially NFTs, are being built on Solana because you can make small transactions without your whole balance being eaten by fees.

I've worked with several smart contract systems(1), and Solana is very nice and low friction in several ways, but there are some unique design decisions that I found difficult to understand without a top-level overview of how all the parts interact.

This article is an attempt to write the motivational overview I with I'd had when I started this project, and share links that were helpful to me. I'll give a high level overview of the Solana architecture, and explain the reasons behind common Solana program structure, with concrete examples of working code from my NFT.

## The Idea

I was excited about the low fees to deploy on Solana, and being able to use Rust for development. I mentioned I was trying to come up with an idea, and got to talking in discord about possibly making an NFT. My friend Josh pitched me a broken, glitched parody of Crypto Punks.

> Oh I was thinking completely broken punks. Like tragically misshapen and mutated punks. \
> People who life has trampled punks. \
> Josh

I was sold. I spent the rest of the weekend learning how to build the contract, deploy all the assets, and hook everything up.

(You can check out the completed app here --> [Glitch Punks](https://glitchpunks.art))

I wanted to learn the framework, so I wrote everything from scratch. There are projects which will let you mint an NFT directly if you just want to have one in-hand as fast as possible, but I wanted to get into the nitty-gritty of building the contract itself.

There were some parts of contract development with Solana that surprised me, or were confusing without explanation, and this is an attempt to put all the explanations that helped me into one place. Hopefully after reading this post and reading the source code, you'll come away with a clear picture of what is required to build a contract on Solana.

## Quick Architecture Comparison

The architecture I'm familiar with for smart contracts is that there's an address corresponding to a public key where people can access your contract, and the contract implements all the logic it needs to serve it's purpose.

Most NFTs on Ethereum or it's derivatives are [ERC-721](https://ethereum.org/en/developers/docs/standards/tokens/erc-721/) tokens, which defines a standardized contract interface for Non Fungible Tokens. But each contract implements the functionality for itself, and has a chunk of memory that is for free-form storage where it stores its data. Usually this is a big map from users to tokens owned.

Solana's architecture has some significant differences in how the programs are written. 
- They call them "programs" rather than "smart contracts"
- An `Account` is the main primitive for interacting with data or programs. Accounts usually correspond to a public key, but don't always.
- An `Account` is like a file. It can hold a program or data, but both are `Accounts`
- If an `Account` is storing data, you allocate the space for the data at account creation time.
- Some core functionality for creating accounts and transfering tokens is implemented by programs like the [Token Program](https://spl.solana.com/token), rather than by language primitives.
- `Accounts` in Solana have a notion of "rent" which the `Account` has to pay to stay open. However, there is a notion of a "rent-exempt" minimum `Account` balance (~ 2 years of rent). If an `Account` has more than this minimum amount it doesn't have to pay rent. In practice everything in the ecosystem seems to just transfer the rent exempt amount and not think about it.
- Everything you need to access in your program is passed into the transaction when you execute it. This includes addresses for programs you need to call, which are loaded by the runtime (also a program) and passed to your entrypoint. This is sorta wonky if you're coming from other smart contract languages, and expect to just hard-code an address for a contract you need to access. The current recommended pattern seems to be that you hardcode an address, ask the user to pass in that account, and then confirm they passed an account with the same adress.
- A program is an ELF binary. Details [Here](https://docs.solana.com/developing/on-chain-programs/overview)

Writing code for Solana felt like writing code for a microcontroller, everything has a very well defined interface that seems un-opinionated about how you use it, but it's rigid in what capabilities it has and the ergonomics aren't great. My final working contract is substantively longer than an equivalent one written in Solidity.

## Program Structure

You can view the source code for the final contract [Here](https://github.com/thecatrine/Nift)

And the [frontend](https://github.com/thecatrine/Glitch-Punks-Client)

A program in solana has an entrypoint that is called by the runtime when the program is executed, and passes in an opaque binary blob containing accounts and arguments. There's very little magic here, so a fair amount of the source code is dedicated to managing encoding and decoding data.

Most projects divide their main program structure into the following files:
- entrypoint.rs  - Boilerplate to invoke instruction logic.
- instruction.rs - Define instructions we expect to get from the network, encode arguments.
- processor.rs   - Handle actual program logic

### Entrypoint

The way you interact with an on-chain program in Solana is by "invoking" an instruction against the program from either another on-chain program, or from outside the chain. These interactions are with a system program responsible for program execution, and they get forwarded to our program by calling a declared entrypoint function.

The entrypoint looks like this:
```
entrypoint!(process_instruction);
fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    Processor::process(program_id, accounts, instruction_data)
}
```

Instructions come across the wire as a `[u8]` and you have to deserialize them yourself into whichever arguments your method needs.
You don't pass needed accounts along as arguments, these are provided in a separate `[AccountInfo]` when your instruction is invoked.
`program_id` is the pubkey of your own program.

This is a very low-level interface, but it's clean and familiar from writing systems code. Most of the rest of the functionality we'll need is in the [solana_program](https://docs.rs/solana-program/1.7.14/solana_program/) crate, and is well documented at the object level.

In some sense that's all there is to understanding how a Solana program works, but let's draw the rest of the Owl.

### Processor

The bulk of the actual program logic is in `processor.rs`. First let's cover the boilerplate that handles the instruction, and then move on to the actual work of minting the token itself.

First we need to process the opaque binary blob we received from the system program into a format that's easier to deal with. We define a function `process` which we call from the entrypoint to interpret the bits on the wire.

```
pub fn process(
    program_id: &Pubkey, 
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    let instruction = NiftInstruction::unpack(instruction_data)?;
```

Here we're calling into `instruction.rs` to do the actual deserialization. We could have done it in-line since we only have one instruction it can be, but I stuck to this architecture because it seemed fairly standard in examples and I could imagine the logic being complicated without an interface if we had many possible instructions.

We define an enum type to represent our instructions and their arguments, and a function to unpack our opaque data to this enum type.

```
pub enum NiftInstruction {
    MintNFT {},
}

pub fn unpack(input: &[u8]) -> Result<Self, ProgramError> {
    let (tag, rest) = input.split_first().ok_or(InvalidInstruction)?;
    Ok(match tag {
        1 => Self::MintNFT {},
    }),
    _ => return Err(InvalidInstruction.into()),
}
```

Since we don't have any arguments in our mint instruction, there's very little to understand there. We could imagine having a more complicated instruction that did have an argument, with some logic to convert an unordered `[u8]` to that type.

The only thing we use is the first byte, provided as a selector of which instruction we're invoking.

Back in `processor.rs` we now have a deserialized instruction with Rust data types for any arguments we need to use, and we shell out to a handler to do the actual work based on which instruction we're invoking. Again we only have one possible instruction, so this is mostly to illustrate what the boilerplate would look like in a more complicated contract.

```
match instruction {
    MintNFT => {
        msg!("Minting NFT");
        Self::process_mint_nft(accounts, program_id)
    }
}
```

With that out of the way, we're ready to write the code that actually interacts with the blockchain to produce the token.

## Minting an NFT

To mint an NFT on Solana we want to do the following:
- Accept a fee.
- Mint an NFT with metadata pointing to our artwork and attributes.

These are both accomplished differently from how I expected.

### Accepting a fee
In Ethereum, there's a notion of ETH being sent along with a method call. So you could call a mint function along with transfering the fee and the method call could check you'd done that.

But in Solana that's not the case. If we want our contract to be aware of the fee we need to pay it during our contract.

Fortunately we can just get a mutable reference to the balance of an account and update it (2):
```
FEE_LAMPORTS = FEE_SOL * 1_000_000_000;
**source_info.try_borrow_mut_lamports()? -= FEE_LAMPORTS;
**dest_info.try_borrow_mut_lamports()? += FEE_LAMPORTS;
```
Unfortunately, Accounts have a notion of owners, and our program doesn't own the user's wallet, so we can't subtract from their balance. You don't need to own an account to add to it's balance, so that's not a problem. (A transaction will fail if the changes in balance don't total up correctly.)

The standard solution here appears to be to have the user create an account owned by our program with the fee already in it, and then have our program (which now owns the fee account) transfer the funds out of that account during the minting process.

### Side-Note about Data Storage and Program Derived Addresses
Many Solana operations depend on creating an Address to store data or tokens or what have you, and I ended up passing them around a lot in the implementation. Many of the instructions that you need to invoke on other contracts also require a list of `[AccountInfo]` to be passed in, and since you /can't/ derive them on-chain without passing the address in at the beginning of the transaction, you end up passing a lot of accounts in.

Additionally, it's important to note that since all data is stored on Accounts, and the Accounts have a defined size at instantiation time that determines the cost to create them (more on that later), it's not possible to add data to an Account in a later transaction and have the user pay for storage like you would when you add a key to a Map in an ERC-721 token contract. If you want the user to pay to instantiate the storage you need to create a new Account to store just their data.

So if you need to store data in a specific place that's related to a user, you need to store it in an Account. But if the user passes in the account then you can't write down which account had that user's data on it, because you'd have to store it in an account.

A [Program Derived Address](https://docs.solana.com/developing/programming-model/calling-between-programs#private-keys-for-program-addresses) is derived from your program id and a string path that produces a key specific to that string. It has no private key and can only be "signed for" by the program it was derived for.

So in the example above you could derive a PDA for the user's data like this:
```
let (user_data_key, user_data_bump_seed) = Pubkey::find_program_address(&[b"user_data", user_id], program_id);
```
### Minting the Damn Thing.

In Solana there is a system owned program that handles all tokens and token accounts, including the Solana native token Sol. We can mint an NFT by interacting with this program to create a new type of token that does not support decimal amounts, and minting a single one of these tokens.

In order to accomplish this we have to do the following steps, according to the docs for the [Token Program](https://spl.solana.com/token).

- Initialize an account as a mint, with a specific account as the mint authority.
- Initialize an account to hold tokens of the type minted by that token mint, owned by the user.
- Mint a single token to that account.

This produces an NFT, but since we're not writing a custom implementation of the protocol as we would on other smart contract platforms, there's no metadata.

Metadata is handled by a different program than the token, according to the metadata standard created by Metaplex [Metadata Docs](https://docs.metaplex.com/nft-standard)

So in order to register metadata which will be recognized by [Phantom Wallet](https://phantom.app/) and NFT marketplaces, we need to register this token we just created with the on-chain [Token Metadata](https://github.com/metaplex-foundation/metaplex/tree/master/rust/token-metadata/program) program. 


- Create a metadata account as a PDA with a path determined by the token we minted so apps know where to look for it later.
-  Call the metadata program on-chain to set the uri to our json files.

From there the calls are fairly straightforward. For each cross-program invocation we construct an instruction by looking at the docs for that contract and constructing it, before "invoking" the instruction, which is the same interface we used to write our entrypoint.rs at the begining.

Here's an example from the code to initialize a token account.
```
let initialize_account_ix = spl_token::instruction::initialize_account(
    token_program.key,
    final_acct.key,
    mint_acct.key,
    signer.key,
)?;
invoke(
    &initialize_account_ix,
    &[
        final_acct.clone(),
        mint_acct.clone(),
        signer.clone(),
        rent_acct.clone(),
    ],
);
```

One thing to mention is that if you are trying to do an operation that requires you to sign with a PDA, then you don't have a private key to sign with. Instead of calling `invoke(ix, &[accts])`, you call `invoke_signed(ix, &[accts], signers_seeds)` where signers_seeds has the type `signers_seeds: &[&[&[u8]]]`.

That type is crazy but if you look at the example from the source it's fairly straightforward in practice. Here's an example of us using a PDA as the minting authority for the NFT.

```
let (mint_authority, mint_authority_bump_seed) = Pubkey::find_program_address(&[b"mint_authority"], program_id);

let mint_nft_ix = spl_token::instruction::mint_to(
    token_program.key,
    mint_acct.key,
    final_acct.key,
    mint_authority_acct.key,
    &[],
    1,
)?;
invoke_signed(
    &mint_nft_ix,
    &[
        mint_acct.clone(),
        final_acct.clone(),
        mint_authority_acct.clone(), // signing authority
    ],
    &[&[b"mint_authority", &[mint_authority_bump_seed]]],
);
```
## Deploying

This is where Solana really shines.

Deploying the smart contract was hardly more difficult than compiling the Rust.

After following the instructions [Here](https://docs.solana.com/wallet-guide/cli) to create a wallet, I followed the instructions [Here](https://docs.solana.com/cli/deploy-a-program) to deploy my program and everything worked without any confusion.

After the setup the following works and returns a program address on the blockchain.
```
cargo build-bpf
solana program deploy ./output_path_from_cargo.so
```

Solana fees are extremely cheap at the time of writing, which is awesome for playing around with projects without eating $20 fees back and forth. It was moderately expensive (~$200) to deploy the contract to mainnet, but the transactions with it are extremely cheap.

Also, it blew my mind that you can UPDATE programs on the blockchain unless you've renounced that ability on launch. Calling the same deploy command again will update your binary, which is a killer feature. The fees for doing this are extremely low as well, you don't have to pay the larger deploy fee again.

Re-deploying costs an additional small fee, but like a normal transaction (<< $1) rather than another deploy.

## Front End

On the frontend we need to do the following:

- Connect to a user's wallet to sign transactions.
- Create the acount with the fee in it, owned by the program we want to pay.
- Create accounts for the token mint and metadata account, owned by the program.
- Construct a transaction that calls out to our program to mint the NFT
- Sign everything

This is all fairly straightforward using the Solana Web3 API, which is well documented [Here](https://solana-labs.github.io/solana-web3.js/)

We need to create the accounts in the frontend because to call the methods needed to mint the NFT you need an `AccountInfo`, and there doesn't currently seem to be a way to get an `AccountInfo` on the chain. You need to have it be passed in by the programming runtime as a result of having been included in the transaction.

Here's an example of creating the mint account. Intuitively I wanted this to be created by the program at runtime, but I wasn't able to figure a way around needing for the account to exist at runtime.

```
const mintAcct = new web3.Account(); // Generate keys for new account
const mintAcctIx = web3.SystemProgram.createAccount({
    programId: spl_token.TOKEN_PROGRAM_ID, // Owned by the token program
    space: spl_token.MintLayout.span,
    lamports: await connection.getMinimumBalanceForRentExemption(spl_token.MintLayout.span, 'singleGossip'),
    fromPubkey: initializerAccount.publicKey,
    newAccountPubkey: mintAcct.publicKey
});
```

Here `space` is how many bytes we're requesting, which needs to match up with what the consuming program expects, and influences the price of rent.

We derive the mint authority account referenced in rust as a PDA so everything matches up as expected and the program receives an `AccountInfo` to pass to the token program during the minting operation.

We also derive the address for the metadata account.

```
const signingAuthority = await web3.PublicKey.findProgramAddress([Buffer.from("mint_authority", 'utf8')], NFT_PROGRAM_ID);

let seeds = [
    Buffer.from("metadata", 'utf8'),
    TOKEN_METADATA_ID.toBuffer(),
    mintAcct.publicKey.toBuffer(),
];

const metadataAcct = await web3.PublicKey.findProgramAddress(seeds, TOKEN_METADATA_ID);
```

Once we've created all the accounts we grab everything 
```
const mintNFTTx = new web3.TransactionInstruction({
    programId: NFT_PROGRAM_ID,
    keys: [
        { pubkey: initializerAccount.publicKey, isSigner: true, isWritable: false },
        { pubkey: signingAuthority[0], isSigner: false, isWritable: false },
        { pubkey: tempTokenAccount.publicKey, isSigner: true, isWritable: true },
        { pubkey: CAT, isSigner: false, isWritable: true },
        { pubkey: CASHIER, isSigner: false, isWritable: true },
        { pubkey: spl_token.TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
        { pubkey: mintAcct.publicKey, isSigner: true, isWritable: true },
        { pubkey: web3.SYSVAR_RENT_PUBKEY, isSigner: false, isWritable: false },
        { pubkey: finalAcct.publicKey, isSigner: true, isWritable: false },
        { pubkey: TOKEN_METADATA_ID, isSigner: false, isWritable: false },
        { pubkey: metadataAcct[0], isSigner: false, isWritable: true },
        { pubkey: web3.SystemProgram.programId, isSigner: false, isWritable: false },
        //{ pubkey: dataStorage[0], isSigner: false, isWritable: false },
    ],
    data: Buffer.from(Uint8Array.of([new BN(1).toArray("le", 0)])), 
});
```

Remember we had that one byte of data to determine that we were calling the `MintNFT` instruction? There it is at the end being packed into a opaque blob of `[u8]`.

All that's left is to sign the transaction. The library didn't seem to like submitting the transaction without signatures for all the accounts we created, so we sign with everything.

```
 let signedTransaction = await provider.signTransaction(tx);
    signedTransaction.partialSign(tempTokenAccount);
    signedTransaction.partialSign(mintAcct);
    signedTransaction.partialSign(finalAcct);

    const serializedTransaction = signedTransaction.serialize()
    const signature = await connection.sendRawTransaction(
        serializedTransaction,
        { skipPreflight: false, preflightCommitment: 'singleGossip' },
    );
```

And that's it!

## Assets and Wrapup

Well, except we had to generate the art and metadata. 

I used python to generate broken looking punks by composing bit-fields with PIL, saving them to images on disk. I also had the program that generated them save a json file of properties as described by the [Metadata Docs](https://docs.metaplex.com/nft-standard).

But the metadata standard requires us to point to a URI containing the image location, as well as metadata about the NFT. We don't have either of those yet.

We do have images named `punk_0.png` through `punk_999.png`. We could store them on our server, but I wanted to store them somewhere forever™.

Some projects have used ipfs for this, I wanted to try out [Arweave](https://www.arweave.org/) which I'd heard good things about.

I uploaded these images to Arweave, using [arweave-deploy](https://github.com/ArweaveTeam/arweave-deploy#deploy-a-directory) to deploy a directory. This functionality seems to have been intended for deploying a website, so it maintains paths from the root of the folder, which is what I wanted.

I actually had to modify the source code for that tool slightly because it has a hard-coded limit of 200 items, and I had 1000 items. Nothing seemed to break but I'm not sure why the limit existed to begin with so modify at your own risk.

This deployment got us a root URL for our folder of deployed punk images. I then used another python script to produce the final `punk_0.json` through `punk_999.json` files with the image URI and metadata properties, bundled those into another folder, and uploaded that folder to Arweave as well.

Then I added that URI root to my rust code, redeployed my contract, and that was it.

!["Screenshot of several Glitch Punks visible in Phantom Wallet"](/assets/solana-wallet-screenshot.png)

You can view [Glitch Punk #1](https://solshop.io/asset/7SNKsaU5aXcU9eJfcdjtkmUoXvjs7424BUA6Pnxg52se/) on solshop.io.

## Conclusion

This was a fun project. I'm not 100% sure everything I did to hack the thing together is current best practice on Solana, but hopefully it's helpful to get an idea for what the shape of the code actually looks like.

I was helped a lot by this [Tutorial](https://paulx.dev/blog/2021/01/14/programming-on-solana-an-introduction/) by paul-schaaf on github, which gives an example of an escrow program.

The project is currently live on mainnet, if you want to purchase a Glitch Punk you can get one at [https://glitchpunks.art](https://glitchpunks.art)

Catherine


---
- 1: [Draw Some Pixels](https://www.pixel.rest/)
- 2: There are checked versions of subtraction available if you're doing something more complicated. You can get into trouble doing raw math if you don't think it through in smart contracts.