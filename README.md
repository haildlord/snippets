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
    private statePda: PublicKey;


    const solana_private_key = process.env.SOLANA_PRIVATE_KEY;
    const solana_rpc_url = process.env.SOLANA_RPC_URL; 
    const solana_private_key_byteArray = bs58.decode(solana_private_key);
    const solanKeyPair: Keypair = Keypair.fromSecretKey(solana_private_key_byteArray);
  
    this.wallet = new Wallet(solanKeyPair);
    this.connection = new Connection(solana_rpc_url, "confirmed");
    this.provider = new AnchorProvider(this.connection, this.wallet, { preflightCommitment: "confirmed" });
    this.program = new Program(solana_idl, this.provider);
    this.statePda = PublicKey.findProgramAddressSync([Buffer.from("lords_pot_state")], this.program.programId)[0];

```
