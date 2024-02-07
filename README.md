# Mips Circuit Turkish Translate

**Bu, ZKM Erken Katılımcı Programının (ECP) 1. Aşaması olarak hizmet veren, Toplum Eğitimi amaçlı ZKMIPS'in demo uygulamasıdır.**

Demo uygulaması Cannon simülatörünü, Zokrates DSL'yi ve Groth16'yı kullanıyor. Minigeth'in tam olarak yürütülmesini destekler, tüm talimat dizisinin çıktısını alır, her talimat için kanıtlar üretir ve bunları doğrulama için zincir üstü bir sözleşmeye gönderir.

## Önkoşullar

- Donanım : MEM >= 8G

- İndir [Rust (>=1.72.1)](https://www.rust-lang.org/tools/install)

- İndir [Go (>=1.21)](https://go.dev/doc/install)

- İndir Make

- İndir [Zokrates](https://zokrates.github.io/gettingstarted.html). Tek satırlı kurulum önerilir veya kaynaktan kurulum yapılıyorsa ayarları buna göre yapın.
  Set `$ZOKRATES_STDLIB` and `$PATH`:

  ```sh
  export ZOKRATES_STDLIB=<path-to>/.zokrates/stdlib
  export PATH=<path-to>/.zokrates/bin:$PATH
  ```

  Bu, MIPS VM devresini derlemek için kullanılacaktır.

- İndir [postgres](https://www.postgresql.org/download/)
  - Takip edebilirsin [this](https://www.youtube.com/watch?v=RdPYA-wDhTA) Docker'ı kullanarak yüklemek için video kılavuzu veya alternatif olarak izleyin [these instructions](https://www.docker.com/blog/how-to-use-the-postgres-docker-official-image)
  - **NOT: Varsayılan boş şifreyi kullanamazsınız**, Kılavuzun geri kalanında basitlik sağlamak amacıyla şifreyi 'postgres' olarak ayarlayın
  - (İsteğe bağlı) İndir [DBeaver](https://dbeaver.io/download/) Ya da [pgadmin](https://www.pgadmin.org/download/): Veritabanı Görüntüleyiciyi kullanmak, verilerde hata ayıklamayı ve düzenlemeyi çok daha kolay hale getirir. Kullanıcı arayüzü olmayan bir sürüm için kullanabilirsiniz [psql](https://www.timescale.com/blog/how-to-install-psql-on-mac-ubuntu-debian-windows/).

## Postgres Kurmak

DBeaver (veya pgadmin) arayüzünden postgres veritabanına sağ tıklayın ve 'SQL Editör > Yeni SQL Komut Dosyası'na gidin

![SQLEditor](/images/SQLEditor.png)

Komut Dosyası sayfasına şu kodu yapıştırın:

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

SQL Komut Dosyasını Yürüt düğmesini tıklayın:

![ExecuteSQL](/images/ExecuteSQL.png)

**Not**: Doğrulanacak veya kanıtlanacak ilk yürütme izinin kimliği 1'dir

**Not**: Aşağıdaki komutları kullanarak kendi <first_execution_trace_id> kimliğinizi belirleyebilirsiniz:

```sql
  INSERT INTO t_witness_block_number(f_block) VALUES(${<first_execution_trace_id>});

  INSERT INTO t_proof_block_number(f_block) VALUES(${$(<first_execution_trace_id>});
```

## Program Yürütme Kayıtları

Klon [cannon-mips](https://github.com/zkMIPS/cannon-mips/) başka bir klasöre kopyalayın ve program yürütme kayıtlarını oluşturmak için talimatlarını izleyin. Notlar:

- Lütfen kendi Alchemy/Infura hesabınızı kullanın
- POSTGRES yapılandırması için şunu kullanabilirsiniz: `export POSTGRES_CONFIG="sslmode=disable user=postgres şifresi=postgres host=172.17.0.2 port=5432 dbname=postgres"`. Ana bilgisayarın köprülü bir Docker ana bilgisayarına işaret ettiğini unutmayın.
- Veritabanı giriş bilgilerinizle ilgili sorun yaşıyorsanız şifreyi sıfırlamayı deneyebilirsiniz: `docker exec -it postgres psql -U postgres -c "ALTER USER postgres with PASSWORD 'postgres';"'

Veritabanındaki `f_traces` tablosunda bir kayıt görmeliyiz (görmüyorsanız tabloyu yenileyin)

![f_traces](/images/f_traces.png)

**Not**: ID'si 1 ile başlayan 1 kayıt olmalıdır. Eğer o kaydın ID'si 1 değilse 1 olarak değiştirin.

Artık izini aldığımıza göre geri dönüp [mips_circuit](https://github.com/zkMIPS/mips_circuit/) klonlamak ve Zokrates kullanarak MIPS VM devresini derlemek istiyoruz.

```sh
cd ../..
git clone https://github.com/zkMIPS/mips_circuit
cd mips_circuit
pushd core/lib/circuit
zokrates compile -i mips_vm_poseidon.zok  # may need several minutes
wget http://ec2-46-51-227-198.ap-northeast-1.compute.amazonaws.com/proving.key -O proving.key
popd
```

## Akıllı Sözleşme Doğrulayıcı Aracılığıyla Doğrulama

Şu adreste bir goerli doğrulama sözleşmesi dağıttık: [0xacd47ec395668320770e7183b9ee817f4ff8774e](https://goerli.etherscan.io/address/0xacd47ec395668320770e7183b9ee817f4ff8774e). Kanıtı doğrulamak için bunu kullanabilirsiniz.

Sonraki adımlar zincirdeki kanıtın doğrulanmasına odaklanacak

### Tanık Oluşturucu

Değişkenleri veritabanınızla, goerli'den RPC'yle ve doğrulayıcı hesabınızın özel anahtarıyla değiştirerek ortam değişkenlerini düzenlememiz gerekiyor; doğrulayıcı hesabının tanık dizesini göndermek için bir miktar Goerli ETH'ye ihtiyacı olduğunu unutmayın.

`setenv.bash` dosyasını düzenleyin:

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

Dosyayı yerel kabuğunuza kaynaklayın (Source the file into your local shell)

```sh
source ./setenv.bash
```

Tanık oluşturucuyu derleyin

```sh
pushd core/bin/server
cargo build --release # may need several minutes
popd
```

Tanık oluşturucuyu çalıştırın

```sh
nohup ./target/release/server > server.output 2>&1 &
```

`server.output` dosyasında görebilmeniz gerekir

```
Witness:
true

witness_str:A_LONG_STRING
```

## Kanıtlayıcı

Artık kanıt zincirde olduğuna göre kanıtı bir Prover kullanarak doğrulayabiliriz

Ortam değişkenlerini ayarlamamız gerekiyor

```sh
source ./setenv.bash
```

Prover'ı(Kanıtlayıcıyı) derleyin

```sh
pushd core/bin/prover
cargo build --release  # may need several minutes
popd
```

Ve Prover'ı çalıştırın

```sh
nohup ./target/release/prover > prover.output 2>&1 &
```

Birkaç saniye içinde işleminizi görebilmelisiniz [here](https://goerli.etherscan.io/address/0xacd47ec395668320770e7183b9ee817f4ff8774e).

Tebrikler! MIPS devresiyle ZK kanıtını gönderme ve doğrulama işlemini tamamladınız.


