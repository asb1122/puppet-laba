# Лабораторная работа: HashiCorp Vault KV v2, Hiera и Vault PKI через Puppet

> Инфраструктура: Rocky Linux / AlmaLinux / RHEL 9, HashiCorp Vault 1.18.3, Puppet Server и Puppet Agent.
>
> Цель: развернуть постоянный Vault с TLS и Raft, подключить Vault KV v2 к Hiera 5 через `hiera-vault`, обеспечить fail-safe поведение при недоступности Vault и подготовить PKI для выдачи сервисных TLS-сертификатов.

## 1. Схема инфраструктуры

| IP | FQDN | Назначение |
|---|---|---|
| `192.168.225.7` | `mgmt2.train.local.a1.by` | management |
| `192.168.225.6` | `git2.train.local.a1.by` | Git-сервер |
| `192.168.225.8` | `node2-01.train.local.a1.by` | Puppet Agent |
| `192.168.225.9` | `puppet2.train.local.a1.by` | Puppet Server |
| `192.168.225.10` | `node2-02.train.local.a1.by` | Puppet Agent |
| `192.168.225.2` | `haproxy.train.local.a1.by` | HAProxy |
| `192.168.225.3` | `vault.train.local.a1.by` | Vault Server |

Открываемые порты:

- Vault API: `8200/tcp`.
- Vault Raft cluster: `8201/tcp`.
- Puppet Server: `8140/tcp`.
- SSH: `22/tcp`.
- HTTPS сервиса: `443/tcp`.

---

## 2. Подготовка DNS или `/etc/hosts`

Если DNS отсутствует, добавьте на все узлы:

```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'
192.168.225.7  mgmt2.train.local.a1.by mgmt2
192.168.225.6  git2.train.local.a1.by git2
192.168.225.8  node2-01.train.local.a1.by node2-01
192.168.225.9  puppet2.train.local.a1.by puppet2 puppet.train.local.a1.by puppet
192.168.225.10 node2-02.train.local.a1.by node2-02
192.168.225.2  haproxy.train.local.a1.by haproxy
192.168.225.3  vault.train.local.a1.by vault
EOF
```

Проверка:

```bash
getent hosts vault.train.local.a1.by
getent hosts puppet2.train.local.a1.by
ping -c 2 vault.train.local.a1.by
```

Синхронизация времени на всех узлах:

```bash
sudo dnf install -y chrony
sudo systemctl enable --now chronyd
timedatectl
chronyc tracking
```

---

# Часть I. Установка постоянного Vault

## 3. Установка Vault 1.18.3

На `vault.train.local.a1.by`:

```bash
sudo -i
cd /tmp

curl -fL \
  --connect-timeout 15 \
  --max-time 600 \
  --retry 3 \
  -o vault_1.18.3_linux_amd64.zip \
  'https://releases.hashicorp.com/vault/1.18.3/vault_1.18.3_linux_amd64.zip'

curl -fL \
  -o vault_1.18.3_SHA256SUMS \
  'https://releases.hashicorp.com/vault/1.18.3/vault_1.18.3_SHA256SUMS'

grep 'vault_1.18.3_linux_amd64.zip' vault_1.18.3_SHA256SUMS | sha256sum -c -
unzip -t vault_1.18.3_linux_amd64.zip
sudo unzip -o vault_1.18.3_linux_amd64.zip -d /usr/local/bin/
sudo chown root:root /usr/local/bin/vault
sudo chmod 0755 /usr/local/bin/vault
sudo ln -sfn /usr/local/bin/vault /usr/bin/vault
vault version
```

Ожидается `Vault v1.18.3`.

Создайте пользователя и каталоги:

```bash
groupadd --system vault 2>/dev/null || true
id vault >/dev/null 2>&1 || useradd \
  --system \
  --gid vault \
  --home-dir /etc/vault.d \
  --shell /sbin/nologin \
  vault

install -d -o vault -g vault -m 0750 /etc/vault.d
install -d -o vault -g vault -m 0700 /opt/vault/data
install -d -o vault -g vault -m 0750 /opt/vault/tls
```

