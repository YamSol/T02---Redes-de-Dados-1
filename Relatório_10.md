<img width="180" alt="image" src="https://github.com/user-attachments/assets/cf68b22b-ecb8-4e9e-8f09-271ef1bc1246" />

<div align="right">
T02 - Redes de Dados
</div>

# Relatório 10: TLS em HTTP com Wireshark (VMs no VirtualBox)
 
Este relatório tem como objetivo **montar um laboratório em NAT Network (rede NAT do VirtualBox)** para demonstrar o uso de **TLS** no protocolo **HTTP**, comparando o tráfego em texto claro (**HTTP sem TLS**) e criptografado (**HTTPS com TLS**) e **analisando as diferenças no Wireshark**.

* **Material de apoio:** `https://github.com/vin1sss/T02---Redes-de-Dados-1/`

---

## I. Introdução

Nesta atividade, você configurará duas VMs (**user 1 - Debian** e **user 2 - Debian**) em uma **NAT Network** do VirtualBox para observar, via Wireshark no **user 2 - Debian**, como o TLS protege o conteúdo das mensagens HTTP. Em texto claro, o payload é legível; com TLS, apenas metadados e o **handshake** ficam visíveis.

O TLS cria um canal cifrado sobre TCP por meio do *handshake*, com negociação de chaves de sessão temporárias para proteger o tráfego de aplicação. Na prática, isso reduz a exposição a interceptação e leitura de conteúdo por terceiros no caminho.

**Pilares abordados:** foco em **Confidencialidade** (com menção a Autenticidade/Integridade via certificados e MAC do TLS).

---

## II. Conceitos e Fundamentos

* **HTTP x HTTPS:** HTTPS = HTTP sobre TLS (camada de segurança entre transporte e aplicação).
* **TLS (Handshake):** ClientHello → ServerHello (+ certificado) → *key exchange* → chaves de sessão → comunicação criptografada.
* **Certificados:** neste lab usaremos certificado **autoassinado** no Apache.
* **Wireshark:** filtros úteis `http`, `tls`, `tcp.port == 443`, `tls.handshake`.

---

## III. Ambiente e Topologia

**VirtualBox — NAT Network:** `NatNetwork`

> **Atenção:** neste relatório usamos **NAT Network** (rede compartilhada entre VMs). Isso é diferente de **NAT** simples por VM (*Attached to: NAT*), que normalmente não permite comunicação direta entre as VMs.

### 1) Resumo do laboratório

| Elemento | Definição |
| --- | --- |
| Rede do VirtualBox | **NAT Network** `NatNetwork` |
| user 1 - Debian | Servidor Apache, usando as portas **80** e **443** |
| user 2 - Debian | Cliente de testes com `curl` e **Wireshark** |
| Adaptadores das VMs | Uma placa por VM, ligada à **NAT Network** |
| Recursos das VMs | **6 GB de RAM** e **3 núcleos de CPU** |

### 2) Dados de referência usados no relatório

| Item | Uso |
| --- | --- |
| Senha das máquinas virtuais | **`pytest`** |
| Senha solicitada pelo `sudo` | **`pytest`** |
| **`<IP_USER1>`** | IP atual do **user 1 - Debian** |
| **`<IP_USER2>`** | IP atual do **user 2 - Debian** |

### 3) Pacotes usados no laboratório

* **user 1 - Debian:** `apache2`, `openssl`
* **user 2 - Debian:** `curl`, `wireshark`

---

## IV. Instalação e Preparação

> Substitua as ocorrências de **`<IP_USER1>`** e **`<IP_USER2>`** pelos endereços obtidos nas suas VMs.

### 1) Preparar a rede no VirtualBox

1. **Abrir o gerenciador de redes NAT:**

   * No VirtualBox, clique em **“Arquivos”**
   * Clique em **“Ferramentas”**
   * Clique em **“Gerenciador de Rede”**
   * Clique em **“Redes NAT”**

2. **Se já existir `NatNetwork`, apenas confira se está configurada:**

   * **Name:** `NatNetwork`
   * **IPv4 Prefix:** verifique o prefixo (ex.: `10.0.2.0/24`)
   * **Habilitar DHCP**: Marcado

