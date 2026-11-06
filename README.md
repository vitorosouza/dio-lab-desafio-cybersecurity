# 🛡️ Simulação de Ataques de Força Bruta e Prevenção com Kali Linux e Medusa

Projeto prático desenvolvido como parte do desafio da plataforma **[DIO (Digital Innovation One)](https://dio.me)**, com o objetivo de auditar vulnerabilidades de autenticação, simular ataques controlados de força bruta e *password spraying*, e documentar medidas eficazes de mitigação e defesa em profundidade.

---

## 🎯 Objetivos de Aprendizagem
- Configuração de ambiente seguro e isolado para testes de segurança ofensiva.
- Execução de ataques de força bruta contra serviços de rede (FTP) utilizando o **Medusa**.
- Automação de tentativas de autenticação em aplicações web (**DVWA**).
- Enumeração de usuários de rede e execução de técnicas de ***Password Spraying*** em **SMB**.
- Análise defensiva e proposição de medidas de mitigação para cada serviço avaliado.

---

## 🖥️ Topologia do Ambiente de Laboratório

O laboratório foi configurado no **Oracle VirtualBox** utilizando uma rede interna exclusiva (**Host-Only / Placa Exclusiva de Hospedeiro**), garantindo isolamento total da internet e da rede corporativa/residencial:

| Máquina Virtual | Sistema Operacional | Função no Lab | Endereço IP (Host-Only) |
|---|---|---|---|
| **Atacante** | Kali Linux | Execução das ferramentas de auditoria | `192.168.56.x` |
| **Alvo** | Metasploitable 2 | Hospedagem dos serviços vulneráveis (FTP, SMB, DVWA) | `192.168.56.101` |

> *Evidências da configuração das placas de rede e do teste de conectividade (ping bem-sucedido entre as VMs):*
> 
> ![Configuração de Rede](images/01-rede-hostonly.png)

---

## 📂 Metodologia e Preparação de Wordlists

Para fins didáticos e otimização de tempo em ambiente de testes, foram criadas *wordlists* pequenas e controladas, demonstrando a prova de conceito sem sobrecarregar a memória das máquinas virtuais:

```bash
# Criação das listas no Kali Linux
echo -e "admin\nuser\nmsfadmin" > usuarios.txt
echo -e "123456\npassword\nadmin\nmsfadmin" > senhas.txt
```

> ![Wordlists Utilizadas](images/02-wordlists.png)

---

## 🧪 Execução dos Cenários de Teste

### Cenário 1: Força Bruta em Serviço FTP (Medusa)
O FTP (*File Transfer Protocol*) transmite credenciais em texto claro e é suscetível a ataques automatizados rápidos.

* **Comando executado:**
  ```bash
  medusa -h 192.168.56.101 -U usuarios.txt -P senhas.txt -M ftp -f
  ```
* **Parâmetros:**
  * `-h`: Endereço IP do alvo.
  * `-U`: Arquivo com a lista de usuários.
  * `-P`: Arquivo com a lista de senhas.
  * `-M ftp`: Módulo de FTP do Medusa.
  * `-f`: Interrompe a execução assim que a primeira credencial válida é encontrada (*Stop on first found*).
* **Resultado:** Credencial válida encontrada com sucesso (`msfadmin:msfadmin`). O acesso foi validado conectando-se diretamente via cliente `ftp`.

> ![Sucesso Medusa FTP](images/03-medusa-ftp.png)

---

### Cenário 2: Automação em Formulário Web (DVWA)
O DVWA (*Damn Vulnerable Web Application*) foi utilizado para simular a tentativa de adivinhar credenciais de login em uma aplicação web.

* **Desafio técnico e solução adotada:** O módulo `web-form` do Medusa v2.2 apresentou instabilidade (*segmentation fault*) no ambiente Kali moderno. Seguindo a flexibilidade proposta no desafio, utilizou-se o **Hydra**, ferramenta padrão de mercado para testes de autenticação HTTP POST:
* **Comando executado:**
  ```bash
  hydra -l admin -P senhas.txt 192.168.56.101 http-post-form "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:Login failed" -f -V
  ```
* **Resultado:** A ferramenta automatizou as requisições POST para `/dvwa/login.php` e identificou a combinação correta (`admin:password`), permitindo o acesso autenticado ao sistema.

> ![Automação Web DVWA](images/04-hydra-dvwa.png)

---

### Cenário 3: Enumeração e *Password Spraying* em SMB
Diferente da força bruta comum (que testa muitas senhas para uma mesma conta e ativa bloqueios), o ***Password Spraying*** testa **uma única senha comum contra múltiplos usuários**, contornando políticas tradicionais de *lockout*.

1. **Enumeração Prévia de Usuários:**
   ```bash
   enum4linux -U 192.168.56.101
   ```
   *Foram enumerados os usuários válidos existentes no serviço Samba/SMB.*

2. **Execução do Password Spraying:**
   ```bash
   hydra -U usuarios.txt -p msfadmin smb://192.168.56.101 -t 1 -f
   ```
   *(Ou via CrackMapExec: `crackmapexec smb 192.168.56.101 -u usuarios.txt -p msfadmin`)*
* **Resultado:** Identificação de que o usuário `msfadmin` compartilhava a senha idêntica ao seu identificador.

> ![Enumeração e Password Spraying](images/05-smb-spray.png)

---

## 🛡️ Medidas de Mitigação e Defesa

Para proteger os ambientes contra os ataques simulados, recomendam-se as seguintes práticas:

### 1. Proteção de Serviços de Rede (FTP e SSH)
- **Desativação de protocolos legados:** Substituição do FTP sem criptografia por **SFTP** ou **FTPS**.
- **Fail2ban / Bloqueio por IP:** Implementação de ferramentas de monitoramento de logs como o `fail2ban`, que bloqueiam temporariamente endereços IP que erram a autenticação repetidas vezes no firewall (`iptables`).
- **Autenticação por Chaves:** No caso de SSH, desativar autenticação por senha e exigir chaves criptográficas assimétricas.

### 2. Proteção de Aplicações Web (Formulários de Login)
- **Rate Limiting:** Limitação da quantidade de requisições por IP ou por sessão em um determinado intervalo de tempo.
- **Mecanismos Anti-Automação:** Implementação de CAPTCHAs (como Google reCAPTCHA ou Cloudflare Turnstile) após 3 tentativas incorretas.
- **Autenticação Multifator (MFA/2FA):** Torna os ataques de força bruta ineficazes mesmo que a senha seja adivinhada.

### 3. Proteção em Ambientes SMB / Active Directory
- **Políticas de Senhas Fortes:** Exigência de senhas longas (mínimo de 14 a 16 caracteres) e proibição de senhas previsíveis ou iguais ao nome de usuário.
- **Detecção de Password Spraying:** Monitoramento de logs de auditoria (ex.: Event ID 4625 no Windows) correlacionando falhas com a mesma senha em contas diferentes dentro de um intervalo de tempo.
- **Segmentação de Rede:** Bloqueio das portas 139 e 445 na borda da rede e controle de acesso via listas de controle (ACLs).

---

## 📚 Conclusão

A realização deste laboratório permitiu compreender na prática a facilidade com que serviços expostos com configurações padrão e senhas fracas podem ser comprometidos. Mais do que executar ferramentas ofensivas, o exercício reforçou a importância da aplicação de **defesa em profundidade**, controles de acesso e monitoramento contínuo em ambientes corporativos.

---
*Projeto desenvolvido por [Seu Nome] durante a formação na [DIO](https://dio.me).*