---

## 4. Создание bootstrap CA

Vault PKI ещё не запущен, поэтому HTTPS самого Vault получает отдельный bootstrap-сертификат.

```bash
sudo -i
mkdir -p /root/vault-bootstrap-ca
chmod 0700 /root/vault-bootstrap-ca
cd /root/vault-bootstrap-ca

openssl genrsa -out vault-bootstrap-ca.key 4096
chmod 0600 vault-bootstrap-ca.key

openssl req \
  -x509 \
  -new \
  -sha256 \
  -days 3650 \
  -key vault-bootstrap-ca.key \
  -out vault-bootstrap-ca.crt \
  -subj '/C=BY/O=Train Lab/OU=Infrastructure/CN=Train Vault Bootstrap CA'

chmod 0644 vault-bootstrap-ca.crt
```

Проверка:

```bash
openssl pkey -in vault-bootstrap-ca.key -check -noout
openssl x509 -in vault-bootstrap-ca.crt -noout -subject -issuer -dates
openssl x509 -in vault-bootstrap-ca.crt -noout -text | grep -A3 'Basic Constraints'
```

Приватный файл `vault-bootstrap-ca.key` нельзя передавать другим узлам.

---

## 5. Сертификат HTTPS для Vault

Создайте SAN-конфигурацию:

```bash
cd /root/vault-bootstrap-ca

cat > vault-server.cnf <<'EOF'
[req]
prompt = no
distinguished_name = distinguished_name
req_extensions = request_extensions

[distinguished_name]
C = BY
O = Train Lab
OU = Infrastructure
CN = vault.train.local.a1.by

[request_extensions]
subjectAltName = @alternative_names
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[alternative_names]
DNS.1 = vault.train.local.a1.by
DNS.2 = vault
IP.1 = 192.168.225.3
IP.2 = 127.0.0.1
EOF

openssl genrsa -out vault.key 2048
chmod 0600 vault.key

openssl req \
  -new \
  -sha256 \
  -key vault.key \
  -out vault.csr \
  -config vault-server.cnf

openssl x509 \
  -req \
  -sha256 \
  -days 825 \
  -in vault.csr \
  -CA vault-bootstrap-ca.crt \
  -CAkey vault-bootstrap-ca.key \
  -CAcreateserial \
  -out vault.crt \
  -extensions request_extensions \
  -extfile vault-server.cnf
```

Проверки:

```bash
openssl verify -CAfile vault-bootstrap-ca.crt vault.crt
openssl verify -CAfile vault-bootstrap-ca.crt -verify_hostname vault.train.local.a1.by vault.crt
openssl verify -CAfile vault-bootstrap-ca.crt -verify_ip 192.168.225.3 vault.crt
openssl x509 -in vault.crt -noout -subject -issuer -dates -ext subjectAltName
```

Установите файлы:

```bash
install -o vault -g vault -m 0644 vault.crt /opt/vault/tls/vault.crt
install -o vault -g vault -m 0640 vault.key /opt/vault/tls/vault.key
install -o vault -g vault -m 0644 vault-bootstrap-ca.crt /opt/vault/tls/ca.crt

install -o root -g root -m 0644 vault-bootstrap-ca.crt \
  /etc/pki/ca-trust/source/anchors/vault-bootstrap-ca.crt
update-ca-trust extract
```

Проверка доступа пользователя Vault:

```bash
sudo -u vault test -r /opt/vault/tls/vault.crt && echo 'certificate: OK'
sudo -u vault test -r /opt/vault/tls/vault.key && echo 'private key: OK'
openssl verify -CAfile /opt/vault/tls/ca.crt /opt/vault/tls/vault.crt
```

---

## 6. Конфигурация Vault с Raft