3. **Se NÃO existir `NatNetwork`: criar uma:**

   * Clique em **“Criar”**
   * **Name:** `NatNetwork`
   * **IPv4 Prefix:** ex.: `10.0.2.0/24` (padrão)
   * Marque **Habilitar DHCP**
   * Confirme com **Aplicar**

<img width="1190" height="862" alt="image" src="https://github.com/user-attachments/assets/a890ec72-c5da-49a0-81c3-e21ee7deb404" />

4. **Vincular cada VM à `NatNetwork`:**

   * Para **user 1 - Debian** e **user 2 - Debian** → **Configurações** (*Settings*) → **Rede** (*Network*)
   * **Adaptador 1** (*Adapter 1*) → **Conectado a:** *NAT Network* → **Nome:** `NatNetwork`
   * OK
  
<img width="808" height="345" alt="image" src="https://github.com/user-attachments/assets/3a4a1d8a-63b4-4c48-b06a-d65de72964a2" />

5. **Ajustar recursos de hardware das VMs (user 1 - Debian e user 2 - Debian):**

   * VirtualBox → **Configurações** → **Sistema** → **Placa-mãe** → **Memória Base: `6144 MB` (6 GB)**
   * VirtualBox → **Configurações** → **Sistema** → **Processador** → **Processadores: `3`**
   * Após ajustar os recursos, **iniciar** as VMs

<img width="466" height="122" alt="image" src="https://github.com/user-attachments/assets/4e1de125-9012-4b75-b395-e3aa73a179a8" />

> **Ao final desta etapa:** as duas VMs devem estar ligadas, vinculadas à `NatNetwork` e prontas para receber IP via DHCP.

### 2) Conferir IPs, padronizar DHCP e validar a comunicação

#### A) (Opcional) Forçar DHCP no Debian

Use este ajuste em **cada VM Debian** quando ela estiver sem IPv4 válido, fora da faixa da `NatNetwork` ou com configuração estática antiga.

1. Identifique a interface de rede e abra o arquivo de interfaces:

   ```bash
   ip link show
   sudo nano /etc/network/interfaces
   ```

   Use o modelo abaixo (substitua `<interface>` pelo nome real, por exemplo `enp0s3`):

   ```text
   auto lo
   iface lo inet loopback

   auto <interface>
   iface <interface> inet dhcp
   ```

   Em seguida, recarregue a interface:

   ```bash
   sudo ifdown <interface> || true
   sudo ifup <interface>
   ```

#### B) Conferir IPs e validar comunicação

1. Em **cada VM Debian**, execute:

   ```bash
   ip a
   ```

2. Identifique o endereço IPv4 usado na interface conectada à **`NatNetwork`** e verifique se os IPs de **user 1 - Debian** e **user 2 - Debian** pertencem à **mesma faixa de rede** configurada no VirtualBox.

   * Exemplo: se a `NatNetwork` estiver em `10.0.2.0/24`, os dois IPs devem seguir o padrão `10.0.2.x/24`.
   * Anote o IP do servidor como **`<IP_USER1>`** e o IP do cliente como **`<IP_USER2>`**.

  <img width="1481" height="445" alt="image" src="https://github.com/user-attachments/assets/70c2e67a-184a-487b-a232-36a1759252e1" />

   > **Anote antes de seguir:** todos os comandos posteriores usam esses dois valores. Se necessário, mantenha os IPs anotados em separado durante a execução do laboratório.

3. **Se os IPs estiverem compatíveis**, avance para o teste de comunicação.

4. **Se os IPs não estiverem na mesma rede**, ou se alguma VM não tiver recebido um IPv4 válido da **`NatNetwork`**, execute em **cada VM Debian**:

   ```bash
   IFACE=$(ip route | awk '/default/ {print $5; exit}')
   sudo dhclient -r "$IFACE" || true
   sudo ip addr flush dev "$IFACE"
   sudo dhclient "$IFACE"
   ip -4 a show "$IFACE"
   ```

   > **Nota:** a interface padrão em Debian no VirtualBox é normalmente **`enp0s3`** (Ethernet, barramento 0, slot 3). Se precisar verificar ou consultar manualmente, execute `ip route` ou `ip a` para confirmar qual interface está ativa.

   Após isso, execute novamente `ip a` nas duas VMs e confirme se os endereços agora estão na mesma faixa de rede.

