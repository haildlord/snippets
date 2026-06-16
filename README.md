# dotenv config :
import dotenv from 'dotenv';
dotenv.config();
--- if above is not working replace it with ---
import 'dotenv/config';


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
    this.program = new Program(solana_idl as SolanaSmartContracts, this.provider);
     this.statePda = PublicKey.findProgramAddressSync([Buffer.from("lords_pot_state")], this.program.programId)[0];  // a simple pda

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


# Basic Deploy.ts
```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { PublicKey } from "@solana/web3.js";
import { 
  TOKEN_PROGRAM_ID, 
  ASSOCIATED_TOKEN_PROGRAM_ID, 
  getAssociatedTokenAddressSync 
} from "@solana/spl-token"; // > need to install
import {SolanaSmartContracts} from "../target/types/solana_smart_contracts" // > get the type of the smart contract

configDotenv();


module.exports = async function (provider: anchor.AnchorProvider) {
    anchor.setProvider(provider);
    const program = anchor.workspace.SolanaSmartContracts as Program<SolanaSmartContracts>;
    console.log(`[Deploy]: Target Program ID: ${program.programId.toBase58()}`);
    console.log(`[Deploy]: Signer/Admin Authority: ${provider.wallet.publicKey.toBase58()}`);  // is gonnna be default : H8Q7CUvPigtSxfd13TKRuFrwdJtc6pJu9BMNhbXF9yAY

    // Your wallet you wanna test could be fetched this way
    const solana_private_key_byteArray = bs58.decode(`${process.env.SOLANA_PRIVATE_KEY}`);
    const solanKeyPair: Keypair = Keypair.fromSecretKey(solana_private_key_byteArray);

    // PDA
    const [lordsPotStatePda] = PublicKey.findProgramAddressSync(
     [Buffer.from("seed_provided_in_smart_contract")],
     program.programId
    );

    // Associated Token Account (ATA) owned by our Vault Authority PDA
    const vaultUsdcAccount = getAssociatedTokenAddressSync(
        DEVNET_USDC_MINT,
        vaultAuthorityPda,
        true, // Allow owner to be a PDA, false if owner is a Keypair
        TOKEN_PROGRAM_ID
    );

    // How to send a tx :
      const tx = await program.methods
      .initialize(NORMAL_MAX, BONUS_MAX, TICKET_PRICE) // <@ params to send
      .accounts({      // below are the accounts you wanna send
        tokenProgram: TOKEN_PROGRAM_ID
      }).signers([ // <@ signers need to be added if you wanna sign ur tx using ur privatekey or else remove the signers section, rpc behind the scens using provider will put default wallet to sign it
        solanKeyPair
      ])
      .rpc();

    // How to fetch state & view the values in it :
    try {
        // 3. Double-check if the contract state has already been initialized
        const stateAccount = await program.account.lordsPotState.fetch(lordsPotStatePda);
        console.log(`[Deploy]: Protocol already initialized! Admin is currently: ${stateAccount.admin.toBase58()}`);
        console.log(`[Deploy]: Protocol already initialized! Admin is currently: ${stateAccount.bonusMax}`);
    } catch (err) {
        console.log("[Deploy]: PDA state not found. Executing fresh initialization transaction...");
    }

}
```

# Setting Custom Signer as the Default Signer either do Versioned or Below :
```typescript
  import {SolanaSmartContracts} from "../target/types/solana_smart_contracts";

  // 1. Establish the default workspace provider context
  anchor.setProvider(provider);

  // 2. Extract the compiled IDL from the workspace before the swap
  const workspaceProgram = anchor.workspace.SolanaSmartContracts;
  const idl = workspaceProgram.idl;

  // 3. Decode your specific private key from your backend environment variables
  const solana_private_key_byteArray = bs58.decode(`${process.env.SOLANA_PRIVATE_KEY}`);
  const solanaKeyPair: Keypair = Keypair.fromSecretKey(solana_private_key_byteArray);

  // 4. Wrap your custom keypair into a new wallet and provider session
  const customWallet = new anchor.Wallet(solanaKeyPair);
  const customProvider = new anchor.AnchorProvider(
    provider.connection, 
    customWallet, 
    anchor.AnchorProvider.defaultOptions()
  );

  // 5. Re-bind the global anchor context to your elite custom provider
  anchor.setProvider(customProvider);

  // 6. Instantiate the program instance using the modern two-argument signature
  const program = new Program<SolanaSmartContracts>(idl, customProvider);

  console.log("--------------------------------------------------");
  console.log(`[Deploy]: Target Program ID: ${program.programId.toBase58()}`);
  console.log(`[Deploy]: New Fee Payer / Admin Authority: ${customProvider.wallet.publicKey.toBase58()}`);
  console.log("--------------------------------------------------");
```