```bash
cat > /etc/vault.d/vault.hcl <<'EOF'
ui = true
disable_mlock = true
cluster_name = "train-vault"

api_addr     = "https://vault.train.local.a1.by:8200"
cluster_addr = "https://vault.train.local.a1.by:8201"

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault.train.local.a1.by"
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"

  tls_disable   = false
  tls_cert_file = "/opt/vault/tls/vault.crt"
  tls_key_file  = "/opt/vault/tls/vault.key"
}

log_level = "info"
default_lease_ttl = "24h"
max_lease_ttl     = "768h"
EOF

chown vault:vault /etc/vault.d/vault.hcl
chmod 0640 /etc/vault.d/vault.hcl
```

Диагностика для Vault 1.18.3:

```bash
sudo -u vault /usr/local/bin/vault operator diagnose \
  -config=/etc/vault.d/vault.hcl
```

До инициализации предупреждение `0 voters found` допустимо. Сообщение о недоступности HCP для OSS Vault также не блокирует работу.

---

## 7. Systemd unit

```bash
cat > /etc/systemd/system/vault.service <<'EOF'
[Unit]
Description=HashiCorp Vault
Documentation=https://developer.hashicorp.com/vault/docs
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault.d/vault.hcl

[Service]
Type=simple
User=vault
Group=vault
ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
LimitNOFILE=65536
PrivateTmp=yes
ProtectHome=yes
ProtectSystem=full
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
EOF

chown root:root /etc/systemd/system/vault.service
chmod 0644 /etc/systemd/system/vault.service
systemd-analyze verify /etc/systemd/system/vault.service
systemctl daemon-reload
```

Откройте firewall:

```bash
firewall-cmd --permanent --add-port=8200/tcp
firewall-cmd --permanent --add-port=8201/tcp
firewall-cmd --reload
```

Запустите:

```bash
systemctl enable --now vault
systemctl status vault --no-pager -l
journalctl -u vault -n 50 --no-pager
ss -lntp | grep -E ':8200|:8201'
```

Нельзя запускать `vault server -dev`. Dev-режим использует `inmem`, HTTP и теряет данные после остановки.

---

## 8. Инициализация и unseal

Настройте CLI:

```bash
cat > /etc/profile.d/vault.sh <<'EOF'
export VAULT_ADDR="https://vault.train.local.a1.by:8200"
export VAULT_CACERT="/opt/vault/tls/ca.crt"
EOF

chmod 0644 /etc/profile.d/vault.sh
source /etc/profile.d/vault.sh
unset VAULT_TOKEN
vault status
```

До инициализации ожидается `Initialized false`, `Sealed true`, `Storage Type raft`.

Инициализируйте **один раз**:

```bash
umask 077
vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json \
  > /root/vault-init.json

chmod 0600 /root/vault-init.json
```

Безопасная проверка:

```bash
jq '{
  unseal_key_count: (.unseal_keys_b64 | length),
  has_root_token: (.root_token | type == "string" and length > 0)
}' /root/vault-init.json
```

Unseal тремя ключами:

```bash
vault operator unseal "$(jq -r '.unseal_keys_b64[0]' /root/vault-init.json)"
vault operator unseal "$(jq -r '.unseal_keys_b64[1]' /root/vault-init.json)"
vault operator unseal "$(jq -r '.unseal_keys_b64[2]' /root/vault-init.json)"
vault status
```

Ожидается `Sealed false`, `HA Mode active`.

Вход initial root token:

```bash
export VAULT_TOKEN="$(jq -r '.root_token' /root/vault-init.json)"
vault token lookup
vault operator raft list-peers
```

Не храните root token в `vault.hcl`, Git, Hiera или `/etc/profile.d`.

---

## 9. Audit log

```bash
install -d -o vault -g vault -m 0750 /var/log/vault
vault audit list
```

Если `file/` отсутствует:

```bash
vault audit enable file file_path=/var/log/vault/audit.log
```

Проверки:

```bash
vault audit list -detailed
stat -c '%A %U:%G %n' /var/log/vault/audit.log
vault token lookup >/dev/null
tail -n 2 /var/log/vault/audit.log
```

