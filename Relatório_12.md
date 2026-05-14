<img width="180" alt="Logo T02" src="https://github.com/user-attachments/assets/cf68b22b-ecb8-4e9e-8f09-271ef1bc1246" />

<div align="right">
T02 - Redes de Dados
</div>

# Relatório 12: Configuração de Firewall de Rede usando **pfSense** (VMs no VirtualBox)

Este relatório descreve a configuração de um **firewall pfSense** atuando como **gateway/NAT e filtro** para uma rede interna com **duas VMs**: **pfSense** (gateway/firewall) e **Debian** (cliente). O objetivo é **montar o ambiente**, aplicar **regras de firewall** típicas e **validar o comportamento** por meio de testes práticos e análise de **logs** no pfSense.

* **Material de apoio:** `https://github.com/vin1sss/T02---Redes-de-Dados-1/`

---

## I. Introdução

Você utilizará um **pfSense** com duas interfaces (WAN/LAN) no VirtualBox:

* **WAN**: acesso à Internet via **NAT** do VirtualBox.
* **LAN**: rede isolada (**Rede Interna**) onde ficará a VM **Debian**.

Aplicaremos **regras** (ex.: **bloquear HTTP**, **bloquear um site específico**, **bloquear ICMP para Internet**) e verificaremos **logs** no pfSense.
**Pilares:** **Confidencialidade** e **Disponibilidade**, com ênfase em **Controle de Acesso** e **Auditoria**.

---

## II. Conceitos e Fundamentos

* **Firewall stateful (pfSense):** regras avaliadas **de cima para baixo**; mantém **estado** das conexões.
* **NAT:** traduz IPs da LAN privada para o IP da WAN (saída para Internet).
* **Ordem de regras:** a **primeira regra compatível** decide (ALLOW/DENY).
* **Logs:** registram eventos **permitidos/bloqueados**, essenciais para auditoria.
* **Testes práticos:** **navegador** (HTTP/HTTPS) e `ping` (ICMP).

---

## III. Ambiente e Topologia

**VirtualBox (2 VMs):**

* **VM-pfSense**v

  * **Adapter 1 (WAN):** **NAT** (DHCP do VBox).
  * **Adapter 2 (LAN):** **Rede Interna** chamada **`LAN_PFS`** (pfSense atende esta LAN).
  * **Recursos da VM:** **6 GB de RAM** e **3 núcleos de CPU**
* **Debian**

  * **Adapter 1:** **Rede Interna** **`LAN_PFS`** (IP via **DHCP** do pfSense).
  * **Recursos da VM:** **6 GB de RAM** e **3 núcleos de CPU**

> **Assunção para o lab:** usar a **configuração padrão do pfSense** para LAN (ex.: `192.168.1.1/24` com DHCP habilitado). Não trataremos o “primeiro setup” do pfSense neste relatório.

> **Observação:** Diferente do Relatório 10 (NAT Network para ambas as VMs), aqui o pfSense **precisa de duas interfaces** (WAN e LAN). Por isso usamos **NAT** (WAN) + **Rede Interna** (LAN).

### 1) Resumo do laboratório

| Elemento | Definição |
| --- | --- |
| Rede do VirtualBox (WAN do pfSense) | **NAT** com DHCP do VirtualBox |
| Rede do VirtualBox (LAN do pfSense) | **Rede Interna** `LAN_PFS` (DHCP fornecido pelo pfSense) |
| VM-pfSense | Gateway/Firewall com duas interfaces (WAN e LAN) |
| VM-Debian | Cliente da LAN; gera o tráfego dos testes |
| Adaptadores da VM-pfSense | Adapter 1 → NAT (WAN); Adapter 2 → Rede Interna `LAN_PFS` (LAN) |
| Adaptador da VM-Debian | Adapter 1 → Rede Interna `LAN_PFS` |
| Recursos das VMs | **6 GB de RAM** e **3 núcleos de CPU** |

