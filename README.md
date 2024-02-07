# Mips Circuit Turkish Translate

**Bu, ZKM Erken Katılımcı Programının (ECP) 1. Aşaması olarak hizmet veren, Toplum Eğitimi amaçlı ZKMIPS'in demo uygulamasıdır.**

Demo uygulaması Cannon simülatörünü, Zokrates DSL'yi ve Groth16'yı kullanıyor. Minigeth'in tam olarak yürütülmesini destekler, tüm talimat dizisinin çıktısını alır, her talimat için kanıtlar üretir ve bunları doğrulama için zincir üstü bir sözleşmeye gönderir.

## Önkoşullar

- Donanım : MEM >= 8G

- Install [Rust (>=1.72.1)](https://www.rust-lang.org/tools/install)

- Install [Go (>=1.21)](https://go.dev/doc/install)

- Install Make

- Install [Zokrates](https://zokrates.github.io/gettingstarted.html). Tek satırlı kurulum önerilir veya kaynaktan kurulum yapılıyorsa ayarları buna göre yapın.
  Set `$ZOKRATES_STDLIB` and `$PATH`:

  ```sh
  export ZOKRATES_STDLIB=<path-to>/.zokrates/stdlib
  export PATH=<path-to>/.zokrates/bin:$PATH
  ```

  Bu, MIPS VM devresini derlemek için kullanılacaktır.

- Install [postgres](https://www.postgresql.org/download/)
  - Takip edebilirsin [this](https://www.youtube.com/watch?v=RdPYA-wDhTA) Docker'ı kullanarak yüklemek için video kılavuzu veya alternatif olarak izleyin [these instructions](https://www.docker.com/blog/how-to-use-the-postgres-docker-official-image)
  - **NOTE: you cannot use a default empty password**, set the password to `postgres` for simplicity for the rest of the guide
  - (Optional) Install [DBeaver](https://dbeaver.io/download/) or [pgadmin](https://www.pgadmin.org/download/): Using a Database Viewer make debugging and editing data much easier. For a non-UI version you can use [psql](https://www.timescale.com/blog/how-to-install-psql-on-mac-ubuntu-debian-windows/).

## Postgres Setup

From the DBeaver (or pgadmin) interface, right click on the postgres database and navigate to `SQL Editor > New SQL Script`

![SQLEditor](/images/SQLEditor.png)

In the Script page, paste this code:

```sql
  DROP TABLE IF EXISTS f_traces;
  CREATE TABLE f_traces
  (
      f_id           bigserial PRIMARY KEY,
      f_trace        jsonb                    NOT NULL,
      f_created_at   TIMESTAMP with time zone NOT NULL DEFAULT now()
  );

  DROP TABLE IF EXISTS t_block_witness_cloud;
  CREATE TABLE t_block_witness_cloud
  (
      f_id             bigserial PRIMARY KEY,
      f_block          BIGINT NOT NULL,
      f_version        BIGINT NOT NULL,
      f_object_key     text   NOT NULL,
      f_object_witness text   NOT NULL
  );

  DROP TABLE IF EXISTS t_prover_job_queue_cloud;
  CREATE TABLE t_prover_job_queue_cloud
  (
      f_id           bigserial PRIMARY KEY,
      f_job_status   INTEGER                  NOT NULL,
      f_job_priority INTEGER                  NOT NULL,
      f_job_type     TEXT                     NOT NULL,
      f_created_at   timestamp with time zone NOT NULL DEFAULT now(),
      f_version      BIGINT                   NOT NULL,
      f_updated_by   TEXT                     NOT NULL,
      f_updated_at   timestamp with time zone NOT NULL,
      f_first_block  BIGINT                   NOT NULL,
      f_last_block   BIGINT                   NOT NULL,
      f_object_key   TEXT                     NOT NULL,
      f_object_job   TEXT                     NOT NULL
  );

  DROP TABLE IF EXISTS t_proofs;
  CREATE TABLE t_proofs
  (
      f_id           bigserial PRIMARY KEY,
      f_block_number BIGINT                   NOT NULL,
      f_proof        jsonb                    NOT NULL,
      f_created_at   TIMESTAMP with time zone NOT NULL DEFAULT now()
  );

  DROP TABLE IF EXISTS t_witness_block_number;
  CREATE TABLE t_witness_block_number
  (
      f_id           bigserial PRIMARY KEY,
      f_block     BIGINT                   NOT NULL
  );

  DROP TABLE IF EXISTS t_proof_block_number;
  CREATE TABLE t_proof_block_number
  (
      f_id           bigserial PRIMARY KEY,
      f_block     BIGINT                   NOT NULL
  );

  DROP TABLE IF EXISTS t_verified_proof_block_number;
  CREATE TABLE t_verified_proof_block_number
  (
      f_id           bigserial PRIMARY KEY,
      f_block       BIGINT                    NOT NULL
  );
```

Click the Execute SQL Script button:

![ExecuteSQL](/images/ExecuteSQL.png)

**Note**: The id of first execution trace to be verified or proved is 1

**Note**: you can specify your own <first_execution_trace_id> by following commands:

```sql
  INSERT INTO t_witness_block_number(f_block) VALUES(${<first_execution_trace_id>});

  INSERT INTO t_proof_block_number(f_block) VALUES(${$(<first_execution_trace_id>});
```

## Program Execution Records

Clone [cannon-mips](https://github.com/zkMIPS/cannon-mips/) into another folder and follow its instructions to generate the program execution records. Notes:

- Please use your own Alchemy/Infura account
- For the POSTGRES configuration, you can use: `export POSTGRES_CONFIG="sslmode=disable user=postgres password=postgres host=172.17.0.2 port=5432 dbname=postgres"`. Note that the host points to a bridged Docker host.
- If you have issues with your db login you can try resetting the password: `docker exec -it postgres psql -U postgres -c "ALTER USER postgres WITH PASSWORD 'postgres';"`

We should see a record in the database in the `f_traces` table (refresh the table if you don't see it)

![f_traces](/images/f_traces.png)

**Note**: There should be 1 record that starts with id of 1. If the id of that record is not 1, change it to 1.

Now that we have the trace, we want to go back, Clone [mips_circuit](https://github.com/zkMIPS/mips_circuit/) and compile the MIPS VM circuit using Zokrates

```sh
cd ../..
git clone https://github.com/zkMIPS/mips_circuit
cd mips_circuit
pushd core/lib/circuit
zokrates compile -i mips_vm_poseidon.zok  # may need several minutes
wget http://ec2-46-51-227-198.ap-northeast-1.compute.amazonaws.com/proving.key -O proving.key
popd
```

## Verification though a Smart Contract Verifier

We have deployed a goerli verify contract at: [0xacd47ec395668320770e7183b9ee817f4ff8774e](https://goerli.etherscan.io/address/0xacd47ec395668320770e7183b9ee817f4ff8774e). You can use this to verify the proof.

The next steps will be focused on verifying the proof on-chain

### Witness Generator

We need to edit the environment variables, replacing the variables with your database, the RPC from goerli, and the private key for your verifier account, note that the verifier account needs some Goerli ETH to post the witness string

Edit the `setenv.bash` file:

```sh
export DATABASE_URL=postgresql://postgres:postgres@localhost:5432/postgres
export DATABASE_POOL_SIZE=10
export API_PROVER_PORT=8088
export API_PROVER_URL=http://127.0.0.1:8088
export API_PROVER_SECRET_AUTH=sample
export PROVER_PROVER_HEARTBEAT_INTERVAL=1000
export PROVER_PROVER_CYCLE_WAIT=500
export PROVER_PROVER_REQUEST_TIMEOUT=10
export PROVER_PROVER_DIE_AFTER_PROOF=false
export PROVER_CORE_GONE_TIMEOUT=60000
export PROVER_CORE_IDLE_PROVERS=1
export PROVER_WITNESS_GENERATOR_PREPARE_DATA_INTERVAL=500
export PROVER_WITNESS_GENERATOR_WITNESS_GENERATORS=2
export CIRCUIT_FILE_PATH=${PWD}/core/lib/circuit/out # generated by zokrates compile -i mips_vm_poseidon.zok
export CIRCUIT_ABI_FILE_PATH=${PWD}/core/lib/circuit/abi.json # generated by zokrates compile -i mips_vm_poseidon.zok
export RUST_LOG=warn
export VERIFIER_CHAIN_URL=PROVIDER_URL # provider url where the verifier contract deployed, Note: please use your own secret key here
export VERIFIER_CONTRACT_ADDRESS=0xacd47ec395668320770e7183b9ee817f4ff8774e # verifier contract address
export VERIFIER_ACCOUNT=PRIVATE_KEY # your goerli account private key
export VERIFIER_ABI_PATH=${PWD}/contract/verifier/g16/verifier
export CHAIN_ETH_NETWORK=goerli
export CIRCUIT_PROVING_KEY_PATH=${PWD}/core/lib/circuit/proving.key # generated by: zokrates compile -i mips_vm_poseidon.zok
```

Source the file into your local shell

```sh
source ./setenv.bash
```

Compile the witness generator

```sh
pushd core/bin/server
cargo build --release # may need several minutes
popd
```

Run the witness generator

```sh
nohup ./target/release/server > server.output 2>&1 &
```

In the `server.output` file, you should be able to see

```
Witness:
true

witness_str:A_LONG_STRING
```

## Prover

Now that the proof is on-chain, we can verify the proof using a Prover

We need to set the environment variables

```sh
source ./setenv.bash
```

Compile the Prover

```sh
pushd core/bin/prover
cargo build --release  # may need several minutes
popd
```

And run the Prover

```sh
nohup ./target/release/prover > prover.output 2>&1 &
```

In a few seconds, you should be able to see your transaction [here](https://goerli.etherscan.io/address/0xacd47ec395668320770e7183b9ee817f4ff8774e).

Congratulations! You have completed the process of posting and verifying a ZK proof with the MIPS circuit.