Ожидаются права `0600 vault:vault`. Токены в audit log должны быть HMAC-хешированы.

---

# Часть II. KV v2 и политика Hiera

## 10. Включение KV v2

```bash
vault secrets list -detailed
vault secrets enable -path=secret -version=2 kv
```

Если путь уже существует, повторно не включайте.

Проверка версии:

```bash
vault secrets list -format=json | jq '."secret/" | {type, options}'
```

Настройка:

```bash
vault write secret/config \
  max_versions=10 \
  cas_required=false \
  delete_version_after=0s

vault read secret/config
```

---

## 11. Структура секретов для `hiera-vault`

Каждый Hiera-ключ хранится отдельным секретом с полем `value`.

Общий секрет:

```bash
read -rsp 'Введите vault_test_password: ' SECRET_VALUE
echo
vault kv put secret/puppet/common/vault_test_password value="$SECRET_VALUE"
unset SECRET_VALUE
```

Проверка без вывода значения:

```bash
vault kv get -format=json secret/puppet/common/vault_test_password | jq '{
  version: .data.metadata.version,
  fields: (.data.data | keys),
  value_not_empty: (.data.data.value | type == "string" and length > 0)
}'
```

Узловые overrides при необходимости:

```bash
vault kv put secret/puppet/node2-01.train.local.a1.by/vault_test_password value='NODE01_SECRET'
vault kv put secret/puppet/node2-02.train.local.a1.by/vault_test_password value='NODE02_SECRET'
```

Иерархия поиска:

```text
secret/puppet/<fqdn>/<hiera-key>
secret/puppet/common/<hiera-key>
```

---

## 12. Политика `puppet-hiera`

```bash
cat > /root/puppet-hiera-policy.hcl <<'EOF'
path "secret/data/puppet/common/*" {
  capabilities = ["read"]
}

path "secret/data/puppet/*" {
  capabilities = ["read"]
}

path "secret/metadata/puppet" {
  capabilities = ["list"]
}

path "secret/metadata/puppet/*" {
  capabilities = ["read", "list"]
}
EOF

chmod 0600 /root/puppet-hiera-policy.hcl
vault policy write puppet-hiera /root/puppet-hiera-policy.hcl
vault policy read puppet-hiera
```

Создайте read-only periodic token:

```bash
umask 077
vault token create \
  -policy=puppet-hiera \
  -period=24h \
  -orphan \
  -display-name=puppet-hiera \
  -format=json \
  > /root/puppet-hiera-token.json

jq -r '.auth.client_token' /root/puppet-hiera-token.json \
  > /root/puppet-hiera.token
chmod 0600 /root/puppet-hiera.token
```

Проверка ограничения:

```bash
ROOT_TOKEN="$VAULT_TOKEN"
export VAULT_TOKEN="$(cat /root/puppet-hiera.token)"

vault kv get -field=value secret/puppet/common/vault_test_password >/dev/null \
  && echo 'read allowed: OK'

if vault kv put secret/puppet/write-test value=test; then
  echo 'ERROR: write unexpectedly allowed'
else
  echo 'write correctly denied: OK'
fi

export VAULT_TOKEN="$ROOT_TOKEN"
unset ROOT_TOKEN
```

Ожидается HTTP `403 permission denied` для записи.

---

# Часть III. Puppet Server и `hiera-vault`

## 13. Передача CA и token

С Vault Server:

```bash
scp /root/puppet-hiera.token admin@puppet2.train.local.a1.by:/home/admin/
scp /root/vault-bootstrap-ca/vault-bootstrap-ca.crt \
  admin@puppet2.train.local.a1.by:/home/admin/
```

На Puppet Server:

