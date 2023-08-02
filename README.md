# Semaphore Merkle Tree Batcher (semaphore-mtb) trusted setup ceremony

[World ID](https://whitepaper.worldcoin.org/proof-of-personhood) is a privacy-preserving proof of personhood protocol that leverages the [Semaphore protocol](https://semaphore.appliedzkp.org/) in order to prove inclusion of members in a merkle tree where each leaf was verified using some method (e.g. [the orb](https://whitepaper.worldcoin.org/technical-implementation)), inserted into the tree and where its corresponding private key is self-custodied by each member. In the Semaphore protocol implementation there are individual sequential insertions. Individual insertions for big merkle trees are very expensive and therefore economically unfeasible onchain operations at the World ID userbase scale (user count can be found [here](https://worldcoin.org/)). In order to scale our efforts we developed the [Semaphore Merkle Tree Batcher](https://github.com/worldcoin/semaphore-mtb) (SMTB) which is composed of custom zero-knowledge circuits written in [gnark](https://github.com/ConsenSys/gnark) in order to do batch insertions into the Semaphore merkle tree. This gnark circuit leverages the groth16 proof system on the bn254 curve and requires a custom trusted setup ceremony to be made in order to achieve verifier soundness.

If you want to read up more on what is a trusted setup and why it is a requirement for [zkSNARKs](https://zkhack.dev/2023/07/27/getting-started-in-zk/) (zero-knowledge proofs) that are non-universal like [groth16](https://eprint.iacr.org/2016/260) which is the proof system powering Semaphore and World ID, read:

- [Understanding Trusted Setups: A Guide - Panther Protocol](https://blog.pantherprotocol.io/a-guide-to-understanding-trusted-setups/#:~:text=A%20trusted%20setup%20is%20a,similar%20cryptographic%20protocols%20rely%20on.)
- [How do trusted setups work? - Vitalik Buterin](https://vitalik.ca/general/2022/03/14/trustedsetup.html)
- [On-Chain Trusted Setup Ceremony - a16z crypto](https://a16zcrypto.com/posts/article/on-chain-trusted-setup-ceremony/#section--1)

### Ceremony Specification

For the phase 1 contribution (a.k.a powers of tau ceremony) we are using the [Perpetual Powers of Tau ceremony](https://github.com/privacy-scaling-explorations/perpetualpowersoftau) (up to the 54th contribution) through the AWS S3 hosted bucket in the [snarkjs repo README](https://github.com/iden3/snarkjs/blob/master/README.md#7-prepare-phase-2). We built a [deserializer](https://github.com/worldcoin/ptau-deserializer) from the `.ptau` format into the `.ph1` format used by gnark and initialized a phase 2 using a fork ([semaphore-mtb-setup](https://github.com/worldcoin/semaphore-mtb-setup)) of a ceremony coordinator wrapper on top of gnark built by the [zkbnb team](https://github.com/bnb-chain/zkbnb-setup).

#### System used

[AWS m5.16xlarge](https://aws.amazon.com/ec2/instance-types/m5/) instance

- 256 GiB RAM
- 64 cores
- 500 GiB Volume

### Pre-Contribution

The chain of comands that has been performed before the first contribution.

```bash
git clone https://github.com/worldcoin/semaphore-mtb-setup
git clone https://github.com/worldcoin/semaphore-mtb
```

Download the trusted setup ceremony coordinator tool and the powers of tau files.

```bash
cd semaphore-mtb-setup && go build -v

# Download Powers of Tau files for each respective circuit
wget https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_20.ptau
mv powersOfTau28_hez_final_20.ptau 20.ptau

wget https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_23.ptau
mv powersOfTau28_hez_final_23.ptau 23.ptau

wget https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_26.ptau
mv powersOfTau28_hez_final_26.ptau 26.ptau

# Convert .ptau format into .ph1
./semaphore-mtb-setup p1i 20.ptau 20.ph1
./semaphore-mtb-setup p1i 23.ptau 23.ph1
./semaphore-mtb-setup p1i 26.ptau 26.ph1

# go up a folder
cd ../
```

Generate r1cs representation of the necessary sizes of the Semaphore Merkle Tree Batcher (SMTB):

- Tree depth: 30, Batch size: 10
  - constraints: 725439
  - powers of Tau needed: 20
- Tree depth: 30, Batch size: 100
  - constraints: 6305289
  - powers of Tau needed: 23
- Tree depth: 30, Batch size: 1000
  - constraints: 60375789
  - powers of Tau needed: 26

```bash
cd semaphore-mtb && go build -v

# requires quite a bit of compute
./gnark-mbu r1cs --batch-size 10 --tree-depth 30 --output b10t30.r1cs
./gnark-mbu r1cs --batch-size 100 --tree-depth 30 --output b100t30.r1cs
./gnark-mbu r1cs --batch-size 1000 --tree-depth 30 --output b1000t30.r1cs

# move the r1cs files into the coordinator folder
mv b10t30.r1cs b100t30.r1cs b1000t30.r1cs ../semaphore-mtb-setup/

# go up a folder
cd ../
```

Initialize the phase 2 of the setup (the slowest process):

```bash
# make folders for each phase 2
mkdir b10 b100 b1000

cd semaphore-mtb-setup

# initialize each respective phase2
./semaphore-mtb-setup p2n 20.ph1 b10t30.r1cs b10t30c0.ph2

mv b10t30c0.ph2 srs.lag evals ../b10/

./semaphore-mtb-setup p2n 23.ph1 b100t30.r1cs b100t30c0.ph2

mv b100t30c0.ph2 srs.lag evals ../b100/

./semaphore-mtb-setup p2n 26.ph1 b1000t30.r1cs b1000t30c0.ph2

mv b1000t30c0.ph2 srs.lag evals ../b1000/
```

### The Contribution Process

#### Requirements

- Go (using 1.20.5)
- Git
- \> 16 GiB RAM
- Good connectivity (to upload and download files fast to s3)
- The more cores the better (shorter contribution time)
- \> 10 GiB storage

#### Steps

Download the corresponding files for contributions (presigned AWS S3 bucket urls)

Contribution time (on aws m5.16xlarge 256GiB RAM <4GB used - the more cores the better):
b10: 5-10sec (50MB file)
b100: < 5 min (419MB file)
b1000: 20-30 min (3.5GB file)

You will receive pre-signed URLs from [dcbuilder.eth](https://twitter.com/DCbuild3r) for a GET request of a .ph2 file off of AWS S3 of the form:

> https://<S3_BUCKET>.s3.amazonaws.com/<FILENAME>?AWSAccessKeyId=<ACCESS_KEY_ID>&Signature=<SIGNATURE>&Expires=<EXPIRY_TIME>

Submit a GET request using:

```bash
curl --output b10t30cXX.ph2 <PRESIGNED_URL>
curl --output b100t30cXX.ph2 <PRESIGNED_URL>
curl --output b1000t30cXX.ph2 <PRESIGNED_URL>
```

where XX is the current contribution number.

Download and build the [`semaphore-mtb-setup`](https://github.com/worldcoin/semaphore-mtb-setup) coordinator tool to perform the contribution:

```bash
git clone https://github.com/worldcoin/semaphore-mtb-setup
cd semaphore-mtb-setup
go build -v
```

Perform the contribution for each individual .ph2 file and increase the XX counter by one. Each command will output a contribution hash, please copy each of these down into a file of the format `<NAME/PSEUDONYM>_CONTRIBUTION.txt` and prepend each value with the corresponding batch size of the .ph2 file you contributed to (b10, b100 or b1000). Please also share via a message what NAME or PSEUDONYM you selected since it is required to generate a pre-signed S3 upload URL.

```bash
./semaphore-mtb-setup p2c b10t30cXX.ph2 b10t30c(XX + 1).ph2
```

```bash
./semaphore-mtb-setup p2c b100t30cXX.ph2 b100t30c(XX + 1).ph2
```

```bash
./semaphore-mtb-setup p2c b100t30cXX.ph2 b100t30c(XX + 1).ph2
```

You will also receive pre-signed URLs to upload your contribution to the S3 bucket, after your contributions are done and you have the output files, upload them using the following commands:

```bash
curl -v -T b10t30c(XX + 1).ph2 <PRESIGNED_URL>
curl -v -T b100t30c(XX + 1).ph2 <PRESIGNED_URL>
curl -v -T b1000t30c(XX + 1).ph2 <PRESIGNED_URL>
curl -v -T <NAME/PSEUDONYM>_CONTRIBUTION.txt <PRESIGNED_URL>
```

> NOTE: if your file is above 5GiB (shouldn't ever get to that) the request will fail. If that happens, please reach out.

Congratulations! You have successfully contributed to our phase 2 trusted setup ceremony!

# List of contributors

### Batch size 10 SMTB circuit

1. [**dcbuilder.eth**](https://twitter.com/DCbuild3r)

- contribution hash: `b0b44102bf1201e83bffb1cb0c492cfb93421656c4d2d113840bb904a48936c5`
- generated file: `b10t30c01.ph2`

2. [**reldev**](http://twitter.com/reldev)

- contribution hash: `ed4570a3668448102b8f0fa2c47cdd592281a7dd30e568f112ec784b5f1d96e2`
- generated file: `b10t30c02.ph2`

3. [**remco**](https://twitter.com/recmo)

- contribution hash: `1a0925e14d4c3035d8ee5d288e376e723299b712181b4e34b59a337ebe08761a`
- generated file: `b10t30c03.ph2`

4. [**worldbridger**](https://twitter.com/shumochu)

- contribution hash: `c3092354da6c45b43a81e410364972d635ee563d46c24db77e1dc171f440a9ca`
- generated file: `b10t30c04.ph2`

5. [**kobigurk**](https://twitter.com/kobigurk)

- contribution hash: `d4e581570bbe53ffa9e08cc93816f48486cfd756df50894de74ac1845ae07feb`
- generated file: `b10t30c05.ph2`

6. [**kustosz**](https://twitter.com/mmkostrzewa)

- contribution hash: `7f13b7d526ec05775c9646908da8d99a35f48777fa341f8c25adf099135bb162`
- generated file: `b10t30c06.ph2`

7. [**m1guelpf**](https://twitter.com/m1guelpf)

- contribution hash: `e4ce787995a6c97b8be12f418ae3c026e26f84ad3d433e3fe6f33d3449d08c91`
- generated file: `b10t30c07.ph2`

8. [**atris**](https://twitter.com/atris_eth)

- contribution hash: `807552014597a0c51e4e0f2e9b989be7633d8a050906a894cab226bc2a85333d`
- generated file: `b10t30c08.ph2`

9. [**zellic**](https://twitter.com/zellic_io)

- contribution hash: `e0e690b8acf0dfd8bed0c65c55c1507fd261ac5eb86ef7f111d04b9a35b81587`
- generated file: `b10t30c09.ph2`

10. [**eddylazzarin**](https://twitter.com/eddylazzarin)

- contribution hash: `43c7cb5dd53447967cfe531ac2120948efd90fa5c80eb3b80fb340df452d7e18`
- generated file: `b10t30c10.ph2`

11. [**emilianobonassi**](https://twitter.com/emilianobonassi)

- contribution hash: `d60c8e2b3ceb98c6807f9e541eb163a011cca54ff9d859c5147503e47dcdbc8b`
- generated file: `b10t30c11.ph2`

12. [**leo21.sismo.eth**](http://twitter.com/leo21_eth)

- contribution hash: `6ae61f6ba229efdd93b7b98e80b5ecf2424038f1fa34e1d941b4ebab15a8d6d8`
- generated file: `b10t30c12.ph2`

13. [**yisun**](https://twitter.com/theyisun)

- contribution hash: `322bd95fe69c2b82e7ecbed4eeed3144869c3ae4a151bb39259d7ead1777d44b`
- generated file: `b10t30c13.ph2`

14. [**dzejkop**](https://github.com/dzejkop)

- contribution hash: `2b85850b40ce8659c932e3e829dc21baf79ea8e2c207f40a3bb0257f3e5cb7de`
- generated file: `b10t30c14.ph2`

### Batch size 100 SMTB circuit

1. [**dcbuilder.eth**](https://twitter.com/DCbuild3r)

- contribution hash: `66e1845efd543078218a92fd575478bd2f71d64b16a3bf427f5032dcc479a808`
- generated file: `b100t30c01.ph2`

2. [**reldev**](http://twitter.com/reldev)

- contribution hash: `b136004f834e6a4db35a5def1050d2aed0b4d41df57cbeb4577eb5c5997f9995`
- generated file: `b10t30c02.ph2`

3. [**remco**](https://twitter.com/recmo)

- contribution hash: `be313375479fa2ce7097129bb187f16df3211dba4e395acdd84e946fb4ae1e4b`
- generated file: `b100t30c03.ph2`

4. [**worldbridger**](https://twitter.com/shumochu)

- contribution hash: `30fac7bc6ff5f2bc24991dd5e6537e203714459c1e6d31c241d512490e966dde`
- generated file: `b100t30c04.ph2`

5. [**kobigurk**](https://twitter.com/kobigurk)

- contribution hash: `e53e7eae98d270b44e25ac8b413e1dcf45a13062c51cb45e6faad089a25d8511`
- generated file: `b100t30c05.ph2`

6. [**kustosz**](https://twitter.com/mmkostrzewa)

- contribution hash: `c202ab3e6a3e29041b1e3781cb1ffdbae7683541d32a91d8d92454b2ce0508c2`
- generated file: `b100t30c06.ph2`

7. [**m1guelpf**](https://twitter.com/m1guelpf)

- contribution hash: `10b5376bfbe4e6235a8b7f9b000699063ee8c5c9626e98dc70ed594f5c2e0326`
- generated file: `b100t30c07.ph2`

8. [**atris**](https://twitter.com/atris_eth)

- contribution hash: `e2b20b3379496da808a502df477b47846e7c6375a40e84e81730cf5d15142e44`
- generated file: `b100t30c08.ph2`

9. [**zellic**](https://twitter.com/zellic_io)

- contribution hash: `6ac522c33145afcbcd05291834f13b0b83833578ca1a13e4ef91a33a187f23e2`
- generated file: `b100t30c09.ph2`

10. [**eddylazzarin**](https://twitter.com/eddylazzarin)

- contribution hash: `3a4112517f6b9082c6d8faf1091164bcd4f818aeb7e53cb8ec2f5aa1eabdd05d`
- generated file: `b100t30c10.ph2`

11. [**emilianobonassi**](https://twitter.com/emilianobonassi)

- contribution hash: `8840c48bf70b565fc978cc893f42d8bd597b6904b4d1cd4b48048a39abda6e32`
- generated file: `b100t30c11.ph2`

12. [**yisun**](https://twitter.com/theyisun)

- contribution hash: `491a46c5aca878bd44f2d7d3b940128127eb705cabc6cedb48de74b12a85bf87`
- generated file: `b100t30c12.ph2`

13. [**leo21.sismo.eth**](http://twitter.com/leo21_eth)

- contribution hash: `7b13d503f70b5a1c899a5390c51e46d0a2194c1b0d97427ff2f6b074ee2263eb`
- generated file: `b1000t30c13.ph2`

14. [**dzejkop**](https://github.com/dzejkop)

- contribution hash: `f77f2e1cf197b09084c5b16ae0d2a61306ccf4cc01cfa9bab90d2b72ef071a24`
- generated file: `b100t30c14.ph2`

### Batch size 1000 SMTB circuit

1. [**dcbuilder.eth**](https://twitter.com/DCbuild3r)

- contribution hash: `6a1d0b08bf79fe564cd701e9af70bb5f3936f668869f529e83d8ecb1a3d474b3`
- generated file: `b1000t30c01.ph2`

2. [**reldev**](http://twitter.com/reldev)

- contribution hash: `d263eafd25ba809748390850966fd689379311033de6d4fc2de69acebb247d2b`
- generated file: `b10t30c02.ph2`

3. [**remco**](https://twitter.com/recmo)

- contribution hash: `1b5bb5f54d1126398b723ae4f42f86d9659550d57af262e67b9f8a6ddc7d926e`
- generated file: `b1000t30c03.ph2`

4. [**worldbridger**](https://twitter.com/shumochu)

- contribution hash: `9155ac2cc06510ffb63ac9a8c01284bf533183542054a94d735d358cc42cdca4`
- generated file: `b1000t30c04.ph2`

5. [**kobigurk**](https://twitter.com/kobigurk)

- contribution hash: `463c17a0c8ed79c56f4579d1eef02beac8edc91549fc9abae51c08e76b8bb4e9`
- generated file: `b1000t30c05.ph2`

6. [**kustosz**](https://twitter.com/mmkostrzewa)

- contribution hash: `dc8992e6d8bf5f9ed4b0cf9e98f582c0594d5775f315446c7ed1e94d3020e787`
- generated file: `b10t30c06.ph2`

7. [**m1guelpf**](https://twitter.com/m1guelpf)

- contribution hash: `1c512817cc55404489dc0184d7d049e6906f8282c542c17d9817a2bdab8ab524`
- generated file: `b1000t30c07.ph2`

8. [**atris**](https://twitter.com/atris_eth)

- contribution hash: `606a17d757c56171416c81dcd79171369e6ff651020d0dba0d7c98fa31ce78ca`
- generated file: `b1000t30c08.ph2`

9. [**zellic**](https://twitter.com/zellic_io)

- contribution hash: `dca70be8175332c8f3afe78018eecb5c205fef05fc0f177577dfed3bdb562304`
- generated file: `b1000t30c09.ph2`

10. [**eddylazzarin**](https://twitter.com/eddylazzarin)

- contribution hash: `34e92e61360b5e36d44736a8b2b6687c7557985e0acccd770b63de1b710c4506`
- generated file: `b1000t30c10.ph2`

11. [**emilianobonassi**](https://twitter.com/emilianobonassi)

- contribution hash: `40cecbd461f4cbb275c0a72c182f4481cfcedb5c0b72026ad84b0a007232cbf4`
- generated file: `b1000t30c11.ph2`

12. [**leo21.sismo.eth**](http://twitter.com/leo21_eth)

- contribution hash: `4c29e019d35c233c1e0f11953293dd42f567ab3d1949afdc4831f8b5c3179c38`
- generated file: `b1000t30c12.ph2`

13. [**yisun**](https://twitter.com/theyisun)

- contribution hash: `7dd5975f1eabee3b51b3e9d2890d558290c55f7cef502cc1fd66fb4e61cc4431`
- generated file: `b1000t30c13.ph2`

14. [**dzejkop**](https://github.com/dzejkop)

- contribution hash: `ba4fc054df53635a5efc9aac72e9de45a910adebd7da4f67aecd76e22e6ae9cc`
- generated file: `b1000t30c14.ph2`

### Ceremony integrity check

Now that the ceremony is done, we need to verify that all contributions were performed correctly and that `semaphore-mtb-setup` integrity tests pass. To do so we run:

```bash
./semaphore-mtb-setup p2v b10t30c14.ph2 b10t30c0.ph2
./semaphore-mtb-setup p2v b100t30c14.ph2 b10t30c0.ph2
./semaphore-mtb-setup p2v b1000t30c14.ph2 b10t30c0.ph2
```

The zeroeth contribution (c0) is the file that is generated after initializing the phase 2 from the phase 1 like described in the **pre-contribution** section. We ran these integrity tests and all passed!

![](https://hackmd.io/_uploads/ryKHuRfYh.png)

![](https://hackmd.io/_uploads/BJxnrORft2.png)

![](https://hackmd.io/_uploads/rkl8O0fFh.png)

### Extracting proving and verifying keys

After we verify everything went correctly we extract the proving and verifying keys from the setup using the following commands:

```bash
./semaphore-mtb-setup key b10t30c14.ph2
./semaphore-mtb-setup key b100t30c14.ph2
./semaphore-mtb-setup key b1000t30c14.ph2
```

> Note: In order to run the `key` command we need the respective `evals` and `srs.lag` files we generated in the **pre-contribution phase** to in the directory we execute `semaphore-mtb-setup` from.

The output of these commands are going to be `pk` and `vk` files for each respective circuit.

### Testing semaphore-mtb end-to-end

Now that we have `pk` and `vk` files for each circuit, let's generate a `semaphore-mtb` gnark proving system, generate some test parameters, generate a proof and test whether they verify!

Now let's generate the gnark proving systems from each of the respective `vk` and `pk` pairs:

```bash
# Clone and build semaphore-mtb, then:
./gnark-mbu import-setup --batch-size 10 --tree-depth 30 --pk ../b10/pk --vk ../b10/vk --output ../b10/ps
./gnark-mbu import-setup --batch-size 100 --tree-depth 30 --pk ../b100/pk --vk ../b100/vk --output ../b100/ps
./gnark-mbu import-setup --batch-size 1000 --tree-depth 30 --pk ../b1000/pk --vk ../b1000/vk --output ../b1000/ps
```

`ps` is the proving system represented as a file which we'll now use to generate sample proofs for test input parameters:

![](https://hackmd.io/_uploads/rkCJU17Y2.png)

Next let's generate some test parameters and generate a sample proof using our new `ps` proof systems and write it as json to a file:

```bash
./gnark-mbu gen-test-params --batch-size 10 --tree-depth 30 > ../b10/params.json
cat ../b10/params.json | ./gnark-mbu prove --keys-file ../b10/ps > ../b10/proof10.json
./gnark-mbu gen-test-params --batch-size 100 --tree-depth 30 > ../b100/params.json
cat ../b100/params.json | ./gnark-mbu prove --keys-file ../b100/ps > ../b100/proof100.json
./gnark-mbu gen-test-params --batch-size 1000 --tree-depth 30 > ../b1000/params.json
cat ../b1000/params.json | ./gnark-mbu prove --keys-file ../b1000/ps > ../b1000/proof1000.json
```

And the last step is to verify the proof!

> Install [yq](https://github.com/mikefarah/yq/#install) to help with parsing json

```bash
# grab the input hash from the params.json using yq
yq e ".inputHash" ../b10/params.json
# verify proof
cat ../b10/proof10.json | ./gnark-mbu verify --input-hash <INPUT_HASH> --keys-file ../b10/ps
yq e ".inputHash" ../b100/params.json
cat ../b100/proof100.json | ./gnark-mbu verify --input-hash <INPUT_HASH> --keys-file ../b100/ps
yq e ".inputHash" ../b100/params.json
cat ../b1000/proof100.json | ./gnark-mbu verify --input-hash <INPUT_HASH> --keys-file ../b1000/ps
```

![](https://hackmd.io/_uploads/SJs55lmKn.png)

And the last step before deploying to production is to generate Solidity verifier contracts for these gnark groth16 proofs. Using `semaphore-mtb` run:

```bash
./gnark-mbu export-solidity --keys-file ../b10/ps --output ../b10/verifier.sol
./gnark-mbu export-solidity --keys-file ../b100/ps --output ../b100/verifier.sol
./gnark-mbu export-solidity --keys-file ../b1000/ps --output ../b1000/verifier.sol
```

And we are ready to go into production!

### Production

Deployed verifier contracts and their corresponding verifying keys can be found here:

- Batch size 10, tree depth 30: [Etherscan](https://etherscan.io/address/0x6e37bAB9d23bd8Bdb42b773C58ae43C6De43A590#code)
- Batch size 100, tree depth 30: [Etherscan](https://etherscan.io/address/0x03ad26786469c1F12595B0309d151FE928db6c4D#code)
- Batch size 1000, tree depth 30: [Etherscan](https://etherscan.io/address/0xf07d3efadD82A1F0b4C5Cc3476806d9a170147Ba#code)

### Fin