5. **Testar comunicação entre as VMs com `ping`:**

   No **user 2 - Debian**, execute:

   ```bash
   ping -c2 <IP_USER1>
   ```

   No **user 1 - Debian**, execute:

   ```bash
   ping -c2 <IP_USER2>
   ```

   <img width="1481" height="370" alt="image" src="https://github.com/user-attachments/assets/e537c6a0-607b-4553-abec-e13e0b7375ad" />

> **Ao final desta etapa:** as duas VMs devem estar na mesma rede e responder ao `ping` entre si.

### 3) Preparar o servidor — user 1 - Debian

#### A) Instalar dependências

```bash
sudo apt update && sudo apt install -y apache2 openssl
```

<img width="695" height="323" alt="image" src="https://github.com/user-attachments/assets/1592bc42-1a35-46b9-8e64-ac8779790f87" />

#### B) Criar página de teste com conteúdo identificável

```bash
sudo mkdir -p /var/www/html
echo "<h1>HELLO_TLS_HTTP</h1>" | sudo tee /var/www/html/index.html
```

<img width="628" height="96" alt="image" src="https://github.com/user-attachments/assets/1d5ca830-2b83-4197-9bb3-a7fd1f6852c1" />

#### C) Ativar o Apache e verificar a porta HTTP

```bash
sudo systemctl enable --now apache2
ss -tulpn | grep :80
```

> **Resultado esperado:** deve aparecer uma linha indicando que o Apache está em estado **LISTEN** na porta **80**.

<img width="723" height="153" alt="image" src="https://github.com/user-attachments/assets/2c5dc094-4ede-4deb-90fd-0c3007cca55d" />

> **Ao final desta etapa:** o servidor deve estar respondendo em **HTTP** na porta **80**.

### 4) Preparar o cliente — user 2 - Debian

#### A) Instalar `curl` e Wireshark

> Observação: escolha "Yes" na configuração do Wireshark

```bash
sudo apt update
sudo apt install -y curl wireshark
```

<img width="596" height="177" alt="image" src="https://github.com/user-attachments/assets/92dbd981-4482-47f4-a52c-a111d0a10500" />

#### B) Abrir o Wireshark e identificar a interface de captura

```bash
sudo wireshark
```

Esse comando **apenas abre o programa**. Na tela inicial, identifique a **interface da NAT Network** (a que mostra o IP do **user 2 - Debian**, costuma ser **enp0s3**) e deixe o Wireshark pronto para iniciar a captura no próximo passo.

<img width="921" height="501" alt="image" src="https://github.com/user-attachments/assets/9e375ed7-f018-4bb6-9ffd-9ccbb67e950a" />

Durante a análise dos pacotes, use como referência o filtro:

```text
ip.addr == <IP_USER1>
```

> **Ao final desta etapa:** o cliente deve estar com `curl` instalado, o Wireshark aberto e a interface correta já identificada.

> **Observação:** os filtros indicados neste relatório são **filtros de exibição** no Wireshark, usados para localizar melhor os pacotes já capturados.

---

## V. Procedimentos (Passo a Passo)

### Checklist antes dos testes

Antes de iniciar os cenários, confirme:

* o IP do servidor está anotado como **`<IP_USER1>`**;
* o Apache responde em **HTTP** na porta **80**;
* o Wireshark está aberto no **user 2 - Debian**;
* a interface da **NAT Network** já foi identificada (costuma ser **enp0s3**).

### Cenário 1 — HTTP **sem TLS** (porta 80)

#### A) Preparar a captura

1. **No user 2 - Debian:** com o Wireshark já aberto, **inicie a captura** na interface da **NAT Network** identificada anteriormente.