### 2) Dados de referência usados no relatório

| Item | Uso |
| --- | --- |
| Senha do Debian | **`pytest`** |
| Senha solicitada pelo `sudo` no Debian | **`pytest`** |
| Credenciais da WebGUI do pfSense | Usuário **`admin`** / senha **`pfsense`** |
| IP LAN do pfSense (gateway) | **`192.168.1.1/24`** |
| Faixa DHCP da LAN | **`192.168.1.10` – `192.168.1.254`** |
| **`<IP_DEBIAN>`** | IP recebido pela VM Debian via DHCP do pfSense |
| **`<IP_WAN_PFS>`** | IP recebido pela WAN do pfSense via DHCP do VirtualBox (ex.: `10.0.2.x`) |

### 3) Pacotes usados no laboratório

* **VM-pfSense:** nenhum pacote adicional — configuração feita pelo CLI e pela WebGUI.
* **VM-Debian:** `net-tools`, `dnsutils`, `curl`

---

## IV. Instalação e Preparação

### 0) **Configurando pfSense**

1. **pfSense — habilitar e configurar as duas interfaces no VirtualBox**

* VirtualBox → **botão direito** na VM **pfSense** → **Configurações** → **Rede**.
* **Adaptador 1 (WAN):**

  * **Habilitar Placa de Rede**
  * **Conectado a:** **NAT**
  * **Anotar o endereço MAC do adaptador** — os últimos 4 caracteres são suficientes para posterior configuração no pfSense.
  
  <img width="800" alt="Adaptador 1 NAT" src="https://github.com/user-attachments/assets/ed4886ae-aed5-4681-b021-6a73fbf13397" />

* **Adaptador 2 (LAN):**

  * **Habilitar Placa de Rede**
  * **Conectado a:** **Rede Interna**
  * **Nome:** **`LAN_PFS`**  ← **digite exatamente este nome** (se não existir, o VirtualBox cria ao salvar).
  * **Anotar o endereço MAC do adaptador** — os últimos 4 caracteres são suficientes para posterior configuração no pfSense.
  
  <img width="800" alt="Adaptador 2 Rede Interna" src="https://github.com/user-attachments/assets/ca317cf7-a5c4-4fff-b7b3-fd1f3e8ca574" />

* **OK**.

2. **Debian — conectar à mesma LAN**

> **Observação**: De preferencia, selecione _`VM1 - servidor`, `VM2 - cliente`, `user 1 - Debian` ou `user 2 - Debian`_

* **Botão direito** na VM **Debian** → **Configurações** → **Rede**.
* **Adaptador 1:** **Rede Interna** → **Nome:** **`LAN_PFS`**.

<img width="800" alt="Debian Rede Interna" src="https://github.com/user-attachments/assets/f28dd00a-cc73-4b28-9f10-47232d2b9d10" />

* **OK**.

3. **Ligar o pfSense**

* Inicie a VM **pfSense** no VirtualBox e aguarde o carregamento completo.

<img width="800" alt="Tela inicial do pfSense" src="https://github.com/user-attachments/assets/6ad39da2-6809-43a4-a026-46120cf96a00" />


4. **OBSERVAÇÕES**
* No menu superior **"Visualisar" > "Tela virtual 1" > "Escalonar para 200%"**. Isso facilita a visualização no terminal.
* Se, em qualquer momento acabar pressionando "Ctrl+C", saindo da inferface CLI do `pfSense`, é possível retornar executando o binário: `/etc/rc.initial`.

#### A) Resetar configurações

No terminal do pfSense:

1. Selecionar a opção **`4) Reset to factory`**: -> Digite 4. Pressione Enter.
2. Confirme digitando **`y`** e pressionando Enter.

<img width="800" alt="Reset to factory defaults" src="https://github.com/user-attachments/assets/96693b1d-45c0-43f6-b8c6-f6f4c0410b93" />


#### B) Associar interfaces de rede

