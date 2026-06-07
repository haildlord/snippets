# Provider, Connection, Wallet, IDL(anchor)  Setup

```typescript
import {PublicKey, Connection, Keypair} from  "@solana/web3.js";
import {Program, AnchorProvider, Wallet} from "@coral-xyz/anchor";
import bs58 from "bs58";

import solana_idl from "../solana_idl/solana_smart_contracts.json" // <@ need IDL

    private connection : Connection;
    private wallet : Wallet;
    private provider : AnchorProvider;
    private program : Program;


    const solana_private_key = process.env.SOLANA_PRIVATE_KEY;
    const solana_rpc_url = process.env.SOLANA_RPC_URL; 
    const solana_private_key_byteArray = bs58.decode(solana_private_key);
    const solanKeyPair: Keypair = Keypair.fromSecretKey(solana_private_key_byteArray);
  
    this.wallet = new Wallet(solanKeyPair);
    this.connection = new Connection(solana_rpc_url, "confirmed");
    this.provider = new AnchorProvider(this.connection, this.wallet, { preflightCommitment: "confirmed" });
    this.program = new Program(solana_idl, this.provider);

```

# simple function call (RPC)
```typescript
    import {PublicKey, TransactionSignature} from  "@solana/web3.js";
    const statePda = PublicKey.findProgramAddressSync([Buffer.from("seed_given_in_smart_contract")], this.program.programId)[0];

    const tx = await this.program.methods.pauseProtocol().accounts({
        admin: this.wallet.publicKey,
        anyMoreState: this.statePda
    }).rpc(); // <@ inside rpc() :- recent blockhash inbuilt exists !

    console.log("some console log:", tx);

    return tx; // <@ retrun type is : Promise<TransactionSignature>, inside an async function
    
```

# Versioned function call -- very imp look closely -- you control the priority
```typescript
    import {ComputeBudgetProgram, TransactionMessage, VersionedTransaction, TransactionInstruction} from  "@solana/web3.js";

     // 1. Extract the raw instruction
    const ix : TransactionInstruction = await this.program.menthods.xYZFunction().accounts({ 
            admin: this.wallet.publickey, 
            anyMoreState: this.statePda
    }).instruction();

    // 2. Fetch network state and configure priority fees
    const { blockhash, lastValidBlockHeight } = await this.connection.getLatestBlockhash();
    const modifyComputeUnits = ComputeBudgetProgram.setComputeUnitLimit({ units: 10_000 }); // <@ this value is hardcode for examplery purpose below will show dynamically how to fetch, notice how await is not used here
    const addPriorityFee = ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 50_000 }); // <@ this value is hardcode for examplery purpose below will show dynamically how to fetch, notice how await is not used here

    // 3. Compile the V0 Message
    const messageV0 = new TransactionMessage({
        payerKey: this.wallet.publicKey,      
        recentBlockhash: blockhash,           
        instructions: [
            modifyComputeUnits,               
            addPriorityFee,                   
            ix                  
        ]
    }).compileToV0Message();

    // 4. Wrap the compiled message in a Versioned Transaction envelope
    let transaction = new VersionedTransaction(messageV0);

    // 5. SIGN THE TRANSACTION (The Missing Link)
    transaction = await this.wallet.signTransaction(transaction);

    // 6. Broadcast the signed bytes to the RPC
    const txSig = await this.connection.sendRawTransaction(transaction.serialize(), {
        skipPreflight: false,
        maxRetries: 3
    });

    console.log(txSig);

```