# you can even create new program to attach differetnt sol payers 
```typescript
  const buyerProgram = new Program<SolanaSmartContracts>(idl, provider);
  const buyer = provider.wallet;
 
  const buyTx = await buyerProgram.methods
  .buyTicket(tickets_to_buy)
  .accounts({
    signer : buyer.publicKey,
    tokenProgram: TOKEN_PROGRAM_ID
  })
  .rpc();

  console.log(`[Success]: LordsPot Protocol boought ticket! Transaction: ${buyTx}`);
```

# surfpool testing :
```typescript
    "build:program": "anchor build && cp ./target/idl/solana_smart_contracts.json ../backend/solana_idl",         // @> command so that whenever you build something it will be available anywhere in the folder
    "start:surfpool": "touch surfnet-state.sqlite && cp surfnet-state.sqlite surfnet-temp.sqlite && surfpool start --rpc-url \"$(grep SOLANA_RPC_URL ./.env | cut -d '=' -f2-)\" --watch --db ./surfnet-temp.sqlite",// <@ copy old state of surfpool to the new and test things
    "commit:surfpool": "cp surfnet-temp.sqlite surfnet-state.sqlite",  // <@ above updated surfpool will be saved in the                                                                                      
    "deploy:surfpool": "anchor program deploy --provider.cluster http://127.0.0.1:8899",
    "migrate:surfpool": "anchor migrate --provider.cluster http://127.0.0.1:8899"
```


# how to deSearalize the events from the solana blockchain, thrown at ur node.js script :
```typescript
import { PublicKey } from "@solana/web3.js";
import IDL from "../solana_idl/solana_smart_contracts.json";
const PROGRAM_ID = new PublicKey("6MCjqsDP4zjxxg2AWCrjDGeKYUiWL3xpG2ccUxLXaMB9");

// usign idl as dictionary knowing how our smart contract is typed, its variables, logs, etc..
const coder = new BorshCoder(IDL as any);
// gives you json, filtered by programId & using the above type dictionry
const eventParser = new EventParser(PROGRAM_ID, coder);
```


# Prisma

1) docker run -d --name solana-postgress-sql -p 5432:5432 -e POSTGRES_PASSWORD=admin@123 -e POSTGRES_DB=lordspot_db postgres

2) installation :
    npm install prisma --save-dev
    npm install @prisma/client
    npx prisma init

3) view Database UI :
    npx prisma studio

4) add this in .env file -> DATABASE_URL="postgresql://postgres:admin@123@localhost:5432/lordspot_db?schema=public"

5) after writing schema run -> `npx prisma migrate dev --name init_ticket_schema` to generate the sql query & get it executed in the database both at same time

```typescript
6) import { PrismaClient } from "../generated/prisma/client";
    const prisma = new PrismaClient();

    try {
    // 8. Bulk insert everything into PostgreSQL instantly
    const inserted = await prisma.ticket.createMany({
        data: [], // array of json with column names as key.
        skipDuplicates: true // Safety net: don't crash if a unique ID collides, but unique id is database generated remove it
    });
    }catch(err){
        throw error
    }

```


## Field Attributes (prefixed with `@`)

| Attribute | Description |
|---|---|
| `@id` | Defines a single-field primary key |
| `@default(...)` | Sets a default value |
| `@unique` | Enforces uniqueness on a field |
| `@map("...")` | Maps field to a different database column name |
| `@relation(...)` | Defines a relation between models |
| `@db.*` | Native database type attributes (e.g. `@db.VarChar(x)`, `@db.Text`, `@db.Boolean`) |