Na tela de boas-vindas do pfSense, é possível verificar que existem duas interfaces: **em0** e **em1**.

<img width="800" alt="Tela welcome com interfaces" src="https://github.com/user-attachments/assets/1417dec6-ddea-4bb7-b24f-99c60ae03e8a" />


1. Digite **`1`** e pressione **Enter** para selecionar **"Assign Interfaces"**.
2. Quando perguntado **"Should VLAN be set up now?**, digite **`n`** e pressione **Enter**.
3. Serão exibidas as interfaces disponíveis (**em0** e **em1**) com seus respectivos **endereços MAC**.
4. Compare os endereços MAC exibidos com os anotados nas configurações do VirtualBox:

   * Para **WAN**, selecione a interface cujo MAC corresponde ao adaptador do tipo **NAT**.
   * Para **LAN**, selecione a interface cujo MAC corresponde ao adaptador **Rede Interna** (`LAN_PFS`).

5. Quando perguntado **"Do you want to proceed?"**, digite **`y`** e pressione **Enter**.

<img width="800" alt="Associação de interfaces no pfSense" src="https://github.com/user-attachments/assets/a8dd0bcc-f28d-4292-a8a2-969345070ee7" />

6. Aguarde a conclusão. Na tela de boas-vindas, a associação estará atualizada — no teste realizado: **em0 → WAN** e **em1 → LAN**.

#### C) Definir endereço IP da interface WAN

Como o adaptador WAN é do tipo **NAT**, o VirtualBox fornece endereço IP automaticamente via DHCP; portanto, a interface será configurada dessa forma.

1. Digite **`2`** e pressione **Enter** para selecionar **"Set interface(s) IP addresses"**.
2. Selecione o número correspondente à **WAN** (no teste realizado: **`1`**) e pressione **Enter**.
3. **"Configure IPv4 address WAN interface via DHCP?"** → Digite **`y`** e pressione **Enter**.
4. **"Configure IPv6 address WAN interface via DHCP6?"** → Digite **`n`** e pressione **Enter**.
5. Pressione **Enter** (para que o endereço IPv6 não seja configurado).
6. **"Do you want to revert to HTTP as the webConfigurator protocol?"** → Digite **`n`** e pressione **Enter**.

<img width="800" alt="Atribuindo DHCP para interface WAN" src="https://github.com/user-attachments/assets/47148e09-163d-4138-9e53-c1bfd7607010" />

7. Quando solicitado, pressione **Enter** para retornar ao menu principal.

Após a conclusão, na tela de boas-vindas a linha **WAN** deverá indicar **v4/DHCP** e um endereço IP condizente com a rede NAT do VirtualBox (ex.: `10.0.2.x`).

<img width="800" alt="pfSense WAN com DHCP" src="https://github.com/user-attachments/assets/3f460d76-e6e5-426d-bd6d-b52e1bce5bcc" />


> **Ao final desta etapa:** a interface WAN do pfSense deve ter recebido endereço IPv4 via DHCP do VirtualBox.

#### D) Definir endereço estático da interface LAN e configurar DHCP da rede

Como a **Rede Interna** não possui nenhum serviço DHCP por parte do VirtualBox, o pfSense será responsável por fornecer endereços IP às máquinas conectadas a essa rede.

1. Digite **`2`** e pressione **Enter** para selecionar **"Set interface(s) IP addresses"**.
2. Selecione o número correspondente à **LAN** (no teste realizado: **`2`**) e pressione **Enter**.
3. **"Configure IPv4 address LAN interface via DHCP?"** → Digite **`n`** e pressione **Enter**.
4. Digite o endereço IP **`192.168.1.1`** e pressione **Enter**.
5. Digite a máscara de sub-rede **`24`** e pressione **Enter**.
6. Pressione **Enter** (sem digitar nada) para o campo de upstream gateway.