```bash
sudo install -m 0644 /home/admin/vault-bootstrap-ca.crt \
  /etc/pki/ca-trust/source/anchors/vault-bootstrap-ca.crt
sudo update-ca-trust extract

sudo install -d -o puppet -g puppet -m 0700 \
  /etc/puppetlabs/puppetserver/vault
sudo install -o puppet -g puppet -m 0600 \
  /home/admin/puppet-hiera.token \
  /etc/puppetlabs/puppetserver/vault/token

sudo shred -u /home/admin/puppet-hiera.token
rm -f /home/admin/vault-bootstrap-ca.crt
```

Проверки:

```bash
curl -sS https://vault.train.local.a1.by:8200/v1/sys/health | jq '{initialized,sealed,standby}'
sudo -u puppet test -r /etc/puppetlabs/puppetserver/vault/token \
  && echo 'token readable: OK'
```

---

## 14. `Puppetfile` и r10k

В control-repo `/tmp/control-repo/Puppetfile`:

```ruby
mod 'puppet-hiera_vault', '3.0.0'
```

Git-команды выполняйте от `admin`, не через `sudo`:

```bash
cd /tmp/control-repo
git add Puppetfile
git commit -m 'Add hiera_vault backend for Vault KV v2'
git push origin production
```

`r10k` запускайте через `sudo`:

```bash
sudo r10k deploy environment production -p -v
sudo jq -r '.name, .version' \
  /etc/puppetlabs/code/environments/production/modules/hiera_vault/metadata.json
```

Ожидается `puppet-hiera_vault`, `3.0.0`.

Если `r10k` не имеет SSH-доступа, создайте отдельный deploy key `/root/.ssh/r10k_ed25519`, добавьте `.pub` в `/home/git/.ssh/authorized_keys` на `git2` и задайте `/root/.ssh/config`:

```text
Host git2.train.local.a1.by
    HostName git2.train.local.a1.by
    User git
    IdentityFile /root/.ssh/r10k_ed25519
    IdentitiesOnly yes
    StrictHostKeyChecking yes
```

Проверка:

```bash
sudo git ls-remote "$(git -C /tmp/control-repo remote get-url origin)"
```

---

## 15. Ruby gems

Для Puppet Ruby 2.7 зафиксируйте совместимую версию `vault 0.18.2`:

```bash
sudo /opt/puppetlabs/puppet/bin/gem install vault \
  --version 0.18.2 --no-document
sudo /opt/puppetlabs/puppet/bin/gem install debouncer --no-document

sudo puppetserver gem install vault \
  --version 0.18.2 --no-document
sudo puppetserver gem install debouncer --no-document
```

Проверки:

```bash
sudo /opt/puppetlabs/puppet/bin/ruby -e \
  'require "vault"; require "debouncer"; puts "Puppet CLI gems: OK"'

sudo puppetserver gem list | grep -E '^(vault|debouncer) '
```

Закрепите CA bundle для Puppet Server:

```bash
sudo systemctl edit puppetserver
```

Добавьте:

```ini
[Service]
Environment="SSL_CERT_FILE=/etc/pki/tls/certs/ca-bundle.crt"
```

Примените:

```bash
sudo systemctl daemon-reload
sudo systemctl restart puppetserver
sudo systemctl status puppetserver --no-pager -l
sudo systemctl show puppetserver -p Environment | grep SSL_CERT_FILE
```

---

## 16. `hiera.yaml`

`/tmp/control-repo/hiera.yaml`:

```yaml
---
version: 5

defaults:
  datadir: data
  data_hash: yaml_data

hierarchy:
  - name: "Vault KV v2"
    lookup_key: hiera_vault
    options:
      address: "https://vault.train.local.a1.by:8200"
      token: "/etc/puppetlabs/puppetserver/vault/token"
      ssl_verify: true
      strict_mode: true
      cache_for: 30
      default_field: "value"
      v1_lookup: false
      v2_guess_mount: false

      confine_to_keys:
        - "^vault_.*"
        - "^.*_password$"
        - "^profile::.*::password$"
        - "^profile::.*::secret$"

      mounts:
        secret/data/puppet:
          - "%{facts.networking.fqdn}"
          - "common"

  - name: "Per-node data"
    path: "nodes/%{facts.networking.fqdn}.yaml"

  - name: "Per-role data"
    path: "roles/%{trusted.extensions.pp_role}.yaml"

  - name: "Common"
    path: "common.yaml"
```