[[Defining Attributes](https://www.prisma.io/docs/orm/prisma-schema/data-model/models#defining-attributes)]

---

## Block Attributes (prefixed with `@@`)

| Attribute | Description |
|---|---|
| `@@unique([...])` | Enforces uniqueness across multiple fields |

[[Defining Attributes](https://www.prisma.io/docs/orm/prisma-schema/data-model/models#defining-attributes)]

---

## `@default()` Functions

| Function | Notes |
|---|---|
| `autoincrement()` | Relational databases only |
| `cuid()` | Generates a cuid string |
| `uuid()` | Generates a UUID string |
| `ulid()` | Generates a ULID string |
| `auto()` | MongoDB only (generates ObjectId) |
| `now()` | Not in sources but commonly used |

[[`@id` Examples](https://www.prisma.io/docs/orm/reference/prisma-schema-reference#examples-6)]

---

## Scalar Field Types

| Prisma Type | Description | JS Type |
|---|---|---|
| `String` | Variable length text | `string` |
| `Boolean` | True or false | `boolean` |
| `Int` | Integer | `number` |
| `BigInt` | Large integer | `bigint` |
| `Float` | Floating point | `number` |
| `Decimal` | Precise decimal | `Decimal` |
| `DateTime` | Date and time | `Date` |
| `Json` | JSON value | `object` |
| `Bytes` | Binary data | `Buffer` |

[[Scalar Types Reference](https://www.prisma.io/docs/orm/reference/prisma-schema-reference#model-field-scalar-types)]

---

## Type Modifiers

| Modifier | Meaning |
|---|---|
| `?` | Optional field (nullable) |
| `[]` | List/array field |

[[Defining Fields](https://www.prisma.io/docs/orm/prisma-schema/data-model/models#defining-fields)]

---

## Native Database Type Attributes (`@db.*`) — PostgreSQL Examples

| Prisma Type | Native Attribute |
|---|---|
| `String` | `@db.Text`, `@db.VarChar(x)`, `@db.Char(x)`, `@db.Uuid`, `@db.Xml`, `@db.Inet` |
| `Boolean` | `@db.Boolean` |
| `Int` | `@db.Int`, `@db.SmallInt` |
| `BigInt` | `@db.BigInt` |
| `Float` | `@db.Real`, `@db.DoublePrecision` |
| `Decimal` | `@db.Decimal(x,y)`, `@db.Money` |
| `DateTime` | `@db.Timestamp`, `@db.Date`, `@db.Timestamptz(x)` |
| `Json` | `@db.Json`, `@db.JsonB` |
| `Bytes` | `@db.ByteA` |

1. Why did the database need to be running?
You are completely right that it pushed no data (no lottery tickets). But it did push the structure.
In databases, there are two types of SQL commands:
DDL (Data Definition Language): Things like CREATE TABLE. This builds the empty shelves.
DML (Data Manipulation Language): Things like INSERT INTO. This puts the boxes on the shelves.
When you ran prisma migrate dev, Prisma did not just write that migration.sql file on your laptop. It immediately took that SQL, walked through Door 5432, and executed it inside your Docker database. If Docker was turned off, Prisma would have thrown a massive error saying: "I wrote the SQL, but I can't find the database to actually build the tables!" Right now, inside your Docker container, a perfectly formatted, empty Ticket table is officially sitting there waiting for data.

2. Will future models go into the same migration.sql file?
No, they will not! And this is why Prisma is so powerful.
Prisma Migrations act exactly like Git Commits for your database.
Think of what you just did as your "Initial Commit." You named it init_ticket_schema, so Prisma created a folder with a timestamp (something like 20240510123000_init_ticket_schema) and put the SQL for the Ticket table inside it.
If tomorrow your boss says, "Hey, we need to add a User Profile table!", here is what will happen:
You will add model User { ... } to your schema.prisma file.
You will run npx prisma migrate dev --name add_user_profile.
Prisma will look at the database and say, "Ah, the Ticket table is already there. I only need to build the User table."
It will create a brand new folder named add_user_profile with a second migration.sql file containing only the CREATE TABLE "User" command.