<img width="800" alt="Atribuindo IP estático para interface LAN" src="https://github.com/user-attachments/assets/ee63ff00-d73b-44ee-a4f8-70c42eccbb8c" />

7. **"Configure IPv6 address LAN interface via DHCP6?"** → Digite **`n`** e pressione **Enter**.
8. Pressione **Enter** (sem digitar nada) para o campo de endereço IPv6.
9. **"Do you want to enable the DHCP server on LAN?"** → Digite **`y`** e pressione **Enter**.
10. **"Enter the start address of the IPv4 client address range:"** → Digite **`192.168.1.10`** e pressione **Enter**.
11. **"Enter the end address of the IPv4 client address range:"** → Digite **`192.168.1.254`** e pressione **Enter**.

<img width="800" alt="Ativando servidor DHCP para interface LAN" src="https://github.com/user-attachments/assets/a87843d7-7493-49e1-9fd4-c617b98efccc" />

12. **"Do you want to revert to HTTP as the webConfigurator protocol?"** → Digite **`n`** e pressione **Enter**.

Quando solicitado que se pressione **Enter**, surgirá a mensagem **"You can access the webConfigurator by opening the following URL in your browser"**, confirmando que o IP foi corretamente configurado. O endereço estático da interface LAN (**`192.168.1.1`**) também será exibido — máquinas conectadas à Rede Interna já podem acessar o pfSense por esse endereço.

<img width="800" alt="Gateway pfSense pronto" src="https://github.com/user-attachments/assets/cb4b76ea-cfe8-472c-b3cb-bc16815cd524" />


> **Ao final desta etapa:** o pfSense deve estar com WAN ativa via DHCP, LAN estática em `192.168.1.1/24` e servidor DHCP ativo na LAN distribuindo IPs em `192.168.1.10-254`.

#### E) Debian — Conectar à Rede Interna

Inicie a VM **Debian**.

> **Importante:** nos relatórios do Bloco 1, as configurações de rede foram definidas pela interface gráfica do Ubuntu, que interage com o **NetworkManager**. Agora, utilizaremos uma segunda forma de configurar a interface de rede: editando diretamente o arquivo `/etc/network/interfaces`.

1. **Descubra o nome da interface de rede:**
   ```bash
   ip link show
   ```
   Anote o nome da interface (ex: `eth0`, `enp0s3`). Substitua `<interface>` nos comandos seguintes.

2. **Edite o arquivo de interfaces:**
   ```bash
   sudo nano /etc/network/interfaces
   ```
   Remova qualquer configuração existente da interface física e substitua pela configuração DHCP. O arquivo deve ficar assim:
   ```
   auto lo
   iface lo inet loopback

   auto <interface>
   iface <interface> inet dhcp
   ```

3. **Recarregue a interface para aplicar:**
   ```bash
   sudo ifdown <interface> ; sudo ifup <interface>
   ```

4. **Verifique a conectividade:**
   ```bash
   ip a        # deve exibir um IP na sub-rede da LAN
   ip route    # o gateway padrão deve apontar para o pfSense
   ping -c2 8.8.8.8
   ```

   <img width="800" alt="Terminal Debian conectado à LAN" src="https://github.com/user-attachments/assets/eaeb4310-122b-43e0-a14b-575813439813" />


6. **Instale utilitários de rede:**
   ```bash
   sudo apt update && sudo apt install -y net-tools dnsutils curl
   ```

   <img width="800" alt="Pacotes instalados no Debian" src="https://github.com/user-attachments/assets/a5555d9c-c045-4f96-8c4a-69c436dd524a" />


8. **Acesse a WebGUI do pfSense:** no navegador do Debian, abra `https://192.168.1.1` (o IP LAN do pfSense).

9. Ignore o aviso de certificado autoassinado: clique em **Advanced...** e depois em **Accept the Risk and Continue**.
   
   <img width="800" alt="Aviso certificado pfSense" src="https://github.com/user-attachments/assets/93f8d318-91b0-4530-a728-449236d65ab5" />