Никогда не помещайте `hvs...` token непосредственно в `hiera.yaml`.

Проверка и deployment:

```bash
cd /tmp/control-repo
/opt/puppetlabs/puppet/bin/ruby -e \
  'require "yaml"; YAML.load_file("hiera.yaml"); puts "hiera.yaml: OK"'
grep -nP '\t' hiera.yaml
git grep -n 'hvs\.'
git add hiera.yaml
git commit -m 'Configure secure Vault KV v2 backend for Hiera'
git push origin production
sudo r10k deploy environment production -p -v
```

---

## 17. Классификация узлов через Hiera

`manifests/site.pp`:

```puppet
node default {
  include profile::base

  $hiera_classes = lookup(
    'classes',
    Array[String],
    'unique',
    [],
  )

  include $hiera_classes
}
```

Не создавайте отдельные `node '...'` блоки, иначе `node default` для них не выполнится.

`data/nodes/node2-01.train.local.a1.by.yaml`:

```yaml
---
classes:
  - role::webserver
  - profile::vault_test

profile::nginx::worker_processes: 4
profile::nginx::max_connections: 2048
profile::nginx::content: "Hello from node2-01\n"
```

`data/nodes/node2-02.train.local.a1.by.yaml`:

```yaml
---
classes:
  - role::webserver
  - profile::vault_test
```

---

## 18. Безопасный Puppet-профиль

`site-modules/profile/manifests/vault_test.pp`:

```puppet
class profile::vault_test {
  $secret = lookup(
    'vault_test_password',
    Optional[String],
    'first',
    undef,
  )

  if $secret == undef or $secret.empty {
    fail('Required Vault secret vault_test_password is missing or empty')
  }

  $sensitive_secret = Sensitive($secret)

  file { '/etc/vault-test.conf':
    ensure    => file,
    owner     => 'root',
    group     => 'root',
    mode      => '0600',
    show_diff => false,
    content   => epp('profile/vault_test.conf.epp', {
      password => $sensitive_secret.unwrap,
    }),
  }
}
```

`site-modules/profile/templates/vault_test.conf.epp`:

```epp
<%- | String[1] $password | -%>
password=<%= $password %>
```

Нельзя делать:

```puppet
lookup('vault_test_password', String, 'first', '')
notify { "Secret: ${secret}": }
```

Проверка:

```bash
puppet parser validate site-modules/profile/manifests/vault_test.pp
puppet epp validate site-modules/profile/templates/vault_test.conf.epp
```

---

# Часть IV. Puppet Agents

## 19. Установка агента на node2-01 и node2-02

На каждом узле:

```bash
sudo dnf install -y https://yum.puppet.com/puppet8-release-el-9.noarch.rpm
sudo dnf install -y puppet-agent
sudo /opt/puppetlabs/bin/puppet --version
```

Для `node2-01`:

```bash
sudo /opt/puppetlabs/bin/puppet config set server puppet2.train.local.a1.by --section main
sudo /opt/puppetlabs/bin/puppet config set certname node2-01.train.local.a1.by --section main
sudo /opt/puppetlabs/bin/puppet config set environment production --section agent
sudo /opt/puppetlabs/bin/puppet agent --test --waitforcert 10
```

Для `node2-02` замените certname на `node2-02.train.local.a1.by`.

На Puppet Server:

```bash
sudo puppetserver ca list
sudo puppetserver ca sign --certname node2-01.train.local.a1.by
sudo puppetserver ca sign --certname node2-02.train.local.a1.by
```

Повторно на агентах:

```bash
sudo /opt/puppetlabs/bin/puppet agent -t
sudo systemctl enable --now puppet
```

---

## 20. Проверки на узлах

На каждом узле:

```bash
sudo /opt/puppetlabs/bin/puppet agent -t
sudo stat -c '%A %U:%G %n' /etc/vault-test.conf
```

Ожидается `-rw------- root:root`.

Проверка непустого пароля без раскрытия:

```bash
sudo awk -F= '
  $1 == "password" && length($2) > 0 { found=1 }
  END {
    if (found) { print "Vault secret: OK"; exit 0 }
    print "Vault secret: EMPTY OR MISSING"; exit 1
  }
' /etc/vault-test.conf
```

Проверка Nginx:

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
curl -s http://127.0.0.1/
```

Для `node2-01` ожидается `Hello from node2-01`. Для `node2-02` используется значение из `common.yaml`, если узлового override нет.

Проверка отсутствия утечки секрета:

```bash
sudo journalctl -u puppet --since '30 minutes ago' --no-pager | \
  grep -Ei 'Vault Debug|password=' || true
```

---

## 21. Негативный fail-safe тест

На каждом агенте сохраните checksum:

```bash
sudo sha256sum /etc/vault-test.conf
```

На Vault Server:

```bash
sudo systemctl stop vault
```

На агентах:

```bash
sudo /opt/puppetlabs/bin/puppet agent -t
```

Ожидается ошибка компиляции или получения каталога. Проверки:

```bash
sudo sha256sum /etc/vault-test.conf
sudo grep -q '^password=$' /etc/vault-test.conf \
  && echo 'ERROR: empty password written' \
  || echo 'Fail-safe check: OK'
```

Checksum должен остаться прежним.

Верните Vault:

```bash
sudo systemctl start vault
source /etc/profile.d/vault.sh
vault operator unseal "$(jq -r '.unseal_keys_b64[0]' /root/vault-init.json)"
vault operator unseal "$(jq -r '.unseal_keys_b64[1]' /root/vault-init.json)"
vault operator unseal "$(jq -r '.unseal_keys_b64[2]' /root/vault-init.json)"
vault status
```

---

# Часть V. Vault PKI

## 22. Включение PKI

На Vault под root token:

```bash
vault secrets enable -path=pki pki
vault secrets tune -max-lease-ttl=87600h pki

vault write pki/root/generate/internal \
  common_name='Train Lab Root CA' \
  issuer_name='train-root-2026' \
  ttl='87600h' \
  key_type='rsa' \
  key_bits=4096

vault write pki/config/urls \
  issuing_certificates='https://vault.train.local.a1.by:8200/v1/pki/ca' \
  crl_distribution_points='https://vault.train.local.a1.by:8200/v1/pki/crl'
```

Для production используйте offline root CA и intermediate CA внутри Vault.

Создайте роль:

```bash
vault write pki/roles/train-services \
  allowed_domains='train.local.a1.by' \
  allow_subdomains=true \
  allow_bare_domains=false \
  allow_localhost=false \
  allow_ip_sans=true \
  enforce_hostnames=true \
  key_type='rsa' \
  key_bits=2048 \
  server_flag=true \
  client_flag=false \
  ttl='72h' \
  max_ttl='168h'
```

Проверка выдачи:

```bash
vault write -format=json pki/issue/train-services \
  common_name='haproxy.train.local.a1.by' \
  alt_names='haproxy' \
  ip_sans='192.168.225.2' \
  ttl='72h' \
  > /root/haproxy-certificate.json

jq -r '.data.certificate' /root/haproxy-certificate.json > /root/haproxy.crt
jq -r '.data.private_key' /root/haproxy-certificate.json > /root/haproxy.key
jq -r '.data.issuing_ca' /root/haproxy-certificate.json > /root/train-pki-ca.crt

openssl x509 -in /root/haproxy.crt -noout -subject -issuer -dates -ext subjectAltName
openssl verify -CAfile /root/train-pki-ca.crt /root/haproxy.crt
```

---

## 23. PKI policy и AppRole

```bash
cat > /root/puppet-pki-policy.hcl <<'EOF'
path "pki/issue/train-services" {
  capabilities = ["create", "update"]
}

