# zkverify-groth16-guide
A working guide for submitting zk proofs to zkVerify using Groth16 and Relayer API



---

````markdown
# 🔐 zkVerify Proof Submission Guide (Groth16 + Relayer API)

> ✅ A 100% working and tested tutorial for generating, proving, and submitting ZK proofs to [[zkVerify] https://points.zkverify.io/loyalty) using Circom, SnarkJS, and their Relayer API.
>


---

## 📦 Prerequisites

- Node.js v18+ (v20+ tested)
- Circom + SnarkJS globally installed
- API key from zkVerify docs

### Install Tools

```bash
npm install -g circom
npm install -g snarkjs
````

---

## 🛠 Project Setup

```bash
mkdir zkverify-relayer && cd zkverify-relayer
mkdir data real-proof
```

### Create a Circuit: `real-proof/sum.circom`

```circom
pragma circom 2.0.0;

template SumCircuit() {
    signal input a;
    signal input b;
    signal output c;
    c <== a + b;
}

component main = SumCircuit();
```

---

## 🧪 Compile & Generate Proof

### 1. Compile Circuit

```bash
cd real-proof
circom sum.circom --r1cs --wasm --sym -o .
```

### 2. Powers of Tau Ceremony

```bash
snarkjs powersoftau new bn128 12 pot12_0000.ptau -v
snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau --name="My Contribution" -v
snarkjs powersoftau prepare phase2 pot12_0001.ptau pot12_final.ptau
```

### 3. Setup Groth16

```bash
snarkjs groth16 setup sum.r1cs pot12_final.ptau sum.zkey
```

### 4. Export Verification Key

```bash
snarkjs zkey export verificationkey sum.zkey verification_key.json
```

---

## 🧮 Generate Proofs

### 1. Create Input: `input.json`

```json
{
  "a": "3",
  "b": "11"
}
```

### 2. Generate Witness & Proof

```bash
snarkjs wtns calculate sum.wasm input.json witness.wtns
snarkjs groth16 prove sum.zkey witness.wtns proof.json public.json
```

### 3. Move Files for Submission

```bash
mv proof.json public.json verification_key.json ../data/
```

---

## 🌐 Submit to zkVerify Relayer API

### 1. Install Dependencies

```bash
cd ..
npm init -y && npm pkg set type=module
npm install axios dotenv
```

### 2. Create `.env` file

```env
API_KEY=your_api_key_here
```

👉 Replace `your_api_key_here` with the one from zkVerify docs. >(598f259f5f5d7476622ae52677395932fa98901f)<

### 3. Create `index.js`

```js
import axios from "axios";
import fs from "fs";
import dotenv from "dotenv";
dotenv.config();

const API_URL = "https://relayer-api.horizenlabs.io/api/v1";
const proof = JSON.parse(fs.readFileSync("./data/proof.json"));
const publicInputs = JSON.parse(fs.readFileSync("./data/public.json"));
const key = JSON.parse(fs.readFileSync("./data/verification_key.json"));

async function main() {
  const params = {
    proofType: "groth16",
    vkRegistered: false,
    proofOptions: {
      library: "snarkjs",
      curve: "bn128",
    },
    proofData: {
      proof,
      publicSignals: publicInputs,
      vk: key,
    },
  };

  const res = await axios.post(`${API_URL}/submit-proof/${process.env.API_KEY}`, params);
  console.log("✅ Proof submitted:", res.data);

  if (res.data.optimisticVerify !== "success") {
    console.error("❌ Optimistic verification failed");
    return;
  }

  while (true) {
    const status = await axios.get(`${API_URL}/job-status/${process.env.API_KEY}/${res.data.jobId}`);
    console.log("🔁 Status:", status.data.status);
    if (status.data.status === "Finalized") {
      console.log("✅ Finalized!");
      console.log(status.data);
      break;
    }
    await new Promise((r) => setTimeout(r, 5000));
  }
}

main();
```

### 4. Run the Script

```bash
node index.js
```

---

## 🏁 Claim Your Points

1. Go to: (https://forms.gle/PVjhLkDt2TbgmspGA)
2. Fill in:

   * `jobId`
   * `Groth16`
   * Your EVM Wallet address

---


| Proofs Submitted | Points     |
| ---------------- | ---------- |
| 100              | 50         |
| 250              | 100        |
| 500              | 200        |
| **1000+**        | **500** 🚀 |

* Repeat proof generation with different inputs (just edit `input.json`)


---

## 🧠 Bonus Tips

* Submit feedback form:(https://forms.gle/PVjhLkDt2TbgmspGA) (+50 pts)
  

---

### ❤️ Built by [@rumeyst]

> GitHub: [github.com/rumeyst](https://github.com/rumeyst)