<img width="366" height="292" alt="image" src="https://github.com/user-attachments/assets/480f3519-fe3c-4dfd-8cec-329fc63a8c0d" />

<img width="777" height="213" alt="image" src="https://github.com/user-attachments/assets/e8be4387-047f-4705-a085-4eafb714164f" />

> **Observação:** o quadrado vermelho indica que a captura está ativa.

#### B) Executar o teste HTTP

1. **No user 2 - Debian:** execute:

```bash
curl -v http://<IP_USER1>
```

> **Resultado esperado no terminal:** o `curl` deve completar a requisição e exibir a resposta do Apache, incluindo o conteúdo da página com `HELLO_TLS_HTTP`.

<img width="467" height="417" alt="image" src="https://github.com/user-attachments/assets/3c0016d3-3879-4451-94a3-f9b2ad3f8a92" />

#### C) Observar no Wireshark

1. **Pause a captura** no Wireshark.

<img width="138" height="92" alt="image" src="https://github.com/user-attachments/assets/cfedfeba-6545-4297-aa1b-530b14688ed8" />

2. Aplique o filtro:

   ```text
   http && ip.addr == <IP_USER1>
   ```

3. **O que observar:** requisição `GET / HTTP/1.1` e resposta `200 OK` com **payload legível** (HTML contendo `HELLO_TLS_HTTP`).

<img width="897" height="532" alt="image" src="https://github.com/user-attachments/assets/43497e00-b9d0-436b-8072-859dcaa368e0" />
<img width="512" height="332" alt="image" src="https://github.com/user-attachments/assets/2310e98c-7f1a-40fc-887a-592cdceb22a9" />

> **Ao final deste cenário:** o tráfego HTTP deve aparecer em texto claro no Wireshark.

### Cenário 2 — HTTP **com TLS** (HTTPS na porta 443)

#### A) Preparar HTTPS no servidor — user 1 - Debian

1. **Ativar SSL no Apache:**

```bash
sudo a2enmod ssl
sudo a2ensite default-ssl
sudo systemctl reload apache2
```

<img width="412" height="277" alt="image" src="https://github.com/user-attachments/assets/46ea3fc5-5434-44a8-8773-2cb63ed55ee3" />

> **Observação:** caso apareça um erro relacionado ao Apache, ignore - ele apenas indica que o Apache não estava rodando anteriormente.

2. **Gerar certificado autoassinado (1 ano):**

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/apache.key \
  -out /etc/ssl/certs/apache.crt \
  -subj "/C=BR/ST=MG/L=Inatel/O=Lab/OU=NetSec/CN=<IP_USER1>"
