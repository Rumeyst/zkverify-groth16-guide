# zkverify-groth16-guide


# 🔐 zkVerify Proof Submission Guide (Groth16 + Relayer API)

> ✅ A 100% working and tested tutorial for generating, proving, and submitting ZK proofs to [zkVerify](https://points.zkverify.io/loyalty) using Circom, SnarkJS, and the zkVerify Relayer API.

---

## 📦 Prerequisites

- Node.js v18+ (v20+ tested)
- Circom + SnarkJS globally installed
- API key from zkVerify (default one included)

### ✅ Install Tools

```bash
npm install -g circom
sudo npm install -g snarkjs
````

---

## 🛠 Project Setup

```bash
mkdir zkverify-relayer && cd zkverify-relayer
mkdir data real-proof
```

### 🧩 Create Circuit: `real-proof/sum.circom`

```bash
cat > real-proof/sum.circom <<'EOF'
template SumCircuit() {
    signal input a;
    signal input b;
    signal output c;

    c <== a + b;
}

component main = SumCircuit();
EOF
```

---

## 🔧 Compile & Generate Proof

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

📌 Enter any random text when prompted (used as entropy).

### 3. Setup Groth16

```bash
snarkjs groth16 setup sum.r1cs pot12_final.ptau sum.zkey
```

### 4. Export Verification Key

```bash
snarkjs zkey export verificationkey sum.zkey verification_key.json
```

---

## 📊 Generate Proofs

### 1. Create Input File inside `real-proof`: `input.json`

```bash
cat > real-proof/input.json <<EOF
{
  "a": "3",
  "b": "11"
}
EOF
```

### 2. Generate Witness & Proof

```bash
snarkjs wtns calculate sum.wasm input.json witness.wtns
snarkjs groth16 prove sum.zkey witness.wtns proof.json public.json
```

### 3. Move Files for API Submission

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

### 2. Create `.env` File

```bash
echo "API_KEY=598f259f5f5d7476622ae52677395932fa98901f" > .env
```

✅ This API key is the **default public key** from the zkVerify docs.
👉 You can also request your **own API key** from the [zkVerify Discord](https://discord.gg/k5cPGcUBY2) or the [official docs](https://points.zkverify.io/docs).

Verify it was created:

```bash
cat .env
```

### 3. Create `index.js`

```bash
cat > index.js << 'EOF'
import axios from 'axios';
import fs from 'fs';
import dotenv from 'dotenv';
dotenv.config();

const API_URL = 'https://relayer-api.horizenlabs.io/api/v1';

const proof = JSON.parse(fs.readFileSync("./data/proof.json"));
const publicInputs = JSON.parse(fs.readFileSync("./data/public.json"));
const key = JSON.parse(fs.readFileSync("./data/verification_key.json"));

async function main() {
    const params = {
        proofType: "groth16",
        vkRegistered: false,
        proofOptions: {
            library: "snarkjs",
            curve: "bn128"
        },
        proofData: {
            proof,
            publicSignals: publicInputs,
            vk: key
        }
    };

    try {
        const response = await axios.post(`${API_URL}/submit-proof/${process.env.API_KEY}`, params);
        console.log("✅ Proof submitted:", response.data);

        const jobId = response.data.jobId;
        if (!jobId) {
            console.error("❌ No jobId returned.");
            return;
        }

        // Wait until finalized
        while (true) {
            const status = await axios.get(`${API_URL}/job-status/${process.env.API_KEY}/${jobId}`);
            console.log("🔁 Status:", status.data.status);
            if (status.data.status === "Finalized") {
                console.log("✅ Finalized!");
                console.log(status.data);
                break;
            }
            await new Promise(res => setTimeout(res, 5000));
        }
    } catch (err) {
        console.error("❌ Error submitting proof:", err.response?.data || err.message);
    }
}

main();
EOF
```

### 4. Run the Script

```bash
node index.js
```

---

## 🏁 Claim Your Points

1. Visit: [https://forms.gle/PVjhLkDt2TbgmspGA](https://forms.gle/PVjhLkDt2TbgmspGA)
2. Submit:

   * `jobId`
   * Proof Type: `Groth16`
   * Your EVM Wallet Address

---

## 🪙 Points Table

| Proofs Submitted | Points     |
| ---------------- | ---------- |
| 100              | 50         |
| 250              | 100        |
| 500              | 200        |
| **1000+**        | **500** 🚀 |

🔁 You can keep submitting by changing values in `real-proof/input.json`.

---

## 🧠 Bonus Tips

* Submit feedback for extra 50 pts: [https://forms.gle/PVjhLkDt2TbgmspGA](https://forms.gle/PVjhLkDt2TbgmspGA)

---

### ❤️ Built by [@rumeyst](https://github.com/rumeyst)


