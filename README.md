# poly-ii

```markdown
# Circom zk-SNARK Example

This repository contains an example of a simple zk-SNARK circuit using Circom. The example demonstrates the steps to compile the circuit, generate a proof, deploy a Solidity verifier contract, and verify the proof on a testnet.

## Requirements

- Node.js
- Circom
- snarkjs
- Hardhat
- An Ethereum wallet with Sepolia or Mumbai testnet funds

## Steps

### 1. Create the Circuit

Create a file named `circuit.circom` with the following content:

```circom
pragma circom 2.0.0;

template SimpleCircuit() {
    signal input A;
    signal input B;
    signal output C;

    C <== A + B;
}

component main = SimpleCircuit();
```

### 2. Compile the Circuit

Compile the circuit to generate intermediate files.

```sh
circom circuit.circom --r1cs --wasm --sym --c
```

This command generates the following files:
- `circuit.r1cs`
- `circuit.wasm`
- `circuit.sym`
- `circuit_cpp` directory

### 3. Generate Proving and Verification Keys

Generate the proving and verification keys:

```sh
snarkjs groth16 setup circuit.r1cs pot12_final.ptau circuit_0000.zkey
snarkjs zkey contribute circuit_0000.zkey circuit_final.zkey --name="First contribution"
snarkjs zkey export verificationkey circuit_final.zkey verification_key.json
```

### 4. Generate the Proof

Create an input file named `input.json` with the following content:

```json
{
    "A": 0,
    "B": 1
}
```

Generate the proof:

```sh
snarkjs groth16 prove circuit_final.zkey input.json proof.json public.json
```

### 5. Deploy the Verifier Contract to a Testnet

Generate the Solidity verifier contract:

```sh
snarkjs zkey export solidityverifier circuit_final.zkey verifier.sol
```

#### Using Hardhat

1. Initialize a new Hardhat project:

    ```sh
    npx hardhat init
    ```

2. Replace `contracts/Greeter.sol` with the `verifier.sol` file.

3. Add a deployment script in `scripts/deploy.js`:

    ```js
    async function main() {
        const Verifier = await ethers.getContractFactory("Verifier");
        const verifier = await Verifier.deploy();
        await verifier.deployed();
        console.log("Verifier deployed to:", verifier.address);
    }

    main()
        .then(() => process.exit(0))
        .catch(error => {
            console.error(error);
            process.exit(1);
        });
    ```

4. Deploy the contract:

    ```sh
    npx hardhat run scripts/deploy.js --network sepolia
    ```

### 6. Verify the Proof

After deploying, call the `verifyProof` method on the deployed contract. Create a file named `verify.js`:

```js
const { ethers } = require("hardhat");
const fs = require("fs");

async function main() {
    const verifierAddress = "YOUR_DEPLOYED_VERIFIER_ADDRESS"; // Replace with your deployed contract address
    const Verifier = await ethers.getContractFactory("Verifier");
    const verifier = Verifier.attach(verifierAddress);

    const proof = JSON.parse(fs.readFileSync("proof.json"));
    const publicSignals = JSON.parse(fs.readFileSync("public.json"));

    const callData = await snarkjs.groth16.exportSolidityCallData(proof, publicSignals);
    const argv = callData.replace(/["[\]\s]/g, "").split(',');

    const a = [argv[0], argv[1]];
    const b = [[argv[2], argv[3]], [argv[4], argv[5]]];
    const c = [argv[6], argv[7]];
    const inputs = argv.slice(8);

    const result = await verifier.verifyProof(a, b, c, inputs);
    console.log("Proof is", result ? "valid" : "invalid");
}

main()
    .then(() => process.exit(0))
    .catch(error => {
        console.error(error);
        process.exit(1);
    });
```

Run the verification script:

```sh
node verify.js
```

This script will output whether the proof is valid or invalid.

## Conclusion

This README provides a step-by-step guide to creating a zk-SNARK circuit in Circom, compiling it, generating a proof, deploying a verifier to a testnet, and verifying the proof. Follow these instructions to successfully complete the process.
```
