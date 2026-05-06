# T02 – Redes de Dados I

# Relatório 10: TLS em HTTP com Wireshark (VMs no VirtualBox)

Este relatório tem como objetivo **montar um laboratório em NAT Network (rede NAT do VirtualBox)** para demonstrar o uso de **TLS** no protocolo **HTTP**, comparando o tráfego em texto claro (**HTTP sem TLS**) e criptografado (**HTTPS com TLS**) e **analisando as diferenças no Wireshark**.

* **Material de apoio:** `https://github.com/vin1sss/T02---Redes-de-Dados-1/`

---

## I. Introdução

Nesta atividade, você configurará duas VMs (**user 1 - Debian** e **user 2 - Debian**) em uma **NAT Network** do VirtualBox para observar, via Wireshark no **user 2 - Debian**, como o TLS protege o conteúdo das mensagens HTTP. Em texto claro, o payload é legível; com TLS, apenas metadados e o **handshake** ficam visíveis.

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

> **Atenção (didático):** neste relatório usamos **NAT Network** (rede compartilhada entre VMs). Isso é diferente de **NAT** simples por VM (*Attached to: NAT*), que normalmente não permite comunicação direta entre as VMs.

* **user 1 - Debian (Servidor):** Apache (80/443)
* **user 2 - Debian (Cliente):** `curl` e **Wireshark** (captura local)

> **Configuração de Adaptadores (todas as VMs)**
> **Adapter único:** **Attached to:** *NAT Network*
> **Name:** `NatNetwork` (**DHCP: On**)
> **Recursos da VM (todas as VMs):** **6 GB de RAM** e **3 núcleos de CPU**
> (uma única placa por VM; Internet + comunicação entre VMs; sem endereçamento manual)

### 1) IPs via DHCP (sem endereçamento manual)

Com **NAT Network (NatNetwork)**, os IPs são atribuídos automaticamente por **DHCP**. Use o **`ip a`** para identificar o IP atual do **user 1 - Debian** e do **user 2 - Debian**. Anote o IP do servidor como **<IP_USER1>** e utilize-o em todos os testes.

### 2) No user 2 - Debian — captura (Wireshark)

Abra o Wireshark com `sudo wireshark` e selecione a **interface da NAT Network** (a que mostra o IP do user 2 - Debian).
**Filtro geral:** `ip.addr == <IP_USER1>`

### 3) Pacotes necessários (resumo)

* **user 1 - Debian:** `apache2`, `openssl`
* **user 2 - Debian:** `curl`, `wireshark`

---

## IV. Instalação e Preparação

### 0) **Preparar a rede (VirtualBox → Ferramentas → Redes NAT)**

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
   * OK e **iniciar** as VMs
  
<img width="808" height="345" alt="image" src="https://github.com/user-attachments/assets/3a4a1d8a-63b4-4c48-b06a-d65de72964a2" />

5. **Ajustar recursos de hardware das VMs (user 1 - Debian e user 2 - Debian):**

   * VirtualBox → **Configurações** → **Sistema** → **Placa-mãe** → **Memória Base: `6144 MB` (6 GB)**
   * VirtualBox → **Configurações** → **Sistema** → **Processador** → **Processadores: `3`**

6. **Verificação rápida dentro das VMs:**

   ```bash
   ip a
   ping -c2 <IP_USER1>   # de user 2 - Debian para user 1 - Debian
   ```
<img width="1500" height="564" alt="image" src="https://github.com/user-attachments/assets/41fda5a8-7fe4-400c-98ac-8a995fc836cb" />

7. **Resetar IP para DHCP da `NatNetwork` (caso a VM esteja com IP estático):**

   Execute em **cada VM Debian**:

   ```bash
   IFACE=$(ip route | awk '/default/ {print $5; exit}')
   sudo dhclient -r "$IFACE" || true
   sudo ip addr flush dev "$IFACE"
   sudo dhclient "$IFACE"
   ip -4 a show "$IFACE"
   ```

### A) user 1 - Debian (Servidor)

**Atualizar e instalar:**

```bash
sudo apt update && sudo apt install -y apache2 openssl
```

**Criar página de teste (conteúdo identificável):**

```bash
echo "<h1>HELLO_TLS_HTTP</h1>" | sudo tee /var/www/html/index.html
```

**Verificar serviço e porta HTTP:**

```bash
sudo systemctl enable --now apache2
ss -tulpn | grep :80
```

### B) user 2 - Debian (Cliente)

**Instalar cliente HTTP e Wireshark:**

> Observação: escolha "Yes" na configuração do Wireshark

```bash
sudo apt update && sudo apt install -y curl wireshark
sudo dpkg-reconfigure wireshark-common
sudo usermod -aG wireshark $USER
newgrp wireshark
```