10. Credenciais de acesso: usuário `admin`, senha `pfsense`.

   <img width="800" alt="Login WebGUI pfSense" src="https://github.com/user-attachments/assets/527dc89e-c399-4cb8-a3aa-451ab270d2c4" />

   <img width="800" alt="Dashboard inicial pfSense" src="https://github.com/user-attachments/assets/e6ab7283-3715-4099-973a-c56755a7a121" />


> **Ao final desta etapa:** o Debian deve estar com IP na sub-rede `192.168.1.0/24`, gateway apontando para `192.168.1.1`, acesso à Internet funcionando e a WebGUI do pfSense aberta e logada.

---


## V. Procedimentos (Passo a Passo)

> [!IMPORTANT] Aviso
> No pfSense, vá em **Firewall > Rules > LAN**. As regras são lidas **de cima para baixo**. Coloque as **regras de bloqueio acima** da regra “allow LAN to any”.

### Checklist antes dos testes

Antes de iniciar os cenários, confirme:

* a VM **pfSense** está ligada com **WAN via DHCP** e **LAN em `192.168.1.1/24`**;
* a VM **Debian** está na **Rede Interna `LAN_PFS`**, recebeu IP na faixa `192.168.1.10-254` e tem gateway `192.168.1.1`;
* a WebGUI do pfSense (`https://192.168.1.1`) está acessível e logada como `admin`;
* o Debian tem `curl`, `dig` (`dnsutils`) e o navegador instalados;
* a regra padrão **allow LAN to any** existe em `Firewall > Rules > LAN` (será mantida abaixo das regras de bloqueio).

### Cenário 1 — Bloquear **HTTP (porta 80)** e permitir **HTTPS**

**Objetivo:** impedir navegação HTTP (em texto claro), mantendo a versão com criptografia (HTTPS) funcional.

1. **pfSense (WebGUI):** Na interface web, abra a página Firewall > Rules > LAN, e clique em `Add (seta para cima)`
   
   <img width="800" alt="Interface web Rules LAN" src="https://github.com/user-attachments/assets/f697d5ae-16d7-4d85-977e-95004caaaff4" />
2. Defina os parâmetro de necessários, listados abaixo:
   * **Action:** *Block*
   * **Interface:** *LAN*
   * **Address Family:** *IPv4*
   * **Protocol:** *TCP*
   * **Source:** *LAN subnets*
   * **Destination:** *any*
   * **Destination Port Range:** `HTTP (80)`
   * **Log:** (marcar opção)
   * **Description:** `BLOCK_LAN_HTTP_OUT`
   * **Save** → **Apply Changes**

   <img width="800" alt="Regra bloquear HTTP" src="https://github.com/user-attachments/assets/5945aa7d-aac7-4588-beed-3e62188e9e64" />

   <img width="800" alt="Regras LAN com bloqueio HTTP" src="https://github.com/user-attachments/assets/9e165a04-99c3-4047-af5b-ec4ed8b984db" />


#### Validação 1 (terminal Debian):
1. Execute o comando abaixo, na tentativa de acessar o domínio `example.com`. Ele deverá falhar, pois a requisição está sendo feita com o protocolo bloqueado (HTTP).
   ```bash
   curl -v -m4 http://example.com
   ```

   <img width="800" alt="Falha curl HTTP" src="https://github.com/user-attachments/assets/e62fcfa9-7746-464c-a0ec-03b11057337e" />

2. Em seguida, altere o comando, substituindo `http` por `https`, e execute-o. A requisição deve retornar o código de resposta `200 OK`.
   
   <img width="800" alt="Sucesso curl HTTPS" src="https://github.com/user-attachments/assets/409fb7f4-176d-4389-918e-a2290bbd6c9d" />

  
#### **Validação 2 (Página Logs):**
Aqui iremos validar o bloqueio realizado por meio da interface web do pfSense:

1. Abra a página **Status > System Logs > Firewall**
2. Selecione os filtros para interface (`LAN`), porta de destino (`80`).
3. Verifique entradas **blocked** oriundas do IP do Debian.

   <img width="800" alt="Logs firewall bloqueio HTTP" src="https://github.com/user-attachments/assets/f3294c68-a7ce-421f-a93c-ddf7095b6351" />

> **Ao final deste cenário:** o tráfego HTTP do Debian deve ser bloqueado pela regra `BLOCK_LAN_HTTP_OUT`, HTTPS deve continuar funcionando, e os bloqueios devem aparecer registrados no `Firewall Log` com action **block** e dst port `80`.

---

### Cenário 2 — Bloquear **um site específico** (por FQDN/alias)

**Objetivo:** impedir acesso a um domínio específico (ex.: `www.wikipedia.org`) mantendo o restante da navegação liberado.

> **Como funciona:** no pfSense, um **Alias** do tipo **Host(s)** aceita **FQDN (Fully Qualified Domain Name)**. O pfSense resolve esse nome para IP(s) e a **regra de bloqueio** usa esse alias como **destino**.
> *Obs.: em sites atrás de CDNs os IPs podem mudar; para fins de laboratório, o método é suficiente.*
> *Obs. 2: a atualização do alias por FQDN pode levar alguns minutos; em sala, aguarde um curto intervalo antes de repetir o teste.*

1. **Criar o Alias (pfSense WebGUI)**

No menu superior, abra: **Firewall > Aliases > Add**

Em "Properties":
* **Name:** `BLOCK_WIKI`
* **Type:** *Host(s)*

Em "Host(s)":
* **IP or FQDN:** `www.wikipedia.org`
* **Description:** `FQDN do site a bloquear`
* **Save** → **Apply Changes**

<img width="800" alt="Criar alias para site" src="https://github.com/user-attachments/assets/d34024a5-29b1-4035-ae02-23ecd81c4c11" />

<img width="800" alt="Alias criado para Wikipedia" src="https://github.com/user-attachments/assets/c0fb3552-c159-4216-948f-6cbd49deb482" />

2. **Criar a regra de bloqueio (pfSense)**

Abra, no menu: **Firewall > Rules > LAN → Add (seta para cima)**. Preencha com os dados a seguir:

* **Action:** *Block*
* **Interface:** *LAN*
* **Address Family:** *IPv4*
* **Protocol:** *TCP*
* **Source:** *LAN subnets*
* **Destination:** (Address or Alias), e o alias criado: **`BLOCK_WIKI`**
* **Destination Port Range:** *any*
* **Description:** `BLOCK_SITE_WIKIPEDIA`
* **Log:** (marcar opção)
* **Save** → **Apply Changes**

<img width="800" alt="Regra bloquear site Wikipedia" src="https://github.com/user-attachments/assets/598c435b-fb9d-4795-9341-be8ec860ba8d" />


> [!IMPORTANT]  
> Mantenha esta regra **acima** da regra “allow LAN to any”.

#### **Validação 1 (Debian, navegador):**