```

<img width="932" height="587" alt="image" src="https://github.com/user-attachments/assets/fa045612-e0ef-4120-9123-b8f47b2144ff" />

3. **Apontar o vhost SSL para o certificado/chave e reiniciar Apache:**

```bash
sudo sed -i 's#SSLCertificateFile .*#SSLCertificateFile /etc/ssl/certs/apache.crt#' /etc/apache2/sites-available/default-ssl.conf
sudo sed -i 's#SSLCertificateKeyFile .*#SSLCertificateKeyFile /etc/ssl/private/apache.key#' /etc/apache2/sites-available/default-ssl.conf
sudo systemctl restart apache2
ss -tulpn | grep :443
```

> **Resultado esperado:** deve aparecer uma linha indicando que o Apache está em estado **LISTEN** na porta **443**.

<img width="930" height="171" alt="image" src="https://github.com/user-attachments/assets/baaaf721-a9ce-4249-8f87-748c2494a942" />

> **Ao final desta etapa:** o Apache deve estar escutando em **HTTPS** na porta **443**.

#### B) Reiniciar a captura e executar o teste HTTPS — user 2 - Debian

1. No Wireshark, **reinicie a captura** na interface da **NAT Network** para registrar somente o tráfego do novo teste.

<img width="786" height="497" alt="image" src="https://github.com/user-attachments/assets/069ce333-c2a3-46b4-8ada-6343845d21f3" />

<img width="777" height="213" alt="image" src="https://github.com/user-attachments/assets/e8be4387-047f-4705-a085-4eafb714164f" />

3. Execute:

```bash
curl -vk https://<IP_USER1>
```

> O `-k` é intencional no laboratório: ele permite seguir com certificado autoassinado.
> **Resultado esperado no terminal:** a conexão HTTPS deve funcionar mesmo com o certificado autoassinado, e o `curl` deve retornar a resposta do Apache.

<img width="766" height="875" alt="image" src="https://github.com/user-attachments/assets/b612937e-31f3-40ec-9cfc-715101247637" />

#### C) Observar no Wireshark

1. **Pause a captura** no Wireshark.

<img width="138" height="92" alt="image" src="https://github.com/user-attachments/assets/cfedfeba-6545-4297-aa1b-530b14688ed8" />

2. Remova o filtro anterior (se houver).
3. Aplique o filtro:

   ```text
   tls && ip.addr == <IP_USER1>
   ```

4. **O que observar:** pacotes de **handshake TLS** (ClientHello/ServerHello/Certificado) e **payload cifrado**.

<img width="1086" height="420" alt="image" src="https://github.com/user-attachments/assets/033753e3-daae-4dfd-966a-ce779528bdf9" />
<img width="1188" height="701" alt="image" src="https://github.com/user-attachments/assets/304bdb65-de32-46b1-a97e-19e5b85d3833" />

> **Ao final deste cenário:** o conteúdo da aplicação não deve aparecer legível no Wireshark; apenas o handshake e os dados criptografados ficam visíveis.

---

## VI. Verificação e Resultados

### 1) Checklist de sucesso do experimento

* **Cenário HTTP (porta 80):** `curl -v http://<IP_USER1>` retorna conteúdo e o Wireshark mostra `GET`/`200 OK` com payload legível.
* **Cenário HTTPS (porta 443):** `curl -vk https://<IP_USER1>` funciona, e no Wireshark aparecem `ClientHello`/`ServerHello` com payload cifrado.

### 2) Quadro comparativo

| Protocolo | Porta | TLS | Visível no Wireshark                                                 | Payload                             |
| --------- | ----: | :-: | -------------------------------------------------------------------- | ----------------------------------- |
| HTTP      |    80 | Não | Métodos/URLs, cabeçalhos, corpo                                      | **Legível** (`HELLO_TLS_HTTP`)     |
| HTTPS     |   443 | Sim | Handshake (ClientHello/ServerHello/Cert), *Application Data*          | **Ilegível**                        |

### 3) Filtros práticos (copiar/colar)

* HTTP claro: `http`
* HTTPS: `tls` *(ou `tcp.port == 443`)*
* Handshake específico: `tls.handshake`
* Verificar string no tráfego claro: `frame contains "HELLO_TLS_HTTP"`

---

## VII. Conclusão

Foi demonstrado, de forma prática, que a adoção de **TLS** em HTTP protege a **confidencialidade** das mensagens. Enquanto no tráfego sem TLS foi possível ler o payload em claro, no tráfego com TLS o Wireshark exibiu apenas metadados do handshake e pacotes cifrados.

---

## Apêndice — Troubleshooting rápido

* **Não vejo tráfego no Wireshark:** selecione a interface da NAT Network correta e gere tráfego com `curl` no user 2 - Debian.
* **HTTPS não sobe:** confirme `a2enmod ssl`, `a2ensite default-ssl`, caminhos de certificado no `default-ssl.conf` e reinício do Apache.
* **`curl -vk https://<IP_USER1>` falha:** verifique se a porta 443 está em LISTEN (`ss -lntp | grep :443`).
* **`ping` entre as VMs falha:** confirme que ambas estão em **NAT Network (`NatNetwork`)**, que os dois IPs estão na mesma faixa de rede e que o teste usa o endereço correto de destino.
* **VMs não se enxergam:** confirme ambas em **NAT Network (`NatNetwork`)** com DHCP ON.
* **Wireshark não abre ou não captura corretamente:** execute novamente com `sudo wireshark` e confirme que a interface da NAT Network foi selecionada.
