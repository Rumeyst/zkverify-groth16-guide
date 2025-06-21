# 🔐 zkVerify Proof Submission Guide (Groth16 + Relayer API)

> ✅ Working and tested tutorial for generating, proving, and submitting ZK proofs to [zkVerify](https://points.zkverify.io/loyalty) using Circom, SnarkJS, and the zkVerify Relayer API.

---

# Prerequisites

- Node.js v18+ (v20+ recommended)
- Circom + SnarkJS globally installed
- API key from zkVerify (default one included)

# ✅ Install Tools

```
npm install -g circom
sudo npm install -g snarkjs
```
# Project Setup
```
mkdir zkverify-relayer && cd zkverify-relayer
mkdir data real-proof
```
# Create Circuit: real-proof/sum.circom
```
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
# Compile & Generate Proof
1️⃣Compile Circuit
```
cd real-proof
circom sum.circom --r1cs --wasm --sym -o .
```
2️⃣ Powers of Tau Ceremony
```
snarkjs powersoftau new bn128 12 pot12_0000.ptau -v
snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau --name="My Contribution" -v
snarkjs powersoftau prepare phase2 pot12_0001.ptau pot12_final.ptau
```
📌 Note: Enter any random text when prompted (used as entropy).

3️⃣ Setup Groth16
```
snarkjs groth16 setup sum.r1cs pot12_final.ptau sum.zkey
```
4️⃣ Export Verification Key
```
snarkjs zkey export verificationkey sum.zkey verification_key.json
```
# Generate Proofs

1️⃣ Create Input File (input.json)
```
cd ..
cat > real-proof/input.json <<EOF
{
  "a": "3",
  "b": "11"
}
EOF
```
2️⃣ Generate Witness & Proof
```
cd real-proof
snarkjs wtns calculate sum.wasm input.json witness.wtns
snarkjs groth16 prove sum.zkey witness.wtns proof.json public.json
```
3️⃣ Move Files for API Submission
```
mv proof.json public.json verification_key.json ../data/
```
# Submit to zkVerify Relayer API

1️⃣ Install Dependencies
```
cd ..
npm init -y && npm pkg set type=module
npm install axios dotenv
```
2️⃣ Create .env File
```
echo "API_KEY=598f259f5f5d7476622ae52677395932fa98901f" > .env
```
✅ This API key is the default public key from the zkVerify docs.

👉 Need your own key? Request one from: [zkVerify Discord](https://discord.gg/k5cPGcUBY2) or the [official docs](https://points.zkverify.io/docs)

Verify creation:
```
cat .env
```
3️⃣ Create index.js
```
cat << 'EOF' > index.js
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

        // Poll for status
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
4️⃣ Run the Script
```
node index.js
```

---

##  Claim Your Points

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

🔁 change values in `real-proof/input.json`. to submit multiple proofs

---

##  Bonus Tips

* Submit feedback for extra 50 pts: [https://forms.gle/PVjhLkDt2TbgmspGA](https://forms.gle/PVjhLkDt2TbgmspGA)