**No user 2 - Debian — captura (Wireshark):**

```bash
sudo wireshark
```

---

## V. Procedimentos (Passo a Passo)

### Cenário 1 — HTTP **sem TLS** (porta 80)

1. **No user 2 - Debian:** iniciar captura no Wireshark.
2. **No user 2 - Debian:** executar teste HTTP:

```bash
curl -v http://<IP_USER1>
```

3. **No Wireshark:** usar filtro `http`.
   **O que observar:** requisição `GET / HTTP/1.1` e resposta `200 OK` com **payload legível** (HTML contendo `HELLO_TLS_HTTP`).

> **Lembre-se de reiniciar a captura no Wireshark para o próximo cenário.**

### Cenário 2 — HTTP **com TLS** (HTTPS na porta 443)

**No user 1 - Debian:**

1. **Ativar SSL no Apache:**

```bash
sudo a2enmod ssl
sudo a2ensite default-ssl
sudo systemctl reload apache2
```

2. **Gerar certificado autoassinado (1 ano):**

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/apache.key \
  -out /etc/ssl/certs/apache.crt \
  -subj "/C=BR/ST=MG/L=Inatel/O=Lab/OU=NetSec/CN=<IP_USER1>"
```

3. **Apontar o vhost SSL para o certificado/chave e reiniciar Apache:**

```bash
sudo sed -i 's#SSLCertificateFile .*#SSLCertificateFile /etc/ssl/certs/apache.crt#' /etc/apache2/sites-available/default-ssl.conf
sudo sed -i 's#SSLCertificateKeyFile .*#SSLCertificateKeyFile /etc/ssl/private/apache.key#' /etc/apache2/sites-available/default-ssl.conf
sudo systemctl restart apache2
ss -tulpn | grep :443
```

**No user 2 - Debian:**

4. Reiniciar captura no Wireshark.
5. Executar teste HTTPS:

```bash
curl -vk https://<IP_USER1>
```

> O `-k` é intencional no laboratório: ele permite seguir com certificado autoassinado.

6. No Wireshark, usar filtro `tls` ou `tcp.port == 443`.
   **O que observar:** pacotes de **handshake TLS** (ClientHello/ServerHello/Certificado) e **payload cifrado**.

---

## VI. Verificação e Resultados

### 1) Quadro comparativo

| Protocolo | Porta | TLS | Visível no Wireshark                                                 | Payload                             |
| --------- | ----: | :-: | -------------------------------------------------------------------- | ----------------------------------- |
| HTTP      |    80 | Não | Métodos/URLs, cabeçalhos, corpo                                      | **Legível** (`HELLO_TLS_HTTP`)     |
| HTTPS     |   443 | Sim | Handshake (ClientHello/ServerHello/Cert), *Application Data*          | **Ilegível**                        |

### 2) Filtros práticos (copiar/colar)

* HTTP claro: `http`
* HTTPS: `tls` *(ou `tcp.port == 443`)*
* Handshake específico: `tls.handshake`
* Verificar string no tráfego claro: `frame contains "HELLO_TLS_HTTP"`

### 3) Critérios de sucesso do experimento (guia)

* **Cenário HTTP (porta 80):** `curl -v http://<IP_USER1>` retorna conteúdo e o Wireshark mostra `GET`/`200 OK` com payload legível.
* **Cenário HTTPS (porta 443):** `curl -vk https://<IP_USER1>` funciona, e no Wireshark aparecem `ClientHello`/`ServerHello` com payload cifrado.

---

## VII. Conclusão

Foi demonstrado, de forma prática, que a adoção de **TLS** em HTTP protege a **confidencialidade** das mensagens. Enquanto no tráfego sem TLS foi possível ler o payload em claro, no tráfego com TLS o Wireshark exibiu apenas metadados do handshake e pacotes cifrados.

---

## Apêndice — Troubleshooting rápido

* **Não vejo tráfego no Wireshark:** selecione a interface da NAT Network correta e gere tráfego com `curl` no user 2 - Debian.
* **HTTPS não sobe:** confirme `a2enmod ssl`, `a2ensite default-ssl`, caminhos de certificado no `default-ssl.conf` e reinício do Apache.
* **`curl -vk https://<IP_USER1>` falha:** verifique se a porta 443 está em LISTEN (`ss -lntp | grep :443`).
* **VMs não se enxergam:** confirme ambas em **NAT Network (`NatNetwork`)** com DHCP ON.
* **Sem captura como usuário comum:** rode `sudo dpkg-reconfigure wireshark-common`, adicione usuário ao grupo `wireshark` e reabra sessão (`newgrp wireshark`).