1. Abra **[https://www.wikipedia.org](https://www.wikipedia.org)** → **deve FALHAR** (bloqueado).
2. Abra **[https://example.com](https://example.com)** → **deve abrir normalmente** (não bloqueado).

   <img width="800" alt="Navegador bloqueio Wikipedia" src="https://github.com/user-attachments/assets/ada64b7d-844a-4fd1-ac27-cc946857d300" />


#### **Validação 2 (Página Logs):**
1. Abra a página **Status > System Logs > Firewall**
2. Selecione os filtros para interface (`LAN`).
3. Verifique entradas **blocked** cujo **Destination** é um dos IPs resolvidos para `www.wikipedia.org`.

   <img width="800" alt="Logs bloqueio Wikipedia" src="https://github.com/user-attachments/assets/877268a9-d1a6-4ca5-b764-81563bcbff0f" />


> **Dica:** Para visualizar a regra é possível limpar os logs anteriores. Na barra `Status /System Logs / Firewall / Normal View`, selecione Configurar (`Manage Log`, icone de ferramenta) e **Clear log**. Repita o teste.

> **Ao final deste cenário:** o domínio `www.wikipedia.org` não deve abrir no navegador do Debian, enquanto outros sites HTTPS continuam acessíveis, e o `Firewall Log` deve mostrar bloqueios cujo destino é um dos IPs resolvidos para esse FQDN.

---

### Cenário 3 — Bloquear **ICMP para Internet**, permitir ICMP para o gateway

**Objetivo:** impedir `ping` para Internet (ex.: 8.8.8.8), mantendo o diagnóstico interno para o gateway.

1. Abra a página **pfSense → Firewall > Rules > LAN → Add (seta para cima):**
2. Configure a nova regra de acordo:
   * **Action:** *Block*
   * **Protocol:** *ICMP*
   * **ICMP subtypes:** *any*
   * **Source:** *LAN subnets*
   * **Destination:** Marque a opção `Inverted`. Selecione *LAN subnets*
   * **Log:** (marcar opção)
   * **Description:** `BLOCK_LAN_ICMP_INTERNET`
   * **Save** -> **Apply Changes**

   <img width="800" alt="Regra bloquear ICMP Internet" src="https://github.com/user-attachments/assets/27e8e81f-d953-474f-a2cb-be955549a738" />

   <img width="800" alt="Regras LAN finais" src="https://github.com/user-attachments/assets/ba0b0f57-3ccf-4090-ac44-9156d896bac6" />


#### **Validação 1 (Debian, terminal):**

1. Acesse o terminal do Debian e digite ambos comandos abaixo. O primeiro deverá falhar, por tentar acessar um IP externo. Já o segundo, deverá ter sucesso, acessando o IP local (do próprio Gateway).
```bash
ping -c2 8.8.8.8
ping -c2 192.168.1.1
```

<img width="800" alt="Validação ICMP ping" src="https://github.com/user-attachments/assets/f46059ea-90a7-4867-b7da-bdffbb36cb54" />

#### **Validação 2 (Página Logs):**
1. Abra a página **Status > System Logs > Firewall**
2. Selecione os filtros para interface (`LAN`) e Destination Address (`8.8.8.8`).
4. Verifique entradas **blocked** cujo **Destination Addres** pertencem ao IP bloqueado.

   <img width="800" alt="Logs bloqueio ICMP" src="https://github.com/user-attachments/assets/2df7f385-9cf2-45a4-a132-6c8298b0d82c" />

> **Ao final deste cenário:** o `ping` para `8.8.8.8` deve falhar, o `ping` para `192.168.1.1` deve responder, e o `fast.log` do firewall deve registrar **blocks** ICMP saindo da LAN.

---

## VI. Verificação e Resultados

### 1) Checklist de sucesso do experimento

* **Cenário HTTP (porta 80):** `curl http://example.com` falha a partir do Debian, e `curl https://example.com` retorna `200 OK`.
* **Cenário site específico:** `https://www.wikipedia.org` não carrega no navegador do Debian, enquanto `https://example.com` carrega normalmente.
* **Cenário ICMP:** `ping 8.8.8.8` falha; `ping 192.168.1.1` (gateway) responde.
* Em **Status > System Logs > Firewall** aparecem entradas **blocked** correspondentes a cada cenário, com a interface `LAN` como origem.

### 2) Quadro comparativo (cenários)

| Cenário | Tráfego testado                          | Regra aplicada (Description)  | Ação    | Resultado esperado no Debian          | Onde verificar no pfSense                |
| :-----: | ---------------------------------------- | ----------------------------- | ------- | -------------------------------------- | ---------------------------------------- |
|    1    | LAN → Internet (HTTP/80)                 | `BLOCK_LAN_HTTP_OUT`          | Block   | `curl http://...` falha; HTTPS funciona | `Status > System Logs > Firewall` (LAN, dst port 80) |
|    2    | LAN → `www.wikipedia.org` (HTTPS)        | `BLOCK_SITE_WIKIPEDIA` + alias | Block   | Site não carrega; outros HTTPS abrem   | `Status > System Logs > Firewall` (LAN, dst = IP do alias) |
|    3    | LAN → Internet (ICMP, ex.: `8.8.8.8`)    | `BLOCK_LAN_ICMP_INTERNET`     | Block   | `ping 8.8.8.8` falha; `ping 192.168.1.1` responde | `Status > System Logs > Firewall` (LAN, dst `8.8.8.8`) |

### 3) Coleta rápida (comandos úteis no Debian)

```bash
# Validar bloqueio HTTP / liberação HTTPS
curl -v -m4 http://example.com
curl -v -m4 https://example.com

# Validar bloqueio de site (resolução DNS continua funcionando, conexão é dropada)
dig +short www.wikipedia.org
curl -v -m4 https://www.wikipedia.org

# Validar bloqueio ICMP com exceção do gateway
ping -c2 8.8.8.8
ping -c2 192.168.1.1
```

> Use este bloco ao final dos testes para registrar evidências textuais junto com os prints dos logs do pfSense.

---

## VII. Conclusão

Demonstrou-se, com **duas VMs** (pfSense e Debian), que o **pfSense** aplica **políticas de saída** eficazes (HTTP geral, **site específico**, ICMP), atuando como **gateway/NAT** e **firewall stateful**. Testes no **navegador** e com **ping** confirmaram os resultados, e os **logs** evidenciaram os eventos, reforçando a importância da **ordem das regras** e do **monitoramento**.

---

## Apêndice 1 — Troubleshooting rápido

* **Debian sem IP na LAN:** confirme **Rede Interna `LAN_PFS`** no adaptador e que o **DHCP da LAN** do pfSense está ativo.
* **Sem acesso à WebGUI:** use `https://192.168.1.1` (ou IP LAN do pfSense) e aceite o certificado; verifique se a regra **allow LAN to any** não foi removida.
* **Regra não surte efeito:** lembre-se da avaliação **top-down**; mantenha **blocks acima** do allow geral; habilite **Log** na regra.
* **Site específico ainda abre:** confirme a **ordem** da regra, o **alias** (`BLOCK_WIKI` → `www.wikipedia.org`), limpe o **cache DNS** do cliente e aguarde a **atualização do alias** pelo pfSense.
* **`http://neverssl.com` abre mesmo bloqueado:** verifique se a regra de **porta 80** está no **topo** e se não há regra conflitante; confira os **logs** do pfSense.
* **`ping 8.8.8.8` ainda sai:** confirme a regra **Block ICMP** e que a exceção **ALLOW_ICMP_TO_GATEWAY** está **acima** dela.

## Apêndice 2 — Tela preta ao carregar a interface gráfica no Debian (VirtualBox)

O Debian moderno utiliza **Wayland** como servidor gráfico padrão, que pode ser incompatível com as controladoras de vídeo virtuais do VirtualBox — a tela fica escura logo após o carregamento dos serviços, impedindo o login gráfico.

**Solução: forçar o uso do Xorg.** Quando a tela ficar escura, pressione **Ctrl + Alt + F2** para abrir um terminal em modo texto, faça login e execute:
```bash
sudo nano /etc/gdm3/daemon.conf
```
Localize a linha `#WaylandEnable=false`, remova o `#` e salve. Depois reinicie:
```bash
sudo reboot
```

Se o problema persistir, desligue a VM, acesse **Configurações > Tela** no VirtualBox e verifique:
- Controladora gráfica: **VMSVGA**
- **Aceleração 3D desativada**
- Memória de vídeo no máximo disponível