path "pki/cert/ca" {
  capabilities = ["read"]
}

path "pki/ca/pem" {
  capabilities = ["read"]
}
EOF

vault policy write puppet-pki /root/puppet-pki-policy.hcl
vault auth enable approle

vault write auth/approle/role/puppet-pki \
  token_policies='puppet-pki' \
  token_ttl='30m' \
  token_max_ttl='2h' \
  secret_id_ttl='720h' \
  secret_id_num_uses=0

vault read -field=role_id auth/approle/role/puppet-pki/role-id \
  > /root/puppet-pki-role-id
vault write -field=secret_id -f auth/approle/role/puppet-pki/secret-id \
  > /root/haproxy-secret-id
chmod 0600 /root/puppet-pki-role-id /root/haproxy-secret-id
```

Для каждого узла используйте отдельный Secret ID.

---

## 24. Критерии сдачи

1. `vault status`: `Initialized true`, `Sealed false`, `Storage Type raft`.
2. HTTPS Vault проверяется без `curl -k`.
3. `vault audit list` показывает `file/`.
4. `secret/` является KV v2.
5. Puppet token читает секрет и получает `403` при записи.
6. `hiera.yaml` содержит путь к token-файлу, `ssl_verify: true`, `strict_mode: true`.
7. В Git отсутствуют строки `hvs...`.
8. `node2-01` получает узловые Nginx-параметры `4` и `2048`.
9. Оба агента получают `profile::vault_test` через `classes` в Hiera.
10. `/etc/vault-test.conf` имеет права `0600` и непустое значение.
11. Секрет не выводится через `notify`, Puppet logs или diff.
12. При выключенном Vault каталог не компилируется, старый конфиг остаётся неизменным.
13. Vault PKI выдаёт сертификат только для `*.train.local.a1.by`.
14. `openssl verify` возвращает `OK`.

---

## 25. Частые ошибки

| Ошибка | Причина | Исправление |
|---|---|---|
| `bind: address already in use` | Работает `vault server -dev` | Остановить dev-процесс, запускать systemd service |
| `flag provided but not defined: -verify-only` | В Vault 1.18.3 нет этого флага | Использовать `vault operator diagnose -config=...` |
| `0 voters found` до init | Raft ещё не инициализирован | Допустимо до `vault operator init` |
| `path already in use` для audit | Audit уже включён | Проверить `vault audit list -detailed` |
| `certificate verify failed` | Puppet Ruby не видит CA | Установить CA и задать `SSL_CERT_FILE` |
| `No facts available for target node` | Локальный lookup не имеет facts | Передать `--facts`, либо тестировать через agent |
| Hiera ищет `nodes/.yaml` | `trusted.certname` пуст в локальном lookup | Использовать `%{facts.networking.fqdn}` |
| `vault` gem требует Ruby >= 3.1 | Puppet embedded Ruby 2.7 | Зафиксировать `vault` gem `0.18.2` |
| `r10k` SSH denied | У root нет deploy key | Создать `/root/.ssh/r10k_ed25519` |
| Git `insufficient permission` | В repo есть root-owned objects | `chown -R admin:admin /tmp/control-repo` |
| Секрет виден в Notice | Используется debug `notify` | Удалить notify, ротировать секрет |

---

## 26. Безопасность

- Не храните root token, unseal keys и Vault token в Git.
- Не используйте `ssl_verify: false` и `curl -k` в итоговой конфигурации.
- Не храните все unseal keys рядом с Vault в production.
- Используйте `Sensitive`, `show_diff => false`, `String[1]` и явный `fail()`.
- После раскрытия токена или пароля немедленно отзывайте или ротируйте его.
- Периодический token с `period=24h` нужно продлевать. Для production настройте Vault Agent Auto-Auth с AppRole.
- Настоящий auto-unseal реализуют через KMS, HSM или Transit Seal, а не скриптом с ключами на том же сервере.
